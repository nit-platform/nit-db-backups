# nit-db-backups / גיבויי מסד נתונים

**English:** Encrypted daily backups of the NIT Calendar Supabase database (`txdubkiyaceyassjcizc`).  
GitHub Actions dumps `public` + `auth` (+ `storage` metadata when readable), encrypts with [age](https://age-encryption.org/), and keeps the `.age` artifact for 14 days. Free plan only — no paid runners or paid storage beyond GitHub Free.

**עברית:** גיבוי יומי מוצפן של מסד הנתונים (Supabase) של nit-calendar.  
ה־workflow מריץ `pg_dump`, מצפין ב־`age`, ומעלה קובץ `.age` כ־artifact ל־14 יום. בלי הוצאות.

## Secrets / Variables (set by Nadav — bots never hold these)

| Name | Where | What |
|------|--------|------|
| `SUPABASE_DB_URL` | Actions **secret** | Session pooler URL, user `backup_ro.<project_ref>`, `sslmode=require` |
| `AGE_PUBLIC_KEY` | Actions **variable** | Public key only (`age1…`). Private key stays offline. |

## Schedule

- Cron: `0 0 * * *` (00:00 UTC = 02:00 Israel winter / 03:00 Israel summer)
- Also: **Actions → Backup → Run workflow**

## Keepalive

`keepalive.yml` runs monthly with a no-op checkout so GitHub does not disable scheduled workflows after ~60 days of inactivity. It needs `contents: write` only for an empty commit if the repo stays quiet; see that file.

## Decrypt (local)

```bash
age -d -i nit-backup-key.txt backup.dump.age > backup.dump
```

## Restore (local / emergency)

Use `pg_restore` against a Postgres **17** instance. Do not restore over production without a deliberate plan.

## Security notes

- Role `backup_ro` is read-only (+ `BYPASSRLS` for dumps). No write grants.
- Never commit the age private key, DB password, or connection string.
- Do not echo secrets in logs.

## Auth schema caveat (hosted Supabase)

On hosted Supabase, `GRANT USAGE ON SCHEMA auth TO backup_ro` is a no-op when run as `postgres`
(no grant option; owner is `supabase_admin`). See
[Supabase KB](https://supabase.com/kb/troubleshooting/permission-errors-when-granting-custom-roles-access-to-the-auth-schema).

The workflow tries `public,auth,storage` then narrower sets. With `backup_ro`, expect **public**
(+ **storage** if granted) unless you point `SUPABASE_DB_URL` at a role that already has `auth`
USAGE (e.g. `postgres`) — only if Nadav chooses that trade-off; bots never store that password.
