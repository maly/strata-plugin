---
name: make-doc
description: Zaznamená výsledky práce a klíčová rozhodnutí z aktuální konverzace do dokumentačního systému Strata. Spouští se příkazem `/make-doc`. Skill analyzuje konverzaci, navrhne kandidáty na dokumentaci, projde je iterativně po jednom a po potvrzení uživatelem zapíše do dokumentačního systému přes MCP server `strata`. NIKDY nezapisuje samostatně bez explicitního potvrzení.
---

# Skill `make-doc`

Skill pro Claude Code, který umožňuje uživateli zaznamenat výsledky aktuální práce do dokumentačního systému Strata.

## Princip — dialogické rozhodování

Skill **nikdy nezapisuje bez potvrzení**. Pracovní tok je iterativní: skill navrhne kandidáty, uživatel rozhodne o výběru, skill projde vybrané kandidáty po jednom (search → návrh vztahu → potvrzení → zápis), po dokončení nabídne další kolo.

## Předpoklady

Skill používá MCP server `strata`. Před prvním krokem ověř, že nástroje `doc_search`, `doc_write`, `doc_update`, `doc_read` jsou dostupné. Pokud jakýkoliv volání MCP nástroje vrátí chybu spojení nebo autentizace, přeruš tok a oznam uživateli: "MCP server `strata` neodpovídá. Zkontroluj připojení a konfiguraci v Claude Code, pak spusť `/make-doc` znovu."

## Pracovní tok

### Krok 1 — analýza kontextu

Projdi aktuální konverzaci od začátku (nebo od posledního `/make-doc`, pokud jsi ho v session už spouštěl) a hledej následující signály:

- **Rozhodnutí** — výroky typu "rozhodneme se pro X", "vybíráme Y místo Z", explicitní volby technologie/přístupu/architektury
- **Instalace a konfigurace** — výsledky bash příkazů (`apt install`, `npm install`, `docker run`, `git clone`), vytvořené nebo upravené config soubory, env proměnné, deployment kroky
- **Návody** — sekvence kroků, které lze zopakovat, kde výsledek má hodnotu pro budoucí použití
- **Krizová řešení** — debugování, kde řešení stojí za zaznamenání jako runbook (co se stalo, jak to opravit)

**Nehledej** signály pro typ `source` — surovou konverzaci skill nezapisuje, to dělá separátní vytěžovací proces.

**Hledej i v tool calls**, ne jen v textu konverzace. Pro `config` a deployment kroky je často nejdůležitější to, co bylo skutečně provedeno v shellu, ne to, co bylo řečeno.

### Krok 2 — návrh kandidátů

Z analýzy sestav seznam kandidátů. **Maximálně 5 nejvýznamnějších** v jednom kole. Pokud najdeš víc, ohlaš to a zbylé nabídni v dalším kole.

Pro každý kandidát uveď:
- pořadové číslo
- navržený typ v hranatých závorkách (`decision`, `howto`, `config`, `runbook`)
- krátký popis kandidáta
- důvod, proč to navrhuješ (z čeho v konverzaci to plyne)

Formát:

```
Analyzoval jsem konverzaci a zde jsou kandidáti na dokumentaci:

1. [decision] Volba MongoDB jako databáze pro Content API
   Důvod: V průběhu jsme zvažovali Postgres a MongoDB, vybrali jsme MongoDB.

2. [howto] Postup nasazení Mongo na hetzner-1 přes Docker
   Důvod: Provedl jsem instalaci, která stojí za zaznamenání pro budoucí použití.

3. [config] Konfigurace replica set rs0 na hetzner-1
   Důvod: Konkrétní stav nasazení po dokončení instalace.

(Pokud najdeš více než 5: "Našel jsem ještě 3 další kandidáty, nabídnu je v dalším kole.")

Které mám zapsat?
- Odpověz čísly oddělenými čárkou (např. "1, 3")
- Nebo "všechny", "nic"
- Nebo "přidej <popis>" pro kandidát, kterého jsem vynechal
- Nebo "uprav N" pro úpravu kandidáta N (změna typu, popisu)
```

Po odpovědi uživatele zpracuj případné úpravy a přidání. Pokud uživatel odpoví "nic", přejdi rovnou na krok 9 (nabídka dalšího kola).

### Krok 3 — iterace přes vybrané kandidáty

Pro každý vybraný kandidát postupně proveď kroky 4–7. **Mezi kandidáty se dotazuj uživatele**, neprocházej je dávkově.

Před zpracováním dalšího kandidáta krátce ohlas: "Pokračuji s kandidátem 2/3: [howto] Postup nasazení Mongo..."

