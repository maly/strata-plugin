---
name: prepare-doc
description: Připraví návrh slovníků (tools, projects) pro dokumentační systém Strata na základě manifest souborů projektu (package.json, pyproject.toml, Cargo.toml, go.mod, docker-compose.yml) a aktuální konverzace. Spouští se jednou na začátku projektu příkazem `/prepare-doc`. Vyrobí soubor `.claude/dictionary-suggestions.yaml`, který si uživatel sám zkopíruje do `docs-repo/meta/dictionaries/`. Nezapisuje do `docs-repo` přímo.
---

# Skill `prepare-doc`

Úvodní bootstrap slovníků pro dokumentační systém Strata. Spouští se **jednou** na začátku projektu, podobně jako `/init`. Po vykopírování návrhu do `docs-repo` lze používat `/make-doc` pro běžné zápisy.

## Princip

Skill **nezapisuje do `docs-repo`**. Jen vyrobí návrh ve složce `.claude/` aktuálního projektu. Uživatel si návrh projde, upraví a manuálně zkopíruje relevantní položky do `tools.yaml` a `projects.yaml` v `docs-repo`. Skill je generátor návrhu, ne automatický zapisovatel.

## Předpoklady

Skill používá MCP server `strata` jen pro čtení — `doc_list_dictionary` pro zjištění aktuálního stavu slovníků. Pokud server neodpovídá, přeruš tok a oznam: "MCP server `strata` neodpovídá. Zkontroluj připojení a spusť `/prepare-doc` znovu."

## Pracovní tok

### Krok 1 — kontrola, zda je bootstrap potřeba

Zavolej `doc_list_dictionary` pro `tools` a `projects`. Pokud projekt z aktuálního kontextu (viz krok 2) je už v `projects.yaml`, oznam:

```
Projekt "<jméno>" je už ve slovníku projects. 
Bootstrap slovníků není potřeba — můžeš rovnou používat /make-doc.

Pokud chceš doplnit chybějící technologie nebo informace, pokračuj manuálně 
úpravou docs-repo/meta/dictionaries/.
```

A skončí. Skill je idempotentní — opakované spuštění na již připraveném projektu nezpůsobí škodu.

### Krok 2 — detekce projektu

Najdi název projektu v tomto pořadí preference:

1. `package.json` → pole `name` (pokud začíná `@scope/`, použij část za lomítkem)
2. `pyproject.toml` → `[project] name`
3. `Cargo.toml` → `[package] name`
4. `go.mod` → první řádek `module <path>`, použij poslední segment cesty
5. Název kořenové složky projektu

Pokud najdeš víc zdrojů s odlišnými názvy, zeptej se uživatele:

```
Našel jsem víc možných názvů projektu:
1. "strata" (z package.json)
2. "strata-server" (z názvu složky)

Který použít? [1/2/jiný]
```

K projektu z `package.json` přidej `description` jako kandidát na `label`.

### Krok 3 — vytěžení technologií

Projdi manifest soubory v projektu a sesbírej technologie. Pro každou si pamatuj:
- původní název (před slugifikací)
- zdroj (který soubor a které pole)
- kategorii (core / utility / runtime — viz dále)

**Manifest soubory a co z nich brát:**

`package.json`:
- `dependencies` a `devDependencies` — názvy balíčků (klíče objektu)
- `engines` — runtime (`node`, `npm`) jako kategorie `runtime`

`pyproject.toml` (PEP 621):
- `[project] dependencies` — Python balíčky
- `[project] requires-python` — runtime kategorie

`requirements.txt`:
- Každá řádka, ignoruj komentáře a verze (vezmi jen jméno před `==` / `>=` / `~=`)

`Cargo.toml`:
- `[dependencies]` a `[dev-dependencies]` — Rust crates

`go.mod`:
- `require (...)` blok — Go moduly (vezmi poslední segment cesty)

