# update-doc

Spusť skill `update-doc` ze složky `skills/update-doc/` v tomto pluginu.

Subagent pro úpravu existujícího dokumentu ve Stratě. Rozhoduje mezi
`update`, `supersede` a `mark_stale`. Může být volaný přímo uživatelem
nebo interně z `make-doc`.

## Přímé volání

`/strata:update-doc` — uživatel cílí na konkrétní dokument a popisuje změnu.
Skill přeskočí fázi vytěžování kandidátů a rovnou pracuje s existujícím dokumentem.