### Krok 4 — vyhledání existujících dokumentů

Pro aktuální kandidát zavolej `doc_search` s `include_l2: true` a relevantními filtry:
- `query` — klíčová slova z popisu kandidáta
- `type` — navržený typ
- `tools`, `projects` — pokud je z konverzace zřejmé

Cílem je najít sémanticky podobné existující dokumenty. Pokud `doc_search` vrátí žádné nebo zjevně nesouvisející výsledky, jdi rovnou na zápis nového dokumentu (krok 7, varianta `new`).

### Krok 5 — rozhodnutí o vztahu

Pokud `doc_search` vrátil kandidáty na shodu, rozhodni o vztahu:

- **`new`** — žádná sémantická shoda, vytváříme nový dokument
- **`update`** — existující dokument popisuje totéž, jen se posunul stav; detailní rozhodnutí (update / supersede / mark_stale) delegujeme na subagenta `update-doc`

Při rozhodování čti L2 nalezených dokumentů a porovnávej s navrhovaným kandidátem. Buď konzervativní — pokud váháš mezi `new` a `update`, raději `new` a nech uživatele rozhodnout v kroku 6.

**Poznámka:** Ty jako orchestrátor rozhoduješ jen binárně (`new` vs. `update`). Konkrétní typ úpravy (doc_update / doc_supersede / doc_mark_stale) a formulaci `reason` zajistí subagent `update-doc` v kroku 6b.

### Krok 6 — potvrzení uživatelem a delegace

**Pokud rozhodnutí je `new`** a žádný kandidát na shodu nebyl, přeskoč potvrzení a zapisuj rovnou (krok 7).

**Pokud rozhodnutí je `new` ale s nalezením podobného dokumentu**, informuj uživatele a nabídni:

```
Pro kandidát "Volba MongoDB jako databáze pro Content API" jsem našel podobný dokument:

  decision-volba-databaze-7a3f (active):
  L1: "Rozhodnutí použít MySQL jako primární databázi pro Content API."

Jak chceš postupovat?
  [n] new — vytvořit nový dokument bez vazby (jiný kontext)
  [u] update — existující dokument upravit (deleguji na update-doc subagenta)
  [r] references — nový dokument odkazuje na existující jako kontext
  [c] cancel — přeskočit
```

Čekej na odpověď uživatele.

**Pokud rozhodnutí je `update`** (sémantická shoda nalezena):

1. Přečti existující dokument přes `doc_read` s `level: full` — budeš ho předávat subagentovi.
2. Spusť subagenta `update-doc` s tímto kontextem:

```
EXISTUJÍCÍ DOKUMENT:
id: <doc_id>
type: <typ>
title: <titulek>
slug: <slug>
l1: <L1>
l2: <L2>
body: <tělo>

NOVÁ INFORMACE:
<obsah kandidáta — co víme z konverzace>

KONTEXT VOLÁNÍ:
source: make-doc
```

3. Subagent vrátí **čistý JSON** (žádný text kolem). Parsuj ho.

4. Validuj minimální strukturu:

| Operace | Povinná pole |
|---|---|
| `doc_update` | `target_id`, `patch` s alespoň jedním polem, `reason.type` |
| `doc_supersede` | `old_id`, `new_doc.title`, `new_doc.body`, `reason.type` |
| `doc_mark_stale` | `target_id`, `reason.type` |

Pokud validace selže → **nezavolej server**. Zobraz uživateli:

```
Subagent vrátil nevalidní výstup. Rozhodněte prosím ručně:
[surový výstup subagenta]

Možnosti: [u] update  [s] supersede  [m] mark_stale  [c] cancel
```

Pokud uživatel zvolí operaci ručně, sestav minimální volání dle kroku 7 a pokračuj.

5. Pokud je výstup validní, zobraz uživateli návrh s odůvodněním:

```
Návrh pro kandidát "[název]":

  Operace: [DOC_UPDATE / DOC_SUPERSEDE / DOC_MARK_STALE]
  Odůvodnění: [rationale ze subagenta]
  Reason: [reason.type] — [reason.note]

Souhlasíš?
  [y] ano — provést
  [u] změnit na update
  [s] změnit na supersede
  [m] změnit na mark_stale
  [c] cancel — přeskočit
```

6. Zpracování přepsání uživatelem (**bez převolání subagenta**):

- `supersede → update`: obsah `new_doc` použij jako `patch` pro `doc_update` na `old_id`.
  Pokud `new_doc.title` se liší od stávajícího titulku, zobraz varování:
  ```
  ⚠ Nástupce měl jiný titulek. Ten bude zachován beze změny — pokud chcete
    titulek změnit, použijte supersede.
  ```
