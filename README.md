# dbhaven

[![CI](https://github.com/miriyaladhanwinn/dbhaven/actions/workflows/ci.yml/badge.svg)](https://github.com/miriyaladhanwinn/dbhaven/actions/workflows/ci.yml)
[![Go](https://img.shields.io/badge/go-1.24%2B-blue)](https://go.dev)
[![License](https://img.shields.io/badge/license-Apache--2.0-green)](LICENSE)

**dbhaven** is a small, sharp CLI for scheduled, encrypted database backups.
Define backup jobs in YAML, run them on cron schedules, store to local disk
or any S3-compatible storage, encrypt everything with AES-256-GCM, and prune
with grandfather-father-son retention.

Supported databases: **PostgreSQL** (`pg_dump`/`pg_restore`), **MySQL/MariaDB**
(`mysqldump`/`mysql`), **MongoDB** (`mongodump`/`mongorestore`).

## Features

- 📸 Logical backups via the native client tools you already trust
- 🗓️ Cron scheduling daemon (`schedule`) or one-shot runs (`backup`)
- 🔐 AES-256-GCM encryption, streaming for multi-GB dumps (chunked, authenticated)
- 💾 Local filesystem + S3-compatible backends (AWS S3, MinIO, R2, …)
- 🗑️ GFS retention (daily/weekly/monthly) or simple keep-last-N
- ♻️ One-command `restore` with decryption + decompression handled
- 🧾 Structured `slog` logging, `--dry-run` to preview any job

## Install

Requires Go 1.24+ and the client tools for the databases you back up
(`pg_dump`, `mysqldump`, `mongodump`).

```bash
git clone https://github.com/miriyaladhanwinn/dbhaven
cd dbhaven
go build -o dbhaven ./cmd/dbhaven
sudo mv dbhaven /usr/local/bin/
```

## Quickstart

```bash
# 1. Generate an encryption key (0600, hex-encoded)
dbhaven keygen --out ~/.dbhaven-key

# 2. Write your config
cp dbhaven.example.yaml dbhaven.yaml
$EDITOR dbhaven.yaml   # set key_file, DSNs, destinations

# 3. Preview what a job would do
dbhaven backup --config dbhaven.yaml --job main-postgres --dry-run

# 4. Run it once
dbhaven backup --config dbhaven.yaml --job main-postgres

# 5. Run the scheduler daemon (cron jobs from the config)
dbhaven schedule --config dbhaven.yaml

# 6. List backups
dbhaven list --config dbhaven.yaml --job main-postgres

# 7. Restore (requires --yes; target defaults to the job's DSN)
dbhaven restore --config dbhaven.yaml --job main-postgres \
  --file main-postgres-postgres-20261017-020000.dump.enc \
  --target "postgres://u:p@localhost:5432/restoredb" --yes
```

## Config reference

```yaml
encryption:
  key_file: /etc/dbhaven/key   # or: key_env: DBHAVEN_KEY (exactly one)

jobs:
  - name: main-postgres
    type: postgres              # postgres | mysql | mariadb | mongo
    dsn: "postgres://user:pass@host:5432/db?sslmode=require"
    schedule: "0 2 * * *"       # standard 5-field cron; optional
    destinations:
      - type: local
        path: /var/backups/dbhaven
      - type: s3
        endpoint: s3.amazonaws.com
        bucket: my-bucket
        prefix: prod/           # optional
        region: us-east-1       # optional
        access_key: ...
        secret_key: ...
        use_ssl: true           # default true
    retention:
      daily: 7                  # GFS tiers…
      weekly: 4
      monthly: 6
      # …or: keep_last: 10      # simple mode (takes precedence)
    extra_args: []              # passed to the dump tool verbatim
```

Notes:

- MySQL DSNs look like `mysql://user:pass@host:3306/db`; they are translated
  to `--host/--port/--user/--password` flags (never exposed on the command
  line beyond what `mysqldump` itself needs).
- Backups are named `<job>-<type>-YYYYMMDD-HHMMSS.<ext>[.enc]`, e.g.
  `main-postgres-postgres-20261017-020000.dump.enc`.
- Without any `encryption` key configured, backups are stored in plaintext
  and dbhaven warns you loudly.
- `restore` refuses to run without `--yes`.

## How it works

```
┌─────────┐   ┌─────────┐   ┌──────────┐   ┌──────────────┐   ┌───────────┐
│ pg_dump │──▶│ gzip*   │──▶│ AES-256  │──▶│ local disk / │──▶│ GFS prune │
│mysqldump│   │(*except │   │ GCM      │   │ S3-compatible│   │ old files │
│mongodump│   │ pg -Fc) │   │ (chunked)│   │ storage      │   │           │
└─────────┘   └─────────┘   └──────────┘   └──────────────┘   └───────────┘
```

## Roadmap

- [ ] Point-in-time / WAL archiving helpers for Postgres
- [ ] Backup verification (`dbhaven verify`: decrypt + test-restore to scratch DB)
- [ ] Prometheus metrics endpoint for the scheduler daemon
- [ ] `rclone`-style additional backends (SFTP, GDrive)
- [ ] Pre/post backup hooks (e.g. notify webhooks)

## License

Apache-2.0 — see [LICENSE](LICENSE).
