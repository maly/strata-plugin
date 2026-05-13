# Příklad: `howto`

**Kdy použít:** Když existuje sekvence kroků, kterou má smysl zopakovat — instalace,
nastavení, postup pro běžný úkol. Howto je "jak", ne "co a proč" (to je `decision`).

Pokud postup řeší krizový stav (něco se rozbilo, je třeba obnovit), použij místo
toho `runbook`.

## Vstup pro `doc_write`

```json
{
  "title": "Nasazení MongoDB přes Docker na hetzner-1",
  "author": "claude-code",
  "l1_draft": "Postup nasazení MongoDB s replica setem rs0 přes Docker Compose na hetzner-1.",
  "l2_draft": "Cíl: rozjet MongoDB v kontejneru s persistencí na host volume a autentizací. Vyžaduje Docker, Docker Compose a SSH přístup na hetzner-1. Hlavní kroky: příprava volume, vytvoření docker-compose.yml, inicializace replica setu, nastavení uživatele s heslem. Výsledek: běžící Mongo dostupné na portu 27017 z localhostu hostitele.",
  "body": "# Nasazení MongoDB na hetzner-1\n\n## Předpoklady\n\n- SSH přístup na hetzner-1\n- Docker a Docker Compose nainstalované\n- Volné 5 GB na disku\n\n## Kroky\n\n### 1. Příprava volume\n\n```bash\nssh hetzner-1\nmkdir -p /opt/mongo/data /opt/mongo/config\nchown -R 999:999 /opt/mongo\n```\n\n### 2. Docker Compose\n\n`/opt/mongo/docker-compose.yml`:\n\n```yaml\nservices:\n  mongo:\n    image: mongo:7\n    restart: unless-stopped\n    ports:\n      - \"127.0.0.1:27017:27017\"\n    volumes:\n      - /opt/mongo/data:/data/db\n      - /opt/mongo/config:/data/configdb\n    command: [\"--replSet\", \"rs0\", \"--bind_ip_all\"]\n```\n\n### 3. Spuštění\n\n```bash\ncd /opt/mongo && docker compose up -d\n```\n\n### 4. Inicializace replica setu\n\n```bash\ndocker exec -it mongo-mongo-1 mongosh --eval 'rs.initiate()'\n```\n\n### 5. Vytvoření uživatele\n\n```bash\ndocker exec -it mongo-mongo-1 mongosh --eval '\n  db.getSiblingDB(\"admin\").createUser({\n    user: \"app\",\n    pwd: \"...\",\n    roles: [{role: \"readWrite\", db: \"content\"}]\n  })\n'\n```\n\n## Ověření\n\n```bash\ncurl localhost:27017\n# má vrátit: It looks like you are trying to access MongoDB...\n```\n\n## Související\n\n- Konfigurace: viz `config-mongo-hetzner1`\n- Rozhodnutí: viz `decision-volba-mongodb-content-api`",
  "tools": ["mongo", "docker"],
  "projects": ["content-api"],
  "links": [
    { "to": "decision-volba-mongodb-content-api-3f8c", "rel": "implements" }
  ],
  "type_specific": {
    "status": "current",
    "audience": "vývojáři"
  }
}
```

## Pravidla pro howto

- `l1_draft` — jedna věta popisující cíl postupu
- `l2_draft` — cíl, předpoklady, hlavní kroky, výsledek (server přepíše)
- `body` — plný postup s code blocky a komentáři
- `tools` — všechny technologie použité v postupu
- `links` — typicky `implements` na decision, pokud postup implementuje konkrétní rozhodnutí
- `type_specific.status` — `current` | `outdated`, default `current`
- `type_specific.audience` — volitelné, kdo je cílový čtenář