- `update → supersede`: obsah `patch` použij jako základ `new_doc`. Chybějící povinná
  pole (`title`, `l1_draft`, `tools`, `projects`) doplň z původního dokumentu
  (máš ho z `doc_read` provedeného před delegací).
- `anything → mark_stale`: sestav `doc_mark_stale` s `target_id` a `reason.type: external_input`.

Čekej na odpověď uživatele před dalším krokem.

### Krok 7 — zápis

Podle rozhodnutí (vlastního z kroku 5 nebo potvrzeného v kroku 6):

**`new`** → volej `doc_write`. Sestav vstup podle struktury MCP nástroje (viz `docs-api-spec.md`).

Jako `reason` předej strukturovaný objekt — mapuj kontext volání:

| Kontext | `reason.type` |
|---|---|
| `/make-doc` bez parametrů, spuštěno v Claude Code chatu | `extraction_from_chat` |
| `/make-doc` po bash výstupu nebo runbook exekuci | `extraction_from_bash` |
| Explicitní hint uživatele „importuji" nebo hromadný import | `import` |
| Uživatel píše dokument ručně bez strojového zdroje | `manual` |

Výchozí hodnota je `extraction_from_chat`. Pokud uživatel poskytl hint, respektuj ho.

- `title` — krátký nadpis kandidáta
- `author` — `claude-code`
- `l1_draft`, `l2_draft` — návrh (server přepíše přes LLM normalizaci)
- `body` — markdown tělo s detaily z konverzace
- `tools`, `projects` — extrahuj z konverzace, použij jen kanonické klíče (volitelně ověř přes `doc_list_dictionary`)
- `component_hint` (volitelný): pokud agent v konverzaci explicitně zmínil konkrétní komponentu nebo oblast ("tohle je o panelu Ask"), předej tento hint sem. Jinak vynech pole nebo pošli `null`.
- `links` — případné vazby na další dokumenty (např. `implements` na decision)
- `type_specific` — pole specifická pro typ (status, trigger, host, service podle typu)

Pro decision navíc:
- `chosen: [string]` — slugy vybraných technologií
- `considered: [{tool, reason_short}]` — slugy zamítnutých technologií s důvody (10–120 znaků)

**`update`** (výstup ze subagenta `doc_update`) → volej `doc_update`:
- `id`: `target_id` z JSON návrhu
- `patch`: z JSON návrhu (jen změněná pole — server provede merge)
- `reason`: z JSON návrhu

**`supersede`** (výstup ze subagenta `doc_supersede`) → volej `doc_supersede`:
- `old_id`: z JSON návrhu
- `new_doc`: z JSON návrhu (title, body, l1_draft, l2_draft, tools, projects)
- `reason`: z JSON návrhu
- Server automaticky zdědí `author` ze starého dokumentu, zapíše symetrické log záznamy,
  vytvoří vazbu `supersedes` a změní status starého dokumentu na `superseded`.

> **Pozor na `reason.type` pro supersede.** Povolené typy jsou `due_to_document`, `external_input`, `reorganization` — **ne** `extraction_from_chat`. Pro supersede iniciované z chatu (typický případ v `/make-doc`) použij `external_input`.

**`mark_stale`** (výstup ze subagenta `doc_mark_stale`) → volej `doc_mark_stale`:
- `id`: `target_id` z JSON návrhu
- `reason`: z JSON návrhu

**Při chybě `unknown_tool` ze serveru:**

Server vrací vždy jen první neznámý klíč, spolu s polem `suggestion` (kanonický alias, pokud existuje) a polem `dictionary_add` (hotové parametry pro volání nástroje).

Postup — bez interakce s uživatelem:

1. **Pokud `suggestion` existuje** → nahraď problematický klíč hodnotou `suggestion`, zopakuj `doc_write` s upraveným vstupem.
2. **Pokud `suggestion` neexistuje** → zavolej `dictionary_add` s parametry z `error.dictionary_add` (doplň `label` = původní hodnota klíče před slugifikací). Informuj uživatele: „Přidávám tool '{key}' do slovníku." Pak zopakuj původní `doc_write` beze změn.
3. **Pokud `dictionary_add` selže** → informuj uživatele a přeskoč kandidát.

Server vrací vždy jen první neznámý klíč — smyčka se může opakovat, dokud `doc_write` neprojde.

Pro chyby jiného typu (neznámý project, chyba schématu, chyba vazby) platí původní pravidlo: zkus jednou napravit z `error_code` a `suggestion`, pak informuj uživatele a přeskoč kandidát.


### Krok 8 — průběžné hlášení

