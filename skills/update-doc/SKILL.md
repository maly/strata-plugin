---
name: update-doc
description: Rozhodovací subagent pro aktualizaci existujícího dokumentu ve Stratě. Dostane existující dokument a novou informaci, vrátí návrh volání MCP operace (doc_update / doc_supersede / doc_mark_stale) jako čistý JSON. Může být volán interně z make-doc nebo přímo uživatelem přes /strata:update-doc.
---

# Skill `update-doc`

Rozhodovací vrstva mezi orchestrátorem (`make-doc`) a MCP serverem.
Dostaneš existující dokument a novou informaci — vrátíš **návrh volání MCP operace**
jako čistý JSON. **Nic nevykonáváš** — rozhoduješ a formuluješ.

---

## Vstup

Dostaneš kontext v jednom z těchto formátů:

**A) Vnitřní volání z make-doc:**

```
EXISTUJÍCÍ DOKUMENT:
id: <doc_id>
type: <typ>
title: <titulek>
slug: <slug>
l1: <L1>
l2: <L2>
body: <tělo nebo výňatek>

NOVÁ INFORMACE:
<volný text — úryvek konverzace, bash výstup, popis>

KONTEXT VOLÁNÍ:
source: make-doc
```

**B) Přímé volání uživatelem (`/strata:update-doc`):**

Uživatel napíše příkaz s `doc_id` nebo popisem dokumentu a novou informací.
Nejprve zavolej `doc_read` pro daný dokument (level `full`), abys měl kompletní kontext.
Pak pokračuj stejnou rozhodovací logikou.

Pole `slug` je **povinné pro rozhodnutí** — přečti ho z `doc_read` pokud ho nemáš ve vstupu.

---

## Rozhodovací logika

### Krok 1 — Pravidlo slug/titulek (aplikuj jako první filtr)

Porovnej titulek a slug stávajícího dokumentu s tím, jak by musely vypadat
po aplikaci nové informace.

> **Pokud bys při update musel změnit slug nebo titulek → supersede, ne update.**

Toto pravidlo má přednost před všemi ostatními úvahami. Aplikuj ho jako první.

### Krok 2 — Hloubka změny (pouze pokud slug/titulek zůstávají)

- Přidání detailu, doplnění sekce, oprava faktu bez změny hlavní teze → **`doc_update`**
- Změna hlavní teze, závěru, doporučení (i kdyby slug mohl technicky zůstat) → **`doc_supersede`**

### Krok 3 — Jistota

Pokud nemáš dostatek kontextu pro rozhodnutí (nová informace je neúplná,
vztah k dokumentu nejasný) → **`doc_mark_stale`**.

`mark_stale` není slabá volba — je to správná odpověď na nejistotu.

### Speciální případ — typ `proposal`

- **Přidání `realized_by`** k existujícímu proposal → vždy `doc_update`, nikdy supersede. Záměr splnil svou roli a zůstává jako historický záznam, nenahrazuje se.
- **Změna podstaty záměru** (jiný scope, jiné řešení) → supersede (starý proposal jako historický záznam, nový ho nahradí).
- **Nastavení `status: outdated`** (zamítnutí, odložení, stažení) → **ruční akt autora**. Nenavrhuj autonomně. Pokud záměr ztrácí relevanci, navrhni `doc_update` s poznámkou, ale nechej finální rozhodnutí na uživateli.

---

## Výběr `reason.type`

| Situace | `reason.type` |
|---|---|
| Nová informace pochází z jiného dokumentu v systému | `due_to_document` |
| Nová informace pochází z konverzace, e-mailu, externího rozhodnutí | `external_input` |
| Zpřesnění nebo doplnění bez vnějšího podnětu | `refinement` |
| Strukturální přeorganizování bez sémantické změny | `refactor` |

`reason.ref` vyplň pokud znáš ID odkazovaného dokumentu v systému. Jinak pole vynech.

---

## Výstupní formát

Vrať **jeden JSON objekt**. Žádný text před nebo za objektem. Žádné markdown fences.

### Varianta A — update

```json
{
  "operation": "doc_update",
  "target_id": "rozhodnuti-mongodb-primarni-databaze-7a3f",
  "reason": {
    "type": "refinement",
    "note": "Doplněna informace o connection pooling limitu."
  },
  "patch": {
    "l2_draft": "V1: ...\nV2: ...\nV3: Doplněno: connection pool limit 100 spojení.",
    "body": "# Rozhodnutí\n\n..."
  },
  "rationale": "Obsah se rozšiřuje o nový detail bez změny podstaty. Slug zůstává přesný."
}
```

### Varianta B — supersede

```json
{
  "operation": "doc_supersede",
  "old_id": "rozhodnuti-mysql-databaze-9c4e",
  "new_doc": {
    "title": "MongoDB jako primární databáze pro Content API",
    "l1_draft": "Rozhodnutí přejít z MySQL na MongoDB pro Content API.",
    "l2_draft": "V1: Provozovali jsme MySQL...\nV2: Přechod na MongoDB kvůli flexibilnímu schématu.",
    "body": "# Rozhodnutí\n\n...",
    "tools": ["mongo"],
    "projects": ["content-api"]
  },
  "reason": {
    "type": "external_input",
    "note": "Změna databázové technologie po zkušenosti s produkčním provozem."
  },
  "rationale": "Mění se databázová volba — nový slug by musel být jiný. Dle pravidla: změna slugu = supersede."
}
```

### Varianta C — mark_stale

```json
{
  "operation": "doc_mark_stale",
  "target_id": "spec-dvouvrstva-klasifikace-3b1f",
  "reason": {
    "type": "due_to_document",
    "ref": "spec-trojvrstva-klasifikace-a2c4",
    "note": "Nový spec trojvrstvé klasifikace může tento dokument překonávat — neověřeno."
  },
  "rationale": "Nemám dostatek kontextu pro rozhodnutí. mark_stale předá rozhodnutí batch workeru."
}
```

### Společná povinná pole

| Pole | Popis |
|---|---|
| `operation` | `doc_update` \| `doc_supersede` \| `doc_mark_stale` |
| `rationale` | Lidsky čitelné odůvodnění — zobrazí se uživateli při potvrzení |
| `reason` | Strukturovaný reason — `type` povinný, `ref` a `note` dle situace |

---

## Přímé volání — tok

Pokud jsi spuštěn přímo (`/strata:update-doc`), postupuj takto:

1. Zjisti, na jaký dokument uživatel cílí:
   - Pokud uvedl `doc_id` nebo slug — zavolej `doc_read` s `level: full`
   - Pokud popsal téma — zavolej `doc_search` a upřesni s uživatelem
2. Zobraz L1 a L2 nalezeného dokumentu uživateli, ať ví, s čím pracuješ
3. Zjisti novou informaci od uživatele (pokud ji neuvedl v příkazu)
4. Proveď rozhodovací kroky 1–3
5. Vrať JSON návrh — ale protože jsi v přímém volání (ne subagent), zobraz ho
   uživateli formátovaně a čekej na potvrzení:

```
Návrh operace: doc_supersede

Odůvodnění: [rationale]

Stávající dokument: [id]
Nový dokument: [title]
Reason: [type] — [note]

Provést? [y] ano  [u] změnit na update  [m] mark_stale  [c] cancel
```

Po potvrzení zavolej příslušný MCP nástroj a oznam výsledek.
