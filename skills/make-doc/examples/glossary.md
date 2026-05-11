# Příklad: `glossary`

**Kdy použít:** Pro definici pojmu specifického pro tento projekt nebo obor,
který se opakovaně objevuje v dokumentaci a jehož přesný význam stojí za zaznamenání.
`glossary` řeší situaci, kdy tým používá termín, ale každý ho chápe trochu jinak.

Nepiš glossary pro obecně známé pojmy (co je JSON, co je REST API) — jen pro
pojmy, kde existuje projektová specifika nebo riziko záměny.

## Vstup pro `doc_write`

```json
{
  "title": "L1 / L2 / L3 — vrstvy obsahu dokumentu",
  "author": "claude-code",
  "l1_draft": "Definice třívrstvého modelu obsahu dokumentu ve Stratě: L1 je jednořádkový výtah, L2 je strukturovaný souhrn, L3 je plné tělo.",
  "l2_draft": "L1: 1–2 věty, max 200 znaků, generuje server přes LLM normalizaci z l1_draft agenta. Slouží jako primární identifikátor pro vyhledávání a přehled. L2: strukturovaný výtah 200–1500 znaků, šablona per typ (V1/V2/V3 pro decision, kroky pro howto atd.), generuje server. L3: plné markdown tělo, zapisuje agent, server ho přebírá beze změny. Třívrstvý model umožňuje různé úrovně detailu v doc_search (l1 rychle, l2 kontext, full vše).",
  "body": "# L1 / L2 / L3 — vrstvy obsahu dokumentu\n\n## Definice\n\n**L1** — jednořádkový výtah\n- Délka: 1–2 věty, 60–200 znaků\n- Generuje: server přes LLM formátor z `l1_draft` agenta\n- Začíná: kanonickým slovem pro typ (Rozhodnutí, Specifikace, Konfigurace, Návod...)\n- Obsahuje: klíčové entity z `tools` a `projects` (v label podobě ze slovníku)\n- Použití: `doc_search` výsledky, `doc_links` přehled sousedů, L1 index\n\n**L2** — strukturovaný souhrn\n- Délka: 200–1500 znaků\n- Generuje: server přes LLM formátor z `l2_draft` agenta\n- Šablona: per typ (viz specifikace L2 šablon)\n  - `decision`: V1/V2/V3 chronologická struktura\n  - `howto`: Pro koho / Kroky / Časté pasti\n  - `config`: Stav / Síťové parametry / Připojení\n  - `runbook`: Trigger / Předpoklady / Hlavní kroky\n- Použití: `doc_read` level=l2 (default), sémantické vyhledávání, make-doc analýza shody\n\n**L3** — plné tělo\n- Délka: bez omezení\n- Generuje: agent, server přebírá beze změny\n- Formát: Markdown\n- Použití: `doc_read` level=full, lidské čtení v editoru\n\n## Proč třívrstvý model\n\nAgent potřebuje různé úrovně detailu v různých situacích:\n- Rychlý přehled co existuje → `doc_search` vrací L1\n- Kontext pro rozhodnutí → `doc_read` level=l2\n- Kompletní informace → `doc_read` level=full\n\nBez vrstev by byl buď každý search pomalý (přenáší celé tělo) nebo každý detail chybějící.",
  "tools": ["mcp"],
  "projects": ["strata"],
  "links": [
    { "to": "specifikace-pravidel-pro-l1-a-sablon-l2-v-mcp-687f", "rel": "references" }
  ],
  "type_specific": {
    "status": "active",
    "term": "L1 / L2 / L3"
  }
}
```

## Pravidla pro glossary

- `l1_draft` — jedna věta: definice pojmu v kontextu projektu
- `l2_draft` — rozvinutá definice s nuancemi, odlišením od příbuzných pojmů
- `body` — kompletní definice s příklady, proč pojem existuje, jak se používá
- `type_specific.term` — **povinné**, definovaný pojem (přesně jak se používá v dokumentaci)
- `type_specific.status` — jen `active`, glossary nemá jiné statusy
- `links` — typicky `references` na spec nebo decision, kde pojem vznikl

## Kdy napsat glossary

Dobrý signál: v konverzaci se opakovaně používá termín a není jasné, co přesně znamená.
Nebo: dva lidé diskutují o tématu a ukáže se, že pojmem myslí různé věci.

Špatný signál: pojem je obecně známý mimo projekt (REST, JSON, Docker image).