Po každém zápisu krátce potvrď výsledek uživateli:

```
✓ decision-volba-mongodb-content-api-3f8c (NEW)
   "Rozhodnutí použít MongoDB jako primární databázi pro Content API."
   Side effects: decision-volba-databaze-7a3f → status superseded

✓ config-mongo-hetzner1-7b2a (UPDATE → doc_update)
   "Konfigurace MongoDB na hetzner-1."
```

Pokud odpověď obsahuje `classification.facets`, zobraz uživateli:
- `layer`: detekovaná vrstva (nebo `null` pokud nezjištěna)
- `components`: seznam komponent
- Varování z `classification.warnings`: každé varování na nový řádek

Pak pokračuj dalším kandidátem (zpět na krok 3).

### Krok 9 — souhrn a nabídka dalšího kola

Po zpracování všech vybraných kandidátů z aktuálního kola udělej souhrn:

```
Hotovo. Zapsáno v tomto kole:

✓ decision-volba-mongodb-content-api-3f8c (NEW)
✓ config-mongo-hetzner1-7b2a (NEW)
⊘ howto-instalace-mongo (přeskočeno — uživatel řekl "cancel")
```

Pak nabídni pokračování:

```
Co dál?
- "pokračuj" — analyzovat zbylé kandidáty z první analýzy (pokud nějací jsou)
- "ještě <popis>" — zaznamenat něco konkrétního, co jsem nenavrhl
- "konec" — ukončit
```

Pokud uživatel řekne "pokračuj" a žádné zbylé kandidáty nejsou, oznam to a ukončí. Pokud řekne "ještě X", vrať se na krok 4 s tímto kandidátem (přeskoč krok 1–3, kandidát už máš).

## Pravidla

1. **Nikdy nezapisuj bez potvrzení.** New dokumenty bez nalezené shody zapisuj rovnou (potvrzení na konci kroku 2 stačí). Update a supersedes vždy potvrď v kroku 6.

2. **Nepiš dokumenty typu `source`.** Surová konverzace má svůj vlastní vytěžovací proces, skill `make-doc` ho neobchází.

3. **Maximálně 5 kandidátů na kolo.** Vyber nejvýznamnější. Zbylé nabídni v dalším kole.

4. **Buď konzervativní u sémantické shody.** Pokud váháš mezi `new` a `update`/`supersedes`, defaultuj na `new` a nech uživatele rozhodnout.

5. **Při chybě validace zkus jednou napravit.** Pak informuj uživatele a přeskoč kandidát.

6. **Nehádej hodnoty slovníků.** Pokud nevíš, jestli `mongodb` nebo `mongo` je kanonický klíč, podívej se přes `doc_list_dictionary`. Server odmítne neznámé hodnoty s návrhem.

7. **Linky vytváří uživatel nebo zjevný kontext.** Skill navrhne `supersedes` (z analýzy). Ostatní vazby (`implements`, `depends_on`, `references`) přidávej jen pokud jsou v konverzaci explicitně řečené nebo zjevné z kontextu (např. config implementující decision).

## Příklady frontmatteru

Konkrétní příklady vstupu pro `doc_write` jsou ve složce `examples/`:

- `examples/decision.md` — rozhodnutí (s `chosen` a `considered`)
- `examples/spec.md` — referenční dokumentace (schémata, pravidla, API)
- `examples/howto.md` — návod (sekvence kroků)
- `examples/config.md` — konfigurace (host, service, stav)
- `examples/runbook.md` — postup řešení incidentu (trigger, severity)
- `examples/tool.md` — dokumentace nástroje nebo technologie (stav, licence)
- `examples/reference.md` — odkaz na externí zdroj (URL, specifikace, RFC)
- `examples/glossary.md` — definice pojmu specifického pro projekt

**Načítej příklady jen podle potřeby** — pokud zpracováváš kandidát typu `decision`, otevři `examples/decision.md`. Není nutné číst všechny.

## Co skill úmyslně neřeší

- **Vytěžování dlouhých session přes LLM** — pokud session má víc než desítky turnů, skill analyzuje jen aktuální logické vlákno. Pro dávkové vytěžování je samostatný proces.
- **Rušení/mazání dokumentů** — skill jen zapisuje a aktualizuje. Mazání je manuální operace.
- **Reorganizaci grafu vazeb** — skill navrhuje `supersedes` z analýzy, ostatní vazby přidává jen pokud jsou zjevné. Údržba grafu je manuální práce.
- **Rozhodování o sloučení duplicit** — pokud najde dva podobné existující dokumenty, neslučuje je. Informuje uživatele a nechá rozhodnout.
