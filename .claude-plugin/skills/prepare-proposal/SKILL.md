---
name: prepare-proposal
description: "Zkompletuje a odešle proposal (návrh čekající na rozhodnutí) do dokumentačního systému Strata z aktuální konverzace. Spouští se příkazem `/prepare-proposal`. Skill najde v konverzaci hotový návrh (typicky artifact vzniklý brainstormingem), ověří strukturu L2 šablony (Záměr/Rozsah/Dopad) a další náležitosti před odesláním, navrhne tagy a metadata, ukáže uživateli shrnutí a po potvrzení zapíše přes MCP server `strata`. Použij vždy, když uživatel řekne `/prepare-proposal`, `pošli proposal`, `odeslat návrh do Straty`, nebo když po společném brainstormingu chce hotový návrh strukturovaně uložit do dokumentace. NIKDY nezapisuje bez explicitního potvrzení uživatele."
---

# Skill `prepare-proposal`

Skill pro Claude (chat) na kompletní zápis hotového proposalu do dokumentačního systému Strata. Vstupem je obsah aktuální konverzace — typicky artifact, který vznikl iterativním brainstormingem nebo úpravou.

## Princip

Proposal vzniká v chatu jako čistá tvůrčí činnost — uživatel s Claude probere problém, vznikne návrh, projde recenzí, dozraje. V momentě, kdy je hotový, ho je potřeba zaznamenat do Straty jako dokument typu `proposal`. Skill tu poslední fázi obstará:

1. Vezme hotový návrh z konverzace.
2. Ověří, že obsah splní pevné požadavky Straty (struktura L2, délky, validní hodnoty).
3. Předvede uživateli shrnutí.
4. Po potvrzení zavolá `doc_write`.

Skill nikdy nezapisuje bez potvrzení.

## Předpoklady

Skill používá MCP server `strata`. Před prvním krokem ověř, že nástroje `doc_write` a `doc_search` jsou dostupné. Pokud volání MCP vrátí chybu spojení, přeruš tok a oznam uživateli: "MCP server `strata` neodpovídá. Zkontroluj připojení a spusť `/prepare-proposal` znovu."

## Pracovní tok

### Krok 1 — výběr zdroje

Najdi v konverzaci hotový proposal. Postup:

1. **Uživatel explicitně řekl, co použít** (např. „použij ten artifact o XYZ", „pošli tu třetí verzi") → respektuj volbu.
2. **V konverzaci je artifact (markdown soubor s `<artifact>` blokem nebo `present_files`)** → použij nejnovější verzi nejnovějšího artifactu.
3. **Není žádný artifact, ale konverzace končí strukturovaným textem návrhu** → použij ten text.
4. **Není zřejmý zdroj** → zeptej se uživatele: „Nenašel jsem v konverzaci jasný proposal. Co mám použít?"

Pokud je v konverzaci víc kandidátů (víc artifactů, víc verzí), ukaž stručný seznam a nech uživatele vybrat.

### Krok 2 — ověření struktury obsahu

Pro typ `proposal` má Strata pevnou L2 šablonu se třemi povinnými sekcemi:

- **Záměr** — co se navrhuje a proč
- **Rozsah** — čeho se návrh týká a čeho ne
- **Dopad** — očekávaný přínos, rizika nebo navazující kroky

Tyto sekce nemusí být v body dokumentu doslova pojmenované — body může mít vlastní strukturu. Ale **`l2_draft` musí tyto tři sekce obsahovat jako čitelné nadpisy s odpovídajícím obsahem**. Server jinak zápis odmítne s `cannot_satisfy_template`.

Postup:
1. Projdi obsah a najdi pro každou sekci odpovídající materiál:
   - Záměr — typicky první odstavec, motivace, problém, který se řeší
   - Rozsah — sekce „Co se mění", „Schéma", „Rozsah změn", vymezení proti tomu, co se nemění
   - Dopad — sekce „Důsledky", „Přínos", „Rizika", „Navazující kroky"
2. Pokud některá sekce zjevně chybí (ne jen není výslovně pojmenovaná, ale chybí materiál), zeptej se uživatele:
   ```
   V obsahu nevidím jasný materiál pro sekci "Rozsah". Co je v rozsahu/mimo rozsah tohoto návrhu?
   ```

Pokud má obsah všechny tři sekce v některé formě, pokračuj.

### Krok 3 — sestavení polí

Sestav vstup pro `doc_write`:

**`title`** — krátký nadpis. Vytáhni z prvního H1 v artifactu, nebo z prvního zřejmého titulku. Max ~80 znaků. Pokud je titulek delší, zkrať a zachovej smysl.

