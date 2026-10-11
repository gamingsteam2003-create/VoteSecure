# VoteSecure

Secure Online College Election Management System. Built and developed by Hriata_khuptong.

Stack: React 19 (CRA + Tailwind + shadcn/ui + TanStack Query + Recharts) · FastAPI · MongoDB (Motor). API base: `/api/v1`. OpenAPI docs: `/api/v1/docs` (disabled when `APP_ENV=production`).

Academic documentation: [docs/PROJECT_REPORT.md](docs/PROJECT_REPORT.md)

## Configuration
Copy `backend/.env.example` to `backend/.env` and fill in real values. Never commit real secrets.
- `INITIAL_ADMIN_PASSWORD`: deployment secret. On first start it creates `INITIAL_ADMIN_EMAIL` as an admin, and the password must be changed at first login. Re-runs never reset an existing account. If the secret is missing, no admin is created and an error is logged.
- `AUDIT_CHECKPOINT_KEY`: a random secret of 32+ characters used to sign audit checkpoints. `AUDIT_CHECKPOINT_DIR` should sit on storage separate from the database.
- In production (`APP_ENV=production`) the server refuses to start without `COOKIE_SECURE=true` and a strong checkpoint key.
- Frontend: `frontend/.env` → `REACT_APP_BACKEND_URL=https://<your-domain>`.

## Run
```
cd backend && pip install -r requirements.txt && uvicorn server:app --host 0.0.0.0 --port 8001
cd frontend && yarn install && yarn build   # serve build/ behind HTTPS reverse proxy; route /api to backend
```
Indexes are created automatically on startup, and they act as the migrations: a schema change means adding an index or field in `core.ensure_indexes`, which is idempotent. No elections, students or candidates are created automatically. Elections never open automatically.

## Tests
```
cd backend && python -m pytest tests/test_votesecure.py -q
```
The tests use the separate `votesecure_test` database with synthetic data and drop it afterwards. Never point tests at production.

## Deployment checklist
1. Use a fresh production database (no test records). A MongoDB replica set or Atlas is recommended so votes are recorded in multi-document transactions.
2. Set the secrets listed above in the platform secret store. Set `APP_ENV=production`, `COOKIE_SECURE=true` and exact `CORS_ORIGINS`.
3. Use HTTPS only. Add a frontend CSP at the reverse proxy (`default-src 'self'; connect-src 'self'; font-src fonts.gstatic.com; style-src 'self' 'unsafe-inline' fonts.googleapis.com; frame-ancestors 'none'`).
4. Sign in as the initial admin, change the password and enable MFA. Create a second admin if you use two-person approval.
5. Import the authorised registry, then verify students.
6. Schedule `scripts/backup.sh` (for example, nightly) and set `BACKUP_STATUS_FILE`. Test a restore into a scratch database.
7. Check dependencies: `pip-audit -r backend/requirements.txt` and `yarn audit` (frontend).

## Backup, restore, rollback
- Backup: `MONGO_URL=... DB_NAME=... BACKUP_DIR=... BACKUP_STATUS_FILE=... scripts/backup.sh`
- Restore: `mongorestore --uri=$MONGO_URL --archive=<file> --gzip`, then run Audit log → Verify integrity.
- Code rollback: redeploy the previous build. Data is backward compatible because there are only additive fields and indexes.
- Correcting a finalised election: no in-app path exists by design. Document the incident, record an auditor finding, and create a new election if a re-run is authorised.

## Platform branding
The app UI contains no third-party promotional branding. The hosting platform may inject its own preview badge. Whether it can be removed depends on the deployment plan or custom-domain setup.
