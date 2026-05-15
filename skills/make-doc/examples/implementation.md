# Příklad: implementation

Retrospektivní záznam realizace existující specifikace. Vzniká až poté, co spec existuje a implementace proběhla (nebo probíhá). Musí obsahovat právě jeden `implements` link na cílovou spec.

## Vstup pro `doc_write`

```json
{
  "title": "Implementace webhook endpointu podle spec git-hooks",
  "author": "claude-code",
  "l1_draft": "Realizace specifikace git-hooks — webhook endpoint pro příjem push událostí.",
  "l2_draft": "Spec git-hooks definuje HMAC ověření a zpracování push eventů. Endpoint POST /api/hooks/git implementován v src/routes/hooks.js. Stav: complete.",
  "body": "## Co bylo implementováno\n\nEndpoint `POST /api/hooks/git` přijímá GitHub push webhooky, ověřuje HMAC podpis a spouští git pull v příslušném repozitáři.\n\n## Odchylky od spec\n\nŽádné — implementace odpovídá spec přesně.\n\n## Stav\n\nNasazeno a funkční od 2026-05-09.",
  "tools": ["node", "git"],
  "projects": ["docs-server"],
  "links": [
    { "rel": "implements", "to": "spec-git-hooks-ab12" }
  ],
  "type_specific": {
    "status": "complete"
  },
  "reason": { "type": "extraction_from_chat" }
}
```

**Poznámky:**
- `links` s `implements` je povinný — server odmítne zápis bez něj.
- Cílová spec musí existovat; ID ověř přes `doc_search` nebo `doc_read` před zápisem.
- `status` je `ongoing` (výchozí) nebo `complete`. Vynechání pole = `ongoing`.
- Jedna spec může mít nejvýše jednu implementaci — při pokusu o druhou server vrátí `implementation_already_exists`.

## Aktualizace stavu — ongoing → complete

```json
{
  "id": "implementation-webhook-endpoint-cd34",
  "type_specific": { "status": "complete" },
  "reason": { "type": "refinement", "note": "Implementace dokončena a nasazena." }
}
```
