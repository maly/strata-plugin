# Příklad: `reference`

**Kdy použít:** Pro odkaz na externí zdroj, který je relevantní pro projekt —
specifikace, RFC, dokumentace, článek, repozitář. `reference` říká "toto existuje
a je to relevantní", ne co to konkrétně říká (to patří do body).

Hodí se pro standardy (MCP specifikace, RFC), oficiální dokumentaci knihoven,
nebo klíčové články a přednášky, které ovlivnily architektonická rozhodnutí.

## Vstup pro `doc_write`

```json
{
  "title": "Model Context Protocol — oficiální specifikace",
  "author": "claude-code",
  "l1_draft": "Oficíální specifikace protokolu MCP (Model Context Protocol) od Anthropic, na které staví Strata MCP server.",
  "l2_draft": "Specifikace definuje stdio a SSE transport, formát JSON-RPC zpráv, lifecycle inicializace, schéma nástrojů (tools/call, tools/list) a chybové kódy. Strata server implementuje MCP server roli přes @modelcontextprotocol/sdk. Specifikace je živý dokument — verzování přes datum v protocolVersion poli.",
  "body": "# Model Context Protocol — specifikace\n\n## URL\n\nhttps://spec.modelcontextprotocol.io/\n\n## Co to je\n\nOtevřený protokol od Anthropic pro komunikaci mezi LLM klienty (Claude Code, Claude Desktop) a MCP servery. Definuje:\n\n- **Transport:** stdio (lokální procesy) a SSE (síťové servery)\n- **Zprávy:** JSON-RPC 2.0 — request/response/notification\n- **Lifecycle:** initialize → initialized → tools/list → tools/call\n- **Nástroje:** schéma vstupu přes JSON Schema, výstup jako pole content bloků\n- **Chyby:** standardní JSON-RPC error kódy + MCP-specifická rozšíření\n\n## Relevance pro Stratu\n\nStrata server implementuje MCP server roli. Klíčové části specifikace:\n- Sekce 'Tools' — schéma pro registraci `doc_write`, `doc_search` atd.\n- Sekce 'Stdio transport' — jak server čte ze stdin a píše na stdout\n- Sekce 'Error handling' — jak mapovat validační chyby na MCP error response\n\n## SDK\n\nOficiální Node.js SDK: `@modelcontextprotocol/sdk`\nhttps://github.com/modelcontextprotocol/typescript-sdk\n\n## Verze\n\nStrata používá `protocolVersion: '2024-11-05'` (viz initialize handshake v testech).",
  "tools": ["mcp"],
  "projects": ["strata"],
  "links": [],
  "type_specific": {
    "status": "active",
    "url": "https://spec.modelcontextprotocol.io/"
  }
}
```

## Pravidla pro reference

- `l1_draft` — jedna věta: co to je + proč je to relevantní pro tento projekt
- `l2_draft` — co zdroj obsahuje, které části jsou relevantní, stav (živý dokument?)
- `body` — URL, shrnutí obsahu, relevantní sekce, verze
- `type_specific.url` — **povinné**, přímý odkaz na zdroj
- `type_specific.status` — `active` | `archived`, default `active`
- `tools`, `projects` — kontext relevance (pro jaký projekt/technologii je zdroj důležitý)
- `links` — typicky žádné, nebo `referenced_by` ze strany jiných dokumentů

## Kdy reference přechází do stavu `archived`

Když zdroj přestane být dostupný nebo relevantní — zastaralá verze specifikace,
smazaný článek, deprecated API dokumentace. `archived` reference zůstává v systému
jako historický záznam.
