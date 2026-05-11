# Supabase Backup to Cloudflare R2

Free automated PostgreSQL database backups from Supabase to Cloudflare R2 storage, powered by GitHub Actions. Zero infrastructure, zero cost. Set up in 5 minutes.

## Why This Exists

Supabase's free tier doesn't include automated backups. If your database gets corrupted or you accidentally drop a table, you're out of luck. This workflow dumps your entire database (roles, schema, and data) to Cloudflare R2 every day, giving you point-in-time recovery without paying for Supabase's backup add-on.

Cloudflare R2 has no egress fees, and GitHub Actions gives you 2,000 free minutes per month on private repos (unlimited on public). The math works out to completely free daily backups for most projects.

## Features

- Dumps roles, schema, and data as separate SQL files
- Compresses everything into a dated zip archive
- Uploads directly to Cloudflare R2 with the AWS S3 API
- Optional 14-day auto-delete lifecycle policy on R2 (keeps storage cheap)
- Manual trigger or cron schedule
- Runs on GitHub's free tier runners (no self-hosting needed)

## How It Works

```
Supabase DB ──▶ GitHub Actions runner ──▶ Cloudflare R2 bucket
                  (pg_dump via            (backup-YYYY-MM-DD.zip)
                   Supabase CLI)
```

Every run creates a zip file containing three SQL dumps:

- `roles.sql` — Database roles and permissions
- `schema.sql` — Full table structure, indexes, and functions
- `data.sql` — All row data (uses COPY for speed)

Files are named `backup-YYYY-MM-DD.zip` and uploaded to your R2 bucket.

## Quick Start

### 1. Fork or clone this repo

```bash
git clone https://github.com/cucoleadan/supabase-to-r2-backup.git
```

### 2. Create a Cloudflare R2 bucket

Go to [Cloudflare Dashboard](https://dash.cloudflare.com) → R2 → Create bucket. Name it something like `supabase-backups`.

Optional but recommended: set a lifecycle rule to auto-delete backups older than 14 days. Go to bucket Settings → Object lifecycle → Create rule with action **Delete**, prefix `backup-`, and 14 days.

### 3. Create an R2 API token

In the R2 dashboard, go to Manage R2 API Tokens → Create API token. Set permissions to **Object Read & Write**, scope it to your backup bucket. Save the Access Key ID and Secret Access Key immediately.

### 4. Get your Supabase connection string

In your Supabase project: Settings → Database → Connection info. Select **Session pooler** (port 5432) and copy the full PostgreSQL URI. Do not use the direct connection, it resolves to IPv6 and breaks on GitHub Actions runners.

### 5. Add repository secrets

Go to your repo's Settings → Secrets and variables → Actions. Add these five secrets:

| Secret | Value |
|--------|-------|
| `SUPABASE_DB_URL` | Session pooler connection string (e.g. `postgresql://postgres.xxx:password@aws-0-eu-central-1.pooler.supabase.com:5432/postgres`) |
| `CF_ACCESS_KEY_ID` | Cloudflare R2 Access Key ID |
| `CF_SECRET_ACCESS_KEY` | Cloudflare R2 Secret Access Key |
| `CF_BUCKET_NAME` | Your R2 bucket name |
| `CF_BUCKET_ENDPOINT` | R2 S3 endpoint (e.g. `https://<account-id>.r2.cloudflarestorage.com`) |

### 6. Enable the schedule

Uncomment the `schedule` block in `.github/workflows/backup.yml`:

```yaml
on:
  workflow_dispatch:
  schedule:
    - cron: '0 2 * * *'  # Runs daily at 2 AM UTC
```

The workflow also supports manual runs from the Actions tab for testing.

## Restoring a Backup

Download the zip from your R2 bucket, extract it, and restore:

```bash
# Restore roles first
psql "$SUPABASE_DB_URL" -f roles.sql

# Then schema
psql "$SUPABASE_DB_URL" -f schema.sql

# Then data
psql "$SUPABASE_DB_URL" -f data.sql
```

Or use the Supabase CLI:

```bash
supabase db restore --db-url "$SUPABASE_DB_URL" backup-2026-05-11.zip
```

## Troubleshooting

### Network unreachable / IPv6 errors
Your connection string is using the direct connection. Switch to the session pooler URL (port 5432) in Supabase settings.

### Wrong password errors
Reset your database password in Supabase Settings → Database, then update the `SUPABASE_DB_URL` secret with the new connection string.

### Socket / "no such file" errors
The workflow secrets are missing or empty. Verify all five secrets are set in your repository settings.

### Supabase CLI failures
The workflow uses the official Supabase CLI for dumps. If you get CLI errors, make sure your `SUPABASE_DB_URL` uses the session pooler and includes the full connection string with password.

## Cost

- GitHub Actions: 2,000 free minutes/month (private repos) or unlimited (public). Daily backups use ~2 minutes each, well under the limit.
- Cloudflare R2: 10 GB free storage. A typical Supabase backup is 1-50 MB. With a 14-day lifecycle, you'll store 14-28 files max. Egress is free.
- Total: $0 for most projects.

## License

MIT