`docker-compose.yml` / `compose.yml`:
- Pro každou službu pole `image:` — část před `:` (verze pryč)
- Pole `services:` klíče jako kandidát na lokální služby (mongo, redis, postgres)

`Dockerfile`:
- `FROM <image>` — base image (jen jméno před `:`)

`.tool-versions`, `.nvmrc`, `.python-version`:
- Runtime verze, kategorie `runtime`

**Doplnění z konverzace:**

Po projití manifest souborů projdi aktuální konverzaci. Pokud uživatel explicitně zmínil technologii, která v manifest souborech není (typicky externí služby — Hetzner, AWS S3, Cloudflare, nebo třeba `nginx` jako reverse proxy), přidej ji s kategorií `core` a zdrojem `konverzace`.

### Krok 4 — kategorizace

Každou technologii zařaď do jedné ze tří kategorií podle těchto pravidel:

**Kategorie `runtime`:**
- Pochází z `engines`, `requires-python`, `.tool-versions`, `.nvmrc`, `.python-version`
- Nebo název odpovídá `node`, `nodejs`, `python`, `ruby`, `go`, `rust`, `java`, `bun`, `deno`

**Kategorie `core` (spíš stojí za zápis do slovníku):**
- Název obsahuje nebo končí na: `server`, `framework`, `client`, `driver`, `orm`, `db`
- Známé frameworky a runtimy: `express`, `fastify`, `koa`, `hono`, `next`, `nuxt`, `react`, `vue`, `svelte`, `astro`, `flask`, `django`, `fastapi`, `actix`, `tokio`, `gin`, `echo`
- Databáze a storage: `mongo`, `mongodb`, `postgres`, `postgresql`, `mysql`, `redis`, `sqlite`, `elasticsearch`, `clickhouse`, `kafka`, `rabbitmq`
- Infrastrukturní nástroje: `docker`, `kubernetes`, `nginx`, `traefik`, `terraform`, `ansible`
- Cloud služby: vše s prefixem `@aws-sdk/`, `@google-cloud/`, `@azure/`
- LLM a AI klienti: `@anthropic-ai/sdk`, `openai`, `langchain`
- Z `docker-compose.yml` `image:` a `Dockerfile` `FROM` — vždy `core`
- Pochází z konverzace (uživatel je explicitně zmínil) — vždy `core`

**Kategorie `utility` (pravděpodobně zbytečné):**
- Vše ostatní — typicky `lodash`, `dotenv`, `chalk`, `commander`, `cors`, `helmet`, `axios`, drobné utility, pomocné knihovny

Když si nejsi jistý, zařaď do `core` — uživatel to v reviewu spíš smaže, než aby přidal něco, co skill přehlédl.

### Krok 5 — filtrace proti existujícímu slovníku

Z výsledku `doc_list_dictionary` pro `tools` máš seznam existujících klíčů (a aliasů). Pro každou technologii ze sběru:
- Slugifikuj název (`Vue.js` → `vuejs`, `better-sqlite3` → `better-sqlite3`)
- Pokud slug nebo původní název **odpovídá existujícímu key nebo aliasu**, vyfiltruj
- Co zbude, jde do výstupu

Stejně postupuj pro projekt — pokud je už ve slovníku (zachycené v kroku 1), do výstupu ho nedávej.

### Krok 6 — zápis výstupu

Zapiš soubor `.claude/dictionary-suggestions.yaml`. Pokud složka `.claude/` neexistuje, vytvoř ji. Pokud soubor už existuje, **přepiš ho** (běh `prepare-doc` nahrazuje předchozí návrh).

**Formát výstupu:**

