# Branch Technical Implementation

## Branch status

| Field | Value |
| --- | --- |
| Branch | `saran2991-branch-technical-document` |
| Base | `origin/main` |
| Purpose | Maintain a working technical record for this branch |
| Initial implementation delta | None; the branch matched `origin/main` before this document was added |
| Last reviewed | 2026-09-05 |

This is a living document. Update the branch-local changes section whenever the
implementation, data model, build process, or verification scope changes.

## Branch-local changes

The only branch-local change currently is this technical document. The
application implementation described below is inherited from `origin/main`.

## Application architecture

Tailspin Toys is a single Astro 7 application that produces fully prerendered
static output. Astro pages read from a local SQLite database during development
and builds; there is no runtime API server or client-side application framework.

The primary implementation layers are:

| Layer | Technology | Responsibility |
| --- | --- | --- |
| Presentation | Astro components and pages | Render routes and reusable UI as static HTML |
| Styling | Tailwind CSS v4 | Provide the dark theme, responsive layout, and component styling |
| Data access | Drizzle ORM with `sqlite-proxy` | Execute typed queries through an injectable database client |
| Storage | Node.js `DatabaseSync` | Read and write the local `tailspin.db` SQLite file |
| Source data | `db/games.csv` | Supply deterministic seed data for local development and builds |
| Unit testing | Vitest | Test transforms, rating helpers, and data-access behavior |
| End-to-end testing | Playwright | Test the generated site and accessibility in Chromium |

## Build-time data flow

1. `predev` or `prebuild` runs `npm run db:setup`.
2. `db/migrate.ts` applies the generated Drizzle migrations.
3. `db/seed.ts` parses `db/games.csv` and inserts missing categories,
   publishers, and games.
4. Astro evaluates page frontmatter during the build.
5. Pages obtain the shared database client from `src/lib/db.ts`.
6. Data-access helpers in `src/lib/games.ts` query and map database rows into
   application types.
7. Astro renders the results into static HTML in `dist/`.

The default database URL is `file:tailspin.db`. `DATABASE_URL` may override it
with another local `file:` URL or `:memory:`. Other URL schemes are rejected.

## Data implementation

### Schema

`db/schema.ts` defines three tables:

- `categories`: unique name and optional description.
- `publishers`: unique name and optional description.
- `games`: title, description, nullable star rating, and required foreign keys
  to a category and publisher.

Schema changes require a generated migration through `npm run db:generate`.

### Database adapter

`src/lib/db.ts` bridges Drizzle's asynchronous SQLite proxy interface to
Node.js's synchronous SQLite driver. It:

- resolves and validates local database paths;
- creates parent directories when needed;
- maps Drizzle `run`, `all`, `values`, and `get` operations to prepared SQLite
  statements;
- executes migration query batches in an explicit transaction; and
- caches one database client for build-time page queries.

Data-access helpers accept a `Database` argument rather than importing the
singleton directly. This keeps query behavior testable with isolated databases.

### Seed behavior

`db/transforms.ts` contains pure CSV and seed transforms. Ratings are derived
from a stable title hash, producing a reproducible value from 3.0 through 5.0.

`db/seed.ts` is idempotent for existing records:

- categories and publishers are matched by unique name;
- games are matched by title; and
- existing game rows are skipped rather than updated.

As a result, changing or removing a CSV row does not reconcile an existing local
database. Delete the generated database and rerun setup when an exact local
rebuild from the CSV is required.

## Route implementation

### Home page

`src/pages/index.astro` queries all games ordered by title. It renders either a
responsive `GameCard` grid or an `EmptyState` when no records exist.

### Game details

`src/pages/game/[id].astro` uses `getStaticPaths()` to create one static route
per game ID. Each route queries the full game record, including category and
publisher details. A missing record redirects to the branded `/404` route.

### Shared presentation

`src/layouts/Layout.astro` owns the document shell, global stylesheet import,
font setup, navigation header, and main content container. Reusable components
under `src/components/` provide cards, buttons, tags, ratings, page headings,
empty states, and navigation.

The site uses semantic HTML, visible keyboard focus treatments, and
`data-testid` attributes for interactive behavior covered by Playwright.

## Static output constraints

- Database access is available only while developing or building the site.
- New database records require a new build before they appear in deployed HTML.
- Dynamic game routes must be known to `getStaticPaths()` at build time.
- Runtime mutations, user accounts, pledges, and live crowdfunding updates
  would require a separate persistence and server strategy.
- Client-side scripts should be added only for genuine interaction that cannot
  be represented through static HTML.

## Development and verification

| Task | Command |
| --- | --- |
| Start development server | `npm run dev` |
| Build static output | `npm run build` |
| Preview static output | `npm run preview` |
| Set up database | `npm run db:setup` |
| Generate a migration | `npm run db:generate` |
| Type-check TypeScript | `npm run typecheck` |
| Type-check Astro | `npm run typecheck:astro` |

Project test and lint commands must be run through the repository's
`quality-checks` skill. Frontend changes also require a direct production build
and Playwright coverage. Data-layer changes require the corresponding Vitest
coverage.

## Branch update log

| Date | Change | Technical impact |
| --- | --- | --- |
| 2026-09-05 | Added this working document | Established the branch's technical baseline and update record |
