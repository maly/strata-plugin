# Příklad: `config`

**Kdy použít:** Pro zachycení skutečného stavu konfigurace běžící služby —
verze, parametry, env proměnné, síťové porty, cesty k datům. Config je "jak je to
teď nasazené", ne "jak to nainstalovat" (to je `howto`).

Config je často párový dokument k `howto`: howto popisuje postup, config zachycuje
výsledný stav.

## Vstup pro `doc_write`

```json
{
  "title": "MongoDB rs0 na hetzner-1",
  "author": "claude-code",
  "l1_draft": "Konfigurace MongoDB replica setu rs0 na hetzner-1 s autentizací a persistencí na /opt/mongo/data.",
  "l2_draft": "Běžící instance MongoDB 7.x v Dockeru, replica set rs0 s jedním nodem (zatím bez replikace). Persistence na host volume /opt/mongo/data, port 27017 vázaný na localhost hostitele. Autentizace zapnutá, uživatel app s rolí readWrite na databázi content. Restart policy unless-stopped.",
  "body": "# Konfigurace MongoDB na hetzner-1\n\n## Stav\n\n- Server: hetzner-1\n- Služba: mongodb\n- Verze: mongo:7\n- Stav: produkce, běží od 2026-04-22\n\n## Síťové parametry\n\n- Port: 27017\n- Bind: 127.0.0.1 (jen localhost hostitele)\n- Replica set: rs0 (single node)\n\n## Persistence\n\n- Data: /opt/mongo/data → /data/db v kontejneru\n- Config: /opt/mongo/config → /data/configdb v kontejneru\n- Vlastník: 999:999 (mongodb user v image)\n\n## Autentizace\n\n- Admin uživatel: `admin` (heslo v 1Password vault)\n- Aplikační uživatel: `app` s rolí `readWrite` na DB `content`\n\n## Docker Compose\n\n`/opt/mongo/docker-compose.yml`:\n\n```yaml\nservices:\n  mongo:\n    image: mongo:7\n    restart: unless-stopped\n    ports:\n      - \"127.0.0.1:27017:27017\"\n    volumes:\n      - /opt/mongo/data:/data/db\n      - /opt/mongo/config:/data/configdb\n    command: [\"--replSet\", \"rs0\", \"--bind_ip_all\", \"--auth\"]\n```\n\n## Připojení z aplikace\n\n```\nmongodb://app:<heslo>@127.0.0.1:27017/content?replicaSet=rs0\n```\n\n## Backup\n\n- Cron: denně v 03:00 přes mongodump\n- Cíl: /backup/mongo/{date}/\n- Retence: 14 dní lokálně, 90 dní na S3",
  "tools": ["mongo", "docker"],
  "projects": ["content-api"],
  "links": [
    { "to": "howto-nasazeni-mongo-hetzner1-2b1d", "rel": "references" },
    { "to": "decision-volba-mongodb-content-api-3f8c", "rel": "implements" }
  ],
  "type_specific": {
    "status": "current",
    "host": "hetzner-1",
    "service": "mongodb"
  }
}
```

## Pravidla pro config

- `l1_draft` — jedna věta s tím, co a kde běží
- `l2_draft` — strukturovaný výtah aktuálního stavu
- `body` — kompletní stav konfigurace včetně cest, portů, env, integrace
- `tools` — všechny komponenty (např. `mongo` a `docker` pokud běží přes Docker)
- `links` — typicky `implements` na decision a `references` na howto
- `type_specific.status` — `current` | `outdated`, default `current`
- `type_specific.host` — kde běží (jméno serveru, "localhost", "kubernetes/cluster-x")
- `type_specific.service` — název služby pro orientaci

## Když se config změní

Při změně konfigurace má smysl použít `doc_update` na existující dokument, ne
vytvářet nový. Config dokument zachycuje aktuální stav, historie je v git logu.

Výjimka: pokud došlo k zásadní změně architektury (přesun na jiný server,
změna technologie), zapiš nový config a starý označ jako `outdated`.
