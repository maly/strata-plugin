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

- **Přidání `realized_by`** k existujícímu proposal → vždy `doc_update`, nikdy supersede. Záměr splnil svou roli a zůstává jako historický záznam, nenahrazuje se. `realized_by` musí obsahovat ID **specifikace** (typ `spec`), ne implementace — řetězec je proposal → spec → implementation.
- **Změna podstaty záměru** (jiný scope, jiné řešení) → supersede (starý proposal jako historický záznam, nový ho nahradí).
- **Nastavení `status: outdated`** (zamítnutí, odložení, stažení) → **ruční akt autora**. Nenavrhuj autonomně. Pokud záměr ztrácí relevanci, navrhni `doc_update` s poznámkou, ale nechej finální rozhodnutí na uživateli.

### Speciální případ — typ `overview`

- **Doplnění nebo oprava `covers`** u existujícího overview → obvykle `doc_update`, nikdy automaticky supersede. Overview je mapa přes dokumenty; změna pokrytí je běžná údržba.
- **Zachování `covers`** je povinné, pokud nová informace nemění pokryté dokumenty. V patchi je nemusíš uvádět; server je zachová.
- **Nový `covers` seznam** uváděj jen když se pokrytí opravdu mění. Musí obsahovat minimálně dvě existující ID dokumentů.
- **Změna `scope`** je `doc_update`, pokud jde o zpřesnění stejné oblasti. Pokud se mění na jinou oblast a titulek/slug by už neseděl, použij pravidlo slug/titulek a navrhni `doc_supersede`.
- `scope` a `covers` jsou pole volání `doc_update`, ne položky uvnitř `type_specific`. V JSON návrhu je můžeš dát do `patch`; orchestrátor/server je sloučí do vstupu `doc_update`.

### Speciální případ — typ `implementation`

- **Změna stavu** (`ongoing` → `complete`) → vždy `doc_update`. Přidej do patche `type_specific: { status: 'complete' }`.
- **Změna cílové spec** (jiná `implements` vazba) → `doc_supersede`. Vazba `implements` je konstitutivní — jiná spec znamená jiná implementace, jiný slug.
- **`implementation` nelze supersede** přes `doc_supersede` API (server to odmítne). Pokud je potřeba nová implementace téže spec, uživatel musí vytvořit nový dokument přes `doc_write`.
- **Nikdy nenavrhuj `doc_supersede` pro změnu statusu** — jde vždy o `doc_update`.

---

## Výběr `reason.type`

Povolené typy jsou vázané na operaci — vybírej jen z podvýčtu pro zvolenou operaci:

**`doc_update`:**

| Situace | `reason.type` |
|---|---|
| Nová informace pochází z jiného dokumentu v systému | `due_to_document` |
| Zpřesnění nebo doplnění bez vnějšího podnětu | `refinement` |
| Nová informace pochází z e-mailu nebo externího rozhodnutí | `external_input` |
| Nová informace pochází z konverzace v Claude Code | `extraction_from_chat` |
| Strukturální přeorganizování bez sémantické změny | `refactor` |

**`doc_supersede`:**

| Situace | `reason.type` |
|---|---|
| Nahrazení vyvolané jiným dokumentem v systému | `due_to_document` |
| Nahrazení na základě konverzace, e-mailu, externího rozhodnutí | `external_input` |
| Zpřesnění bez vnějšího podnětu | `refinement` |
| Strukturální přeorganizování bez sémantické změny | `refactor` |

**`doc_mark_stale`:** **jen** `due_to_document` (podnětem je jiný dokument v systému) a `external_input` (podnět zvenčí — konverzace, e-mail, externí rozhodnutí). `refinement` a `refactor` tu server záměrně nepovoluje — když víš, co opravit, použij update.

`reason.ref` vyplň pokud znáš ID odkazovaného dokumentu v systému. Jinak pole vynech. Pro `due_to_document` je `ref` **povinný**. `ref` musí být ID dokumentu existujícího v systému — jinak server vrátí `reason_ref_not_found`.

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

### Varianta A2 — update overview covers/scope

```json
{
  "operation": "doc_update",
  "target_id": "prehled-overview-typu-strata-8f21",
  "reason": {
    "type": "refinement",
    "note": "Doplněn nový dokument, který patří do přehledu typu overview."
  },
  "patch": {
    "scope": "Typ overview ve Strata dokumentaci a serveru",
    "covers": [
      "plan-overview-type-2026",
      "decision-overview-covers",
      "spec-overview-frontmatter"
    ],
    "body": "# Přehled typu overview\n\n..."
  },
  "rationale": "Jde o údržbu mapy existujících dokumentů. Titulek a slug zůstávají přesné, proto stačí doc_update."
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
| `reason` | Strukturovaný reason — `type` povinný a jeho hodnota musí být z podvýčtu pro zvolenou `operation` (viz sekce Výběr `reason.type`), `ref` a `note` dle situace |

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
