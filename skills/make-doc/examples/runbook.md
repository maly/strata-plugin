# Příklad: `runbook`

**Kdy použít:** Pro postup řešení konkrétního problému nebo incidentu — co se
stalo, jak to detekovat, jak to opravit. Runbook má vždy `trigger` (co spouští
jeho použití) a typicky předpokládá, že čtenář je pod tlakem ("něco hoří, co dělat").

Rozdíl oproti `howto`: howto je rutinní postup ("jak nainstalovat X"), runbook je
reakce na problém ("co dělat když X nefunguje").

## Vstup pro `doc_write`

```json
{
  "title": "Obnova MongoDB po výpadku disku",
  "author": "claude-code",
  "l1_draft": "Postup obnovy MongoDB instance na hetzner-1 po výpadku datového disku včetně obnovy z backupu.",
  "l2_draft": "Trigger: MongoDB nestartuje, v logu chyby na /opt/mongo/data, nebo disk plný/nedostupný. Severity: high. Hlavní kroky: zastavit kontejner, ověřit stav disku, obnovit z posledního backupu, znovu spustit, ověřit replikaci a aplikaci. Časový odhad: 15–30 minut včetně obnovy z S3.",
  "body": "# Obnova MongoDB po výpadku disku\n\n## Trigger\n\nNěkterý z těchto signálů:\n- Aplikace hlásí chybu připojení k MongoDB\n- `docker ps` ukazuje kontejner `mongo` jako restarting nebo exited\n- V logu kontejneru chyby typu `Failed to open file`, `IOError`\n- Monitoring hlásí disk usage > 95% na /opt/mongo/data\n\n## Předpoklady\n\n- SSH přístup na hetzner-1\n- Přístup k S3 backup bucketu (AWS credentials v `/root/.aws/credentials`)\n- 1Password přístup (heslo admin uživatele)\n\n## Postup\n\n### 1. Zastavit kontejner\n\n```bash\nssh hetzner-1\ncd /opt/mongo\ndocker compose down\n```\n\n### 2. Ověřit stav disku\n\n```bash\ndf -h /opt/mongo/data\nls -la /opt/mongo/data\ndmesg | tail -50\n```\n\nPokud je disk plný — uvolnit místo (smazat staré backupy z `/backup/mongo/`).\nPokud je disk nedostupný — kontaktovat Hetzner support, runbook tu končí.\n\n### 3. Obnova z backupu\n\nNajít poslední úspěšný backup:\n\n```bash\nls -lt /backup/mongo/ | head -5\n```\n\nPokud lokální backup chybí, stáhnout z S3:\n\n```bash\naws s3 ls s3://backup-content-api/mongo/ | tail -3\naws s3 sync s3://backup-content-api/mongo/2026-05-07/ /tmp/restore/\n```\n\nObnovit data:\n\n```bash\nrm -rf /opt/mongo/data/*\ndocker run --rm -v /opt/mongo/data:/data/db -v /tmp/restore:/restore mongo:7 \\\n  mongorestore --dir=/restore /data/db\nchown -R 999:999 /opt/mongo/data\n```\n\n### 4. Spustit kontejner\n\n```bash\ncd /opt/mongo && docker compose up -d\ndocker logs -f mongo-mongo-1\n```\n\nPočkat na log `Waiting for connections`.\n\n### 5. Ověřit replica set\n\n```bash\ndocker exec -it mongo-mongo-1 mongosh -u admin -p \\\n  --eval 'rs.status()'\n```\n\nStav by měl být `PRIMARY`. Pokud `SECONDARY` nebo `OTHER`, počkat 30s a zkusit znovu.\n\n### 6. Ověřit aplikaci\n\n```bash\ncurl https://api.example.com/health\n```\n\nMá vrátit 200 a v body `\"db\": \"ok\"`.\n\n## Když nic nefunguje\n\nKontaktovat Martina (telefon v 1Password). Mezitím přepnout aplikaci do read-only\nmódu nastavením env `READ_ONLY=true` v API service.\n\n## Po obnově\n\n- Sepsat `source` dokument s časovou osou incidentu\n- Pokud vznikl nový poznatek, zvážit update tohoto runbooku přes `/make-doc`",
  "tools": ["mongo", "docker"],
  "projects": ["content-api"],
  "links": [
    { "to": "config-mongo-hetzner1-7b2a", "rel": "references" }
  ],
  "type_specific": {
    "status": "current",
    "trigger": "MongoDB nestartuje, chyby na datovém disku, nebo disk usage > 95%",
    "severity": "high"
  }
}
```

## Pravidla pro runbook

- `l1_draft` — jedna věta popisující situaci a cíl (obnova/oprava čeho)
- `l2_draft` — trigger, severity, hlavní kroky, časový odhad
- `body` — detailní postup s code blocky, ověřením, eskalací
- `type_specific.trigger` — **povinné**, popis spouštěče (co se musí stát, aby se sáhlo po runbooku)
- `type_specific.severity` — `low` | `medium` | `high` | `critical`, volitelné
- `type_specific.status` — `current` | `outdated`, default `current`
- `links` — `references` na související config (kde najít cesty, porty, hesla)
- `tools` — technologie zapojené v incidentu

## Co runbook obsahuje

Vždy popsat: trigger, předpoklady, postup, ověření, eskalaci ("co když nic nefunguje").
Cíl je, aby člověk pod tlakem nemusel přemýšlet — jen postupoval.
