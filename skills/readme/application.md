# Application / API Service README

Applications and services are things people run, not import. The README needs to get
someone from clone to running locally, and point them to deployment docs.

## Sections (in order)

### Prerequisites
List what needs to be installed before setup. Be specific about versions. Only list things
that aren't obvious (don't say "a computer" or "an internet connection").

**Pattern:**
```markdown
## Prerequisites

- Node.js 20+
- PostgreSQL 15+
- Redis 7+ (for job queue)
```

### Setup
Step-by-step from clone to running. Number the steps. Keep it to the minimum viable path
— don't cover every optional service.

**Pattern:**
```markdown
## Setup

1. Clone and install dependencies:
   ```bash
   git clone https://github.com/user/my-app && cd my-app
   npm install
   ```

2. Set up the database:
   ```bash
   cp .env.example .env      # then edit .env with your DB credentials
   npm run db:migrate
   ```

3. Start the dev server:
   ```bash
   npm run dev
   ```

   Open http://localhost:3000.
```

If there's a Docker path that's simpler, show that first and put the manual setup in a
`<details>` block (or vice versa — whichever is the primary development workflow).

### Environment variables
Show a table of required env vars. Don't list every optional var — just the ones needed to
run. Link to `.env.example` for the full set.

**Pattern:**
```markdown
## Configuration

| Variable | Description | Required |
|----------|-------------|----------|
| `DATABASE_URL` | PostgreSQL connection string | Yes |
| `REDIS_URL` | Redis connection string | Yes |
| `SECRET_KEY` | Session signing key | Yes |
| `PORT` | Server port (default: 3000) | No |

Copy `.env.example` to `.env` and fill in the values.
```

### Architecture (optional, brief)
Only if the project has a non-obvious structure. A 2-3 sentence description of the major
components and how they connect. A simple ASCII diagram or a single paragraph — not a
multi-page architecture doc.

```markdown
## Architecture

The app has three components: a Next.js frontend, an Express API server, and a PostgreSQL
database. The API server also runs background jobs via a Redis-backed queue. See
[docs/architecture.md](docs/architecture.md) for details.
```

### Deployment (link only)
Don't put full deployment instructions in the README. Link to a deployment guide.

```markdown
## Deployment

See [docs/deployment.md](docs/deployment.md) for production setup.
```

If no deployment doc exists yet, include a one-line note: "Deployment docs coming soon."

### Development
Brief notes on running tests and the dev workflow.

```markdown
## Development

```bash
npm test           # run tests
npm run lint       # check code style
npm run db:seed    # load sample data
```
```

## Things to skip for applications
- API endpoint documentation (that belongs in API docs, an OpenAPI spec, or a docs site)
- Detailed database schema (link to migrations or an ERD)
- Full infrastructure diagrams
- Production configuration details
