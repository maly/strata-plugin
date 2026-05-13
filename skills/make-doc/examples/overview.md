# Příklad: overview

Přehled propojuje více existujících interních dokumentů do jedné mapy tématu.
Použij ho, když už existují alespoň dva dokumenty, které stojí za společný vstupní bod.

## Vstup pro `doc_write`

```json
{
  "title": "Přehled dokumentace k typu overview ve Stratě",
  "author": "claude-code",
  "l1_draft": "Přehled změn, pravidel a navazujících kroků pro dokumentový typ overview ve Stratě.",
  "l2_draft": "Rozsah: typ overview v docs-serveru a meta pravidlech. Klíčové dokumenty: plán implementace, rozhodnutí o poli covers, meta šablony. Kdy číst: před tvorbou nebo úpravou overview dokumentů.",
  "body": "## Rozsah\n\nTento přehled mapuje zavedení typu `overview`...\n\n## Klíčové dokumenty\n\n- `plan-overview-type-2026` — implementační plán\n- `decision-overview-covers` — rozhodnutí o povinném poli `covers`\n\n## Doporučený postup\n\nZačni plánem, potom zkontroluj aktuální meta pravidla a skilly.",
  "tools": ["strata"],
  "projects": ["strata"],
  "scope": "Typ overview ve Strata dokumentaci a serveru",
  "covers": [
    "plan-overview-type-2026",
    "decision-overview-covers"
  ],
  "reason": { "type": "extraction_from_chat" }
}
```

## Pravidla pro overview

- `scope` je povinný a musí jasně vymezit oblast, kterou overview pokrývá.
- `covers` je povinné pole s minimálně dvěma existujícími ID dokumentů.
- `covers` nevkládej do `links`; server z něj vytvoří interní vazby typu `covers`.
- Overview není proposal ani spec: nenavrhuje změnu a nedefinuje nové pravidlo, jen mapuje existující dokumenty.
- Při update zachovej původní `covers`, pokud se pokrytí tématu nezměnilo.
