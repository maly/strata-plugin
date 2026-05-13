# Příklad: `spec`

**Kdy použít:** Pro referenční dokumentaci — co systém dělá, jak je navržen,
jaká jsou pravidla nebo schémata. Spec je "jak to je", ne "proč to tak je"
(to je `decision`) a ne "jak to udělat" (to je `howto`).

Spec typicky navazuje na decision (`implements`) nebo ho superseduje, pokud se
specifikace mění v důsledku nového rozhodnutí.

## Vstup pro `doc_write`

```json
{
  "title": "Schéma frontmatteru pro typ decision",
  "author": "claude-code",
  "l1_draft": "Specifikace polí frontmatteru pro typ decision včetně volitelných polí chosen a considered.",
  "l2_draft": "Povinná pole: status (draft|active|superseded|rejected, default active). Volitelná pole specifická pro decision: chosen (seznam slugů vybraných technologií), considered (seznam objektů s tool a reason_short). Obě pole jsou volitelná — decision bez nich je validní pro organizační nebo procesní rozhodnutí. Server slugifikuje hodnoty tool před zápisem.",
  "body": "# Schéma frontmatteru pro typ `decision`\n\n## Povinná pole (společná pro všechny typy)\n\nViz specifikace společného frontmatteru.\n\n## Pole specifická pro typ `decision`\n\n```yaml\nstatus: draft | active | superseded | rejected  # default: active\nchosen: [string]                                 # volitelné — slugy vybraných technologií\nconsidered:                                      # volitelné — zamítnuté alternativy\n  - tool: string          # slug technologie (server slugifikuje vstup)\n    reason_short: string  # důvod zamítnutí, 10–120 znaků, faktický tón\n```\n\n## Pravidla pro `chosen`\n\n- Obsahuje kanonické klíče ze `tools.yaml`\n- Pokud klíč neexistuje, server ho přidá automaticky\n- Může být prázdné — decision může být organizační bez výběru technologie\n\n## Pravidla pro `considered`\n\n- `tool` se slugifikuje (`Vue.js` → `vuejs`, `React` → `react`)\n- `reason_short` je povinný, 10–120 znaků, bez podmiňování (žádné 'možná', 'asi')\n- Server indexuje do `document_considered` a aktualizuje `considered.yaml`\n- `doc_search` podporuje filtr `considered_tool` pro dotazování zamítnutých technologií\n\n## Automatické side effects\n\nPři zápisu decision s `links: [{ to: X, rel: 'supersedes' }]`:\n- Server změní status dokumentu X z `active` na `superseded`\n- V indexu doplní X jako `superseded_by` nový dokument\n\n## Příklad frontmatteru\n\n```yaml\n---\nid: rozhodnuti-pouzit-svelte-pro-frontend-a3f1\ntype: decision\ntitle: Volba frontendového frameworku\nstatus: active\nchosen: [svelte]\nconsidered:\n  - tool: vue\n    reason_short: \"Menší ekosystém knihoven pro náš případ použití.\"\n  - tool: react\n    reason_short: \"Vyšší komplexita pro dvoučlenný tým.\"\ntools: [svelte]\nprojects: [pruvodce-svetem]\n---\n```",
  "tools": ["mcp"],
  "projects": ["strata"],
  "links": [
    { "to": "rozhodnuti-pridat-pole-chosen-a-considered-do-2141", "rel": "implements" }
  ],
  "type_specific": {
    "status": "approved"
  }
}
```

## Pravidla pro spec

- `l1_draft` — jedna věta popisující co spec specifikuje
- `l2_draft` — strukturovaný výtah: co je povinné, co volitelné, klíčová pravidla
- `body` — kompletní referenční dokumentace, schémata, příklady
- `tools`, `projects` — technologie a projekty, kterých se spec týká
- `links` — typicky `implements` na decision, nebo `supersedes` na starší verzi spec
- `type_specific.status` — `draft` | `approved` | `deprecated`, default `approved`

## Kdy spec superseduje jiný spec

Pokud se schéma nebo pravidla změní (typicky v důsledku nového decision), zapiš
nový spec a propoj vazbou `supersedes` na původní. Původní spec server automaticky
označí jako `deprecated`.

Vzor:
```
nový decision (změna architektury)
    ↓ implements
nový spec (aktualizované schéma)
    ↓ supersedes
původní spec (deprecated)
```
