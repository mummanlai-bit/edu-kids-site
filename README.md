# Edu Kids — Хүүхэд хөгжлийн төв

Portable production website for Edu Kids in Ulaanbaatar. The repository contains a React/Vite frontend, a standard Express API, and a PostgreSQL database layer. It can run on Replit or on another Node.js host without rebuilding the product from scratch.

## Project architecture

```text
artifacts/panda-daycare/   React + Vite public website and /admin editor
artifacts/api-server/      Express API for content, admin login, and health checks
lib/api-spec/              OpenAPI source of truth
lib/api-client-react/      Generated React Query API client
lib/api-zod/               Generated server/client validation schemas
lib/db/                    Drizzle PostgreSQL schema and database client
database/migrations/       Portable SQL migrations
database/*.sh              PostgreSQL migration and backup helpers
attached_assets/           Source assets supplied for the website
```

The public site reads editable text from `GET /api/content`. The protected `/admin` screen updates the same content through `PUT /api/admin/content`.

## Technologies

- Node.js 20+ (Node.js 24 is used in the current Replit workspace)
- pnpm 10+
- React, Vite, TypeScript, Tailwind CSS
- Express 5
- PostgreSQL with `pg` and Drizzle ORM
- OpenAPI + Orval-generated TypeScript clients
- HMAC-signed, HTTP-only cookie sessions for the single admin password

## Local setup

1. Install Node.js and pnpm.
2. Install dependencies:

   ```sh
   pnpm install --frozen-lockfile
   ```

3. Copy `.env.example` to `.env` and set `DATABASE_URL`, `ADMIN_PASSWORD`, and `SESSION_SECRET`.
4. Create a PostgreSQL database and run the schema migration:

   ```sh
   sh database/migrate.sh
   ```

5. Start the API:

   ```sh
   NODE_ENV=development PORT=5000 pnpm --filter @workspace/api-server run build
   NODE_ENV=development PORT=5000 pnpm --filter @workspace/api-server run start
   ```

6. In a second terminal, start the frontend:

   ```sh
   BASE_PATH=/ PORT=5173 pnpm --filter @workspace/panda-daycare run dev
   ```

   When both services run on separate local ports, set `VITE_API_BASE_URL=http://localhost:5000` before starting Vite. For a same-origin reverse proxy, omit it and proxy `/api` to the API server.

The first request to `GET /api/content` creates the initial content row from the server-side defaults if the table is empty.

## Production build and deployment

### API server

```sh
pnpm install --frozen-lockfile
sh database/migrate.sh
pnpm --filter @workspace/api-server run build
NODE_ENV=production PORT=5000 pnpm --filter @workspace/api-server run start
```

Required API environment variables:

- `NODE_ENV=production`
- `PORT`
- `DATABASE_URL`
- `ADMIN_PASSWORD`
- `SESSION_SECRET`
- `CORS_ORIGIN` when the frontend is hosted on a different origin
- `COOKIE_SAME_SITE=none` only when the frontend and API are on different sites and both use HTTPS; otherwise keep `lax`

The API health endpoint is `GET /api/healthz`.

### Frontend

Build the static frontend:

```sh
BASE_PATH=/ VITE_API_BASE_URL=https://api.example.com \
  pnpm --filter @workspace/panda-daycare run build
```

Deploy `artifacts/panda-daycare/dist/public` to any static host such as Nginx, Cloudflare Pages, Netlify, or an object-storage website host. Configure the static host to rewrite unknown routes to `index.html` so `/admin` can load directly.

For a single-domain deployment, reverse-proxy `/api` to the API server and omit `VITE_API_BASE_URL`. For separate frontend and API origins, set `VITE_API_BASE_URL` to the API origin and set `CORS_ORIGIN` to the exact frontend origin. If they are on different sites, set `COOKIE_SAME_SITE=none`; HTTPS is required for production admin cookies.

## Database backup and restore

Back up the PostgreSQL database before moving providers:

```sh
sh database/backup.sh backups/edu-kids.dump
```

Create the target database, set its `DATABASE_URL`, then restore:

```sh
sh database/restore.sh backups/edu-kids.dump
```

The database is standard PostgreSQL. The only application table is `site_content`, which stores the editable public content as JSONB. Do not put `DATABASE_URL`, passwords, or session secrets in the repository.

## Environment variables

See `.env.example` for the complete list. `ADMIN_PASSWORD` and `SESSION_SECRET` must be configured as secrets on whichever host runs the API. `VITE_API_BASE_URL` is a frontend build-time variable, not a secret.

## External services

- PostgreSQL: required for persisted admin-editable content.
- Google Fonts CDN: optional visual dependency referenced by the HTML; the site still runs if the font request is unavailable.
- Facebook: external link only (`https://www.facebook.com/edukidsmn`); no API integration.

There are no payment, analytics, storage, AI, or Replit API integrations in the product runtime.

## Replit-specific items

The `.replit` file, artifact metadata under `.replit-artifact/`, workflows, and post-merge helper are Replit development/deployment conveniences only. The product runtime does not require a Replit SDK, Replit database, Replit authentication, or Replit storage.

The application source uses standard Node.js, Express, PostgreSQL, React, and Vite APIs. The Vite app no longer requires Replit development plugins.

## API contract changes

When changing API routes or payloads:

```sh
pnpm --filter @workspace/api-spec run codegen
pnpm run typecheck
```

## Validation

```sh
pnpm run typecheck
pnpm --filter @workspace/api-server run build
PORT=5173 BASE_PATH=/ pnpm --filter @workspace/panda-daycare run build
```

## MIGRATION CHECKLIST

- [ ] Download or clone the full repository, including `pnpm-lock.yaml`, `attached_assets/`, database migrations, and generated API files.
- [ ] Provision a standard PostgreSQL database on the new provider.
- [ ] Set `DATABASE_URL`, `ADMIN_PASSWORD`, and a newly generated `SESSION_SECRET`.
- [ ] Set `CORS_ORIGIN` and, only for cross-site frontend/API hosting, `COOKIE_SAME_SITE=none`.
- [ ] Run `sh database/migrate.sh`, or restore the previous database backup with `sh database/restore.sh`.
- [ ] Deploy the API build and expose `/api/*` over HTTPS.
- [ ] Build and deploy `artifacts/panda-daycare/dist/public` as a static site.
- [ ] Either reverse-proxy `/api` on the same domain or set `VITE_API_BASE_URL` and `CORS_ORIGIN` for separate domains.
- [ ] Configure SPA fallback to `index.html`.
- [ ] Confirm `/`, `/admin`, `/api/healthz`, admin login, content save, and a page reload.
- [ ] Point the domain DNS records to the new frontend/API host.
- [ ] Keep Replit only as a development workspace, or download the project and database backup before cancelling the Replit plan.

## Remaining manual steps

- Choose and provision the new PostgreSQL and hosting providers.
- Configure production environment variables as secrets.
- Configure DNS and HTTPS.
- Perform the final admin login/save test after the new host is live.