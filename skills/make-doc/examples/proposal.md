# Příklad: proposal

Záměr čekající na rozhodnutí. Popisuje co a proč — ne jak (to patří do spec/howto po schválení).

## Vstup pro `doc_write`

```json
{
  "title": "Záměr přidat typ dokumentu proposal do Strata systému",
  "author": "claude-code",
  "l1_draft": "Záměr přidat nový typ dokumentu proposal pro záměry čekající na rozhodnutí.",
  "l2_draft": "Záměr: přidat typ proposal. Motivace: záměry vznikají organicky ale žádný typ jim nesedí. Dopad: server, klasifikátor, UI. Stav: current.",
  "body": "## Motivace\n\nStrata nemá místo pro dokumenty popisující záměr něco udělat...\n\n## Návrh\n\nNový typ `proposal` s polem `realized_by`...\n\n## Dopad\n\nZměny v serveru, klasifikátoru a skilly.",
  "tools": ["strata"],
  "projects": ["strata"],
  "reason": { "type": "extraction_from_chat" }
}
```

**Poznámka:** `realized_by` se nevyplňuje při vzniku — záměr teprve čeká na rozhodnutí.

## Po realizaci — update s `realized_by`

```json
{
  "id": "zamer-pridat-typ-proposal-ab12",
  "realized_by": ["spec-proposal-typ-cd34", "rozhodnuti-pridat-proposal-ef56"],
  "reason": { "type": "refinement", "note": "Záměr byl realizován — doplněny vazby na výsledné dokumenty." }
}
```