```yaml
# Návrh doplnění slovníků pro projekt <name>
# Vygenerováno skillem prepare-doc dne <ISO datum>
#
# Postup:
# 1. Zkontroluj a uprav níže uvedené položky (smaž, co nechceš; doplň label kde chybí)
# 2. Zkopíruj sekci 'projects' do docs-repo/meta/dictionaries/projects.yaml
# 3. Zkopíruj sekci 'tools' do docs-repo/meta/dictionaries/tools.yaml
#    (jen položky bez prefixu # utility, případně i ty které potřebuješ)
# 4. Commit do docs-repo: git commit -am "dictionaries: bootstrap pro <name>"
# 5. Reload Strata serveru: POST na /admin/reload nebo `docs-server reindex --notify`

projects:
  - key: <slug>
    label: <label>
    # zdroj: <kde nalezeno>

tools:
  # === core (doporučeno k zařazení) ===

  - key: <slug>
    label: <label>
    # zdroj: <kde nalezeno>
    # ...

  # === runtime ===

  - key: <slug>
    label: <label>
    # zdroj: <kde nalezeno>
    # ...

  # === utility (pravděpodobně zbytečné — projdi a vyber jen relevantní) ===

  - key: <slug>
    label: <label>
    # zdroj: <kde nalezeno>
    # ...
```

**Pravidla pro `key` a `label`:**
- `key` je slugifikovaný název (lowercase, pomlčky, bez diakritiky)
- `label` je čitelná podoba — pro známé technologie z kategorizace v kroku 4 použij kanonickou podobu (`Node.js`, `MongoDB`, `PostgreSQL`), pro ostatní původní název z manifestu
- Když si nejsi jistý label, dej tam slug a komentář `# TODO: doplň label`

**Pravidla pro pořadí:**
- V rámci každé kategorie řaď podle relevance: nejdřív frameworky a runtime, pak databáze, pak ostatní core nástroje
- `utility` na konci, abecedně (uživatel se k nim často nedostane)

### Krok 7 — souhrn pro uživatele

Po zápisu souboru ohlas:

```
Připravil jsem návrh slovníků: .claude/dictionary-suggestions.yaml

Obsahuje:
- 1 projekt: <název>
- <N> nástrojů: <X> core, <Y> runtime, <Z> utility

Další postup:
1. Otevři .claude/dictionary-suggestions.yaml
2. Projdi sekci 'core' a 'runtime' — to obvykle chceš celé do tools.yaml
3. Sekce 'utility' projdi a vyber jen položky, které ti budou ve slovníku k užitku
4. Zkopíruj 'projects' a vybrané 'tools' do docs-repo/meta/dictionaries/
5. Commit a reload serveru

Pak můžeš začít používat /make-doc.
```

## Pravidla

1. **Skill nezapisuje do `docs-repo`.** Jen do lokálního projektu, do `.claude/dictionary-suggestions.yaml`.

2. **Skill je idempotentní.** Opakované spuštění na již nakonfigurovaném projektu jen oznámí, že není co dělat.

3. **Skill nepotřebuje potvrzení uživatele před zápisem souboru.** Soubor je jen návrh, uživatel ho stejně musí ručně přenést. Souhlas se získává implicitně tím, že vůbec spustí `/prepare-doc`.

4. **Skill nehádá technologie z code analýzy.** Vychází z manifest souborů, které jsou strukturované a spolehlivé. Detekce z `import`/`require` v zdrojových souborech není v rozsahu.

5. **Při nejistotě defaultuje na `core`.** Lépe nabídnout něco zbytečného než vynechat něco důležitého — uživatel filtruje při reviewu, ne při sběru.

## Co skill úmyslně neřeší

- **Auto-kopírování do `docs-repo`** — `docs-repo` je samostatný repozitář, skill k němu nemá přístup. Manuální kopírování je správný handoff.
- **Aliasy** — skill je negeneruje. Pokud `package.json` obsahuje `mongodb` a chceš v slovníku `mongo` jako kanonický klíč s aliasem `mongodb`, vyřeš to při manuálním kopírování.
- **Aktualizaci `considered.yaml`** — to dělá server automaticky při zápisu decision (etapa 8a). `prepare-doc` se týká jen úvodního naplnění `tools.yaml` a `projects.yaml`.
- **Detekci tech stacku z konverzační historie napříč session** — skill čte jen aktuální session, ne projekty z minulosti.
