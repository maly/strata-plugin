# Příklad: `decision`

**Kdy použít:** Když došlo k volbě mezi alternativami — technologie, architektura,
přístup. Decision je "co a proč", ne "jak" (to je `howto`).

## Vstup pro `doc_write`

```json
{
  "title": "MongoDB pro Content API",
  "author": "claude-code",
  "l1_draft": "Rozhodnutí použít MongoDB jako primární databázi pro Content API místo MySQL.",
  "l2_draft": "Zvažovali jsme PostgreSQL, MySQL a MongoDB. Vybrali jsme MongoDB kvůli flexibilitě schématu pro různorodý obsah a horizontální škálovatelnosti. PostgreSQL měl výhodu v transakcích, ale pro náš případ použití nepotřebujeme striktní ACID napříč kolekcemi. MySQL byl vyřazen kvůli složitější práci s vnořenými strukturami.",
  "body": "# Volba databáze pro Content API\n\n## Kontext\n\nContent API potřebuje ukládat obsah s různorodou strukturou — články, podcasty, videa. Schéma se mění v čase podle typů obsahu.\n\n## Zvažované možnosti\n\n### PostgreSQL\nVýhody: ACID, JSON sloupce, široký ekosystém.\nNevýhody: Migrace schématu při změnách typů obsahu.\n\n### MySQL\nVýhody: Tým má zkušenosti.\nNevýhody: Slabší práce s vnořenými strukturami.\n\n### MongoDB\nVýhody: Flexibilní schéma, horizontální škálování, agregační pipeline.\nNevýhody: Eventual consistency, vyžaduje pozornost u vztahů.\n\n## Rozhodnutí\n\nMongoDB s replica setem rs0 na hetzner-1.\n\n## Důsledky\n\n- Datový model definovat v aplikační vrstvě (Mongoose není použit, viz separátní rozhodnutí)\n- Transakce jen tam, kde jsou opravdu nutné\n- Indexy navrhovat podle dotazů, ne podle struktury",
  "tools": ["mongo"],
  "projects": ["content-api"],
  "links": [
    { "to": "decision-databaze-content-api-7a3f", "rel": "supersedes" }
  ],
  "type_specific": {
    "status": "active"
  },
  "chosen": ["mongo"],
  "considered": [
    {
      "tool": "postgresql",
      "reason_short": "Migrace schématu při změnách typů obsahu by byla častá."
    },
    {
      "tool": "mysql",
      "reason_short": "Slabší práce s vnořenými strukturami obsahu."
    }
  ]
}
```

## Pravidla pro decision

- `l1_draft` — jedna věta s rozhodnutím a kontextem (bude přepsáno serverem)
- `l2_draft` — strukturovaný výtah s důvody (bude přepsáno serverem)
- `body` — plný kontext, zvažované možnosti, důsledky
- `chosen` — slugy vybraných technologií, server přidá do `tools.yaml`
- `considered` — slugy zamítnutých technologií s důvody (10–120 znaků), server přidá
  do `considered.yaml`
- `links` se vztahem `supersedes` — pokud nahrazuje starší rozhodnutí, server
  automaticky změní status starého dokumentu na `superseded`
- `type_specific.status` — `draft` | `active` | `superseded` | `rejected`, default `active`

## Decision bez technologií

Organizační nebo procesní rozhodnutí (např. "Code review je povinný před merge")
nemá `chosen` ani `considered` — obě pole jsou volitelná. `tools` může být prázdné,
`projects` většinou ano.
