# Příklad: `tool`

**Kdy použít:** Pro dokumentaci konkrétního nástroje, technologie nebo služby,
kterou tým používá nebo hodnotí. `tool` popisuje nástroj jako entitu — co to je,
proč to používáme, stav, licence. Není to návod na použití (to je `howto`) ani
rozhodnutí o výběru (to je `decision`).

Typicky vzniká jako doplněk k `decision` — decision říká "vybrali jsme X", `tool`
říká "X je toto a takhle ho používáme".

## Vstup pro `doc_write`

```json
{
  "title": "better-sqlite3",
  "author": "claude-code",
  "l1_draft": "Synchronní SQLite knihovna pro Node.js používaná jako indexovací vrstva Strata serveru.",
  "l2_draft": "Knihovna better-sqlite3 poskytuje synchronní API pro SQLite v Node.js. Používáme ji místo async alternativ (node-sqlite3, sql.js) kvůli jednoduchosti v kontextu MCP serveru s stdio transportem. Licence: MIT. Stav: aktivně používaná v produkci jako jediná databázová vrstva.",
  "body": "# better-sqlite3\n\n## Co to je\n\nNativní Node.js bindings pro SQLite s plně synchronním API. Na rozdíl od async alternativ nepoužívá callbacks ani promises — volání jsou blokující a vracejí výsledek přímo.\n\n## Proč to používáme\n\nV MCP serveru s stdio transportem je synchronní API výhoda: odpadá nutnost řešit async/await u každé DB operace. Server zpracovává požadavky sekvenčně, souběžnost není potřeba.\n\nVizualizace výhody:\n```js\n// better-sqlite3 — přímočaré\nconst row = db.prepare('SELECT * FROM documents WHERE id = ?').get(id);\n\n// node-sqlite3 — callback hell\ndb.get('SELECT * FROM documents WHERE id = ?', [id], (err, row) => { ... });\n```\n\n## Instalace\n\n```bash\nnpm install better-sqlite3\n```\n\nVyžaduje nativní kompilaci (node-gyp). Na Alpine Linux (Docker) je potřeba:\n```bash\napk add python3 make g++\n```\n\n## Klíčové vlastnosti relevantní pro Stratu\n\n- FTS5 podpora — virtuální tabulky pro fulltext\n- Rekurzivní CTE — pro grafové průchody v `doc_links`\n- WAL mode — pro lepší souběžnost čtení při zápisu\n- Transakce — atomické operace pro `doc_write` pipeline\n\n## Verze a kompatibilita\n\n- Vyžaduje Node.js 12+\n- SQLite verze bundlovaná s knihovnou (není závislost na systémovém SQLite)\n- Verze: viz `package.json`\n\n## Licence\n\nMIT — bez omezení pro komerční použití.",
  "tools": ["sqlite", "nodejs"],
  "projects": ["strata"],
  "links": [
    { "to": "rozhodnuti-pouzit-sqlite-jako-indexovaci-vrstvu-55b8", "rel": "implements" }
  ],
  "type_specific": {
    "status": "in-use",
    "license": "MIT"
  }
}
```

## Pravidla pro tool

- `l1_draft` — jedna věta: co to je + k čemu to používáme v kontextu projektu
- `l2_draft` — stručný popis, proč to používáme, stav, licence
- `body` — detaily: instalace, klíčové vlastnosti, verze, příklady použití
- `links` — typicky `implements` na decision, který tento nástroj vybral
- `type_specific.status` — `in-use` | `evaluating` | `retired`, default `in-use`
- `type_specific.license` — volitelné, SPDX identifikátor (MIT, Apache-2.0, GPL-3.0…)
- `type_specific.expires` — volitelné, ISO8601 datum expirace licence (pro komerční nástroje)

## Kdy tool přechází do stavu `retired`

Když tým přestane nástroj používat — typicky v důsledku nového decision který vybral
náhradu. Starý `tool` dostane status `retired`, nový `tool` status `in-use`.
`retired` dokumenty zůstávají v systému jako historický záznam.