Odstraň meta-prefixy, které jen oznamují, že jde o návrh — `Proposal:`, `Návrh:`, `Záměr:` na začátku. Server klasifikuje typ sám; takový prefix v titulku je redundantní. Příklad: `Proposal: typ dokumentu implementation` → `Návrh typu dokumentu implementation pro Stratu` (po doplnění kontextu projektu, pokud chybí).

**`l1_draft`** — jedna věta 30–200 znaků. Shrnutí: *co se navrhuje a kde*. Formuluj tak, aby název projektu byl explicitně zmíněn (pomáhá normalizaci serveru).

Příklady L1:
- „Návrh přidání typu dokumentu implementation pro doložení realizace specifikací ve Stratě."
- „Záměr: Přidání typu overview do MCP server Strata - dokumentační systém pro rozcestníky."

**`l2_draft`** — 100–1500 znaků. Tři sekce, oddělené prázdným řádkem, každá uvozená přesným prefixem `Záměr:`, `Rozsah:`, `Dopad:` na začátku řádku. Server hledá tyto přesné fráze v textu. Tělo l2 z obsahu, ale **zhuštěné** — l2 je výtah, ne kopie body.

Šablona (přesný formát):
```
Záměr: <co se navrhuje a proč>.

Rozsah: <čeho se návrh týká a čeho ne>.

Dopad: <očekávaný přínos, rizika nebo navazující kroky>.
```

Pokud výsledné L2 přesahuje 1500 znaků, zkrať. Pokud je pod 100, doplň podstatu (ne výplň).

**`body`** — markdown tělo. Použij obsah artifactu jak je. Server normalizuje l1/l2, ale body bere doslova.

**`author`** — `claude-chat` (skill běží v chatu, ne v Claude Code).

**`reason`**:
```json
{
  "type": "extraction_from_chat",
  "note": "Návrh vznikl iterativní diskusí v chatu, čeká na rozhodnutí."
}
```
`ref` u `extraction_from_chat` je volitelné — nevyplňuj.

### Krok 4 — projekty

Z obsahu odvoď, kterých projektů se proposal týká. Typické signály:
- explicitní zmínka názvu projektu v titulku nebo v prvním odstavci
- názvy komponent vázané na konkrétní projekt
- kontext, ze kterého konverzace vyšla

Návrh `projects` ukaž uživateli k potvrzení:
```
Navrhuji projekty: ["strata-server"]. OK, nebo upravit?
```

**Důležité:** projekty musí být ze slovníku. Pokud si nejsi jistý slugem, ověř přes `doc_list_dictionary` s parametrem `{"dictionary": "projects"}` — dostaneš kanonický seznam. Pokud server přesto odmítne `unknown_project`, ukáže `valid_values` — vyber z nich a zopakuj.

**`tools`** u proposalu se obvykle neřeší. Nech prázdné, pokud obsah explicitně nezmiňuje konkrétní technologii jako předmět návrhu (vzácné).

### Krok 5 — kontrola duplicit

Před zápisem proveď `doc_search` s `type: ["proposal"]` a `query` z titulku (1–3 klíčová slova). Cíl: zachytit, že podobný proposal už existuje.

Pokud najde podobné, ukaž uživateli:
```
Pozor, ve Stratě je už podobný proposal:
  navrh-pridani-typu-overview-do-mcp-server-strata-f6de:
    L1: "Návrh přidání typu overview do MCP server Strata..."

Pokračovat se zápisem nového, nebo zrušit?
```

Skill duplicitu **neblokuje** — jen upozorňuje. Rozhoduje uživatel.

### Krok 6 — shrnutí před zápisem

Ukaž uživateli kompletní náhled:

```
Připraveno k zápisu do Straty:

  title: Návrh typu dokumentu implementation pro Stratu
  l1:    Návrh přidání dvanáctého typu dokumentu implementation pro doložení
         realizace specifikací ve Stratě.
  l2:    Záměr: Zavést dvanáctý typ dokumentu implementation jako doklad realizace...
         Rozsah: Týká se přidání jednoho nového typu se striktními invarianty...
         Dopad: Strata získá strukturovanou odpověď na otázku "je spec realizovaná?"...
  projects: ["strata-server"]
  body: 248 řádků markdownu
  author: claude-chat
  reason: extraction_from_chat

Zapsat? [y/n]
```

Čekej na potvrzení. Bez `y` nezapisuj.

### Krok 7 — zápis a chyby

Po potvrzení zavolej `doc_write` se sestaveným vstupem.

**Při úspěchu** ohlas:
```
✓ navrh-pridani-typu-dokumentu-implementation-pro-8ecf (proposal)
  Klasifikace: proposal (confidence: high)
  Path: docs/proposals/navrh-pridani-typu-dokumentu-implementation-pro-8ecf.md
```

