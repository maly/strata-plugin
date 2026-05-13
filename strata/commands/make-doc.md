# make-doc

Spusť skill `make-doc` ze složky `skills/make-doc/` v tomto pluginu.

Skill analyzuje aktuální konverzaci, navrhne kandidáty na dokumentaci
a po potvrzení uživatele je zapíše do Straty přes MCP nástroje.

## Varianty

- `/strata:make-doc` — analyzuje celou session, navrhne kandidáty
- `/strata:make-doc decision` — zaměří se jen na rozhodnutí
- `/strata:make-doc this` — zaměří se na poslední téma konverzace
- `/strata:make-doc <volný text>` — explicitní zadání ("zapiš instalaci X")