Pokud klasifikace má varování (`classification.warnings`) nebo alternativy s vyšší než nulovou jistotou, ukaž je uživateli.

**Při chybě podle error_code:**

| `error_code` | Postup |
|---|---|
| `cannot_satisfy_template` | Server odmítl L2 — vrať se ke kroku 2, ukaž chybovou zprávu, navrhni úpravu. |
| `l1_normalization_failed` | L1 je mimo 30–200 znaků. Zkrať/rozšiř a zkus znovu (max 1×). |
| `l2_normalization_failed` | L2 je mimo 100–1500 znaků nebo nesplňuje šablonu. Vrať se ke kroku 3. |
| `unknown_project` | Server vrátí `valid_values`. Ukaž uživateli, nech vybrat, zopakuj. |
| `unknown_tool` | Stejně jako `unknown_project`. Pro proposal je tools vzácné — pokud nastane, pravděpodobně chyba v rozpoznání. |
| jiné | Ukaž `message` a `suggestion`, nech uživatele rozhodnout. |

Pokud chyba neumožní opravu, oznam: „Zápis se nezdařil. Detail chyby: [...]. Můžeš upravit a spustit `/prepare-proposal` znovu."

## Pravidla

1. **Nikdy nezapisuj bez potvrzení uživatele v kroku 6.**

2. **L1/L2 délky a struktura jsou tvrdé požadavky serveru.** Když je nesplníš, server odmítne. Skill je ověřuje *před* voláním, aby chybu zachytil dřív.

3. **L2 musí mít sekce uvozené přesně `Záměr:`, `Rozsah:`, `Dopad:`.** Server hledá tyto klíčové fráze v textu.

4. **Body bere server doslova — nepřepisuj ho ani nezkracuj.** Normalizace probíhá jen v L1 a L2.

5. **Projekty ze slovníku, ne výmysl.** Při nejistotě se zeptej uživatele nebo zavolej `doc_list_dictionary` s `{"dictionary": "projects"}` — dostaneš kanonický seznam slugů.

6. **Kontrola duplicit upozorňuje, neblokuje.** Někdy je „podobný" proposal vlastně jiný — rozhodnutí má uživatel.

7. **`reason.type` je fixní:** `extraction_from_chat` s notou „Návrh vznikl iterativní diskusí v chatu, čeká na rozhodnutí." Pokud uživatel řekne, že to byl manual zápis nebo import, respektuj — ale defaultně `extraction_from_chat`.

## Co skill neřeší

- **Vznik proposalu samotného** — skill nepředpokládá, že vede brainstorming. Bere hotový obsah z konverzace. Brainstorming je věc předchozí fáze chatu.
- **Update existujícího proposalu** — pokud uživatel chce existující proposal upravit, je na to `make-doc` se subagentem `update-doc`.
- **Jiné typy dokumentů** — skill je úzký na `proposal`. Pro decision, spec, howto apod. použij `make-doc`.
- **Brainstorming verzí** — pokud uživatel chce dva alternativní proposaly, ať si je sám rozhodne v konverzaci; skill zapíše ten, co řekne.
- **Mazání nebo rušení** — skill jen zapisuje. Mazání je manuální operace.

## Příklad běhu

**Uživatel:** `/prepare-proposal`

**Skill:** Vidím v konverzaci artifact `proposal-implementation-type.md` (návrh dvanáctého typu dokumentu pro Stratu). Použiju ten.

L2 sekce ověřuji:
  ✓ Záměr (zavést typ implementation jako doklad realizace spec)
  ✓ Rozsah (přidání typu, invarianty, odvozené pole, žádný nový typ vazby)
  ✓ Dopad (strukturovaná odpověď „je spec hotová?", dopad na make-doc, vztah k realized_by)

Navrhuji projekty: `["strata-server"]`. Z titulku a obsahu je zřejmé, že jde o Stratu.

Hledám duplicity... [doc_search proposal "implementation"] ... nic podobného.

Připraveno k zápisu:
```
title: Návrh typu dokumentu implementation pro Stratu
l1: Návrh přidání dvanáctého typu dokumentu implementation pro doložení realizace specifikací ve Stratě.
l2: Záměr: Zavést dvanáctý typ dokumentu... Rozsah: Týká se přidání... Dopad: Strata získá...
projects: ["strata-server"]
body: 248 řádků markdownu
author: claude-chat
reason: extraction_from_chat
```

Zapsat? `y/n`

**Uživatel:** `y`

**Skill:** [doc_write...] ✓ `navrh-pridani-typu-dokumentu-implementation-pro-8ecf` zapsán.
