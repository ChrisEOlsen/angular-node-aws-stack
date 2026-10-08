# System Design — How This Stack Works

An educational tour of the template: what each piece does, why it exists,
and how data and deploys flow through it. For build instructions, see
[AGENTS.md](AGENTS.md).

## The idea in one paragraph

One TypeScript codebase produces one Docker image. That image runs as two
services — a web app and a background worker — on a single EC2 host in
production, or on your laptop via Docker Compose locally. The web app serves
both the JSON API and the Angular frontend. Postgres holds all durable state,
Redis holds only queue data, files live in S3, and GitHub Actions deploys
everything by assuming short-lived AWS credentials (OIDC, no stored keys).

## Big picture

```
                        ┌──────────────────────────────────┐
                        │  EC2 host (prod)                   │
                        │  Docker Compose                    │
  Browser / future iOS  │  ┌────────┐  ┌────────┐            │
  ──────────────────────┼─▶│  app   │  │ worker │  same image│
                        │  │ :3000  │  │        │  diff cmd  │
                        │  └──┬──┬──┘  └──┬──┬──┘            │
                        └─────┼──┼────────┼──┼───────────────┘
                              │  │        │  │
              ┌───────────────┘  │        │  └───────────────┐
              ▼                  ▼        ▼                  ▼
        ┌──────────┐      ┌──────────┐ ┌──────────┐   ┌──────────┐
        │ RDS      │      │ S3       │ │ Redis    │   │ ECR      │
        │ Postgres │      │ files +  │ │ queue    │   │ image    │
        │ sessions │      │ tfstate  │ │ data only│   │ registry │
        │ app data │      │ buckets  │ │          │   │          │
        └──────────┘      └──────────┘ └──────────┘   └──────────┘
              ▲                                              ▲
              │              ┌──────────┐                    │
              └──────────────│ Terraform│────────────────────┘
               provisions    │ flat root│     (EC2+RDS+S3+SGs,
                             │ module   │      default VPC)
                             └──────────┘
                                   ▲
                             ┌──────────┐
                             │ GitHub   │
                             │ Actions  │── assumes IAM role via OIDC
                             │ per-env  │── secrets from Environment
                             └──────────┘

Locally: `docker compose up` gives you app + Postgres + Redis on your
laptop, reading a gitignored `.env`. Same Compose shape as production.
```

## The pieces

### 1. One Node process serves everything (`app`)

The Express server handles three kinds of traffic:

- `GET /api/v1/*` — the REST JSON API. Every future client (web, iOS)
  talks to this. Inputs are validated, errors use the envelope
  `{ error: { code, message } }`, lists paginate with `?page=&per_page=`.
- `/api/auth/*` — login/signup/session endpoints, owned by Better Auth.
- Everything else — the built Angular app (`index.html` + assets) with
  SPA fallback, so client-side routing works on refresh.

Why one process? Fewer moving parts, one image to build, and the app
stays stateless (see below), so a future load balancer can add copies
without changing code.

### 2. Frontend: Angular + Angular CLI + Tailwind, built to static files

The UI is a client-rendered Angular SPA. Angular CLI's `ng build` emits
static files that the Express process serves — there is no separate
production frontend server or hosting service. The browser calls the
same origin's `/api/v1`, so there are no CORS gymnastics.

For frontend development, Angular CLI's `ng serve` uses its built-in Vite
development server; there is no standalone Vite configuration. It supports
HMR for component templates and styles. TypeScript application logic
changes may require a full page reload. Production builds use Angular
CLI's esbuild-based tooling. Angular CLI is allowed for building and
serving, but code generation remains prohibited by the template rules.

### 3. Auth: Better Auth, sessions in Postgres

Better Auth runs inside the Express process (not a separate service).
Sessions, users, and accounts are rows in Postgres — Drizzle's schema
is the single source of truth, including the auth tables. Because
sessions live in the database and not in process memory, any app copy
can serve any logged-in user: the app is stateless.

Secrets: `BETTER_AUTH_SECRET` and `BETTER_AUTH_URL`, from `.env`
locally and GitHub Environment secrets in production.

### 4. Database: Postgres 16, via Drizzle

- Local: Postgres runs as a Compose service on your laptop.
- Prod: Amazon RDS runs Postgres; the app just points at a different
  connection string.
- `src/db/schema.ts` defines every table. Zod validators are derived
  from it with `drizzle-zod` — never hand-duplicated — so validation
  and storage can't drift apart.
- Migrations are generated from the schema and applied to RDS at
  deploy time.
- Hot paths get indexes; related writes are batched. Standard
  efficient-SQL discipline, enforced by convention.

### 5. Background jobs: BullMQ + Redis, separate `worker` service

Some work doesn't belong in a web request: sending email, scheduled
reminders, cleanup, retries. That work goes on BullMQ queues:

- The `app` enqueues jobs; Redis stores the queue; the `worker`
  (same Docker image, different start command) picks jobs up and runs
  them, reading/writing Postgres and S3 as needed.
- Recurring work uses BullMQ repeatable jobs — never `setInterval`
  in the API process, never a separate always-on scheduler.
- Redis holds queue/scheduling data only. If Redis were wiped, you'd
  lose pending jobs but zero system-of-record state — that's all in
  Postgres by rule.

### 6. Files: S3 via presigned URLs

Large files never flow through the app. The flow is:

1. Browser asks the API "I want to upload X."
2. API returns a presigned S3 URL (short-lived, scoped to that object).
3. Browser uploads directly to S3.
4. Browser tells the API "done, here's the key" — the API stores the
   key/reference in Postgres.

Downloads work the same way in reverse. The app stays small and fast.

### 7. Infrastructure: Terraform, one flat root module

`terraform/` is a single flat module (no submodules) provisioning:

- EC2 host (+ security groups, key pair, Elastic IP) in the account's
  default VPC — no custom networking until a plan calls for it.
- RDS Postgres instance (+ subnet/parameter groups).
- S3 buckets for app files, plus the Terraform remote-state bucket
  (remote state from day one, so state never lives on a laptop).
- ECR repository for the Docker image.

No ECS, no load balancer — deliberately. Those get added only when a
project plan explicitly needs them.

### 8. Secrets: `.env` + GitHub Environments, zero third parties

- Local: a gitignored `.env` file (real values). `.env.example` is
  committed with placeholders so newcomers know what to fill in.
- CI/CD: GitHub Environment secrets per deploy target (`staging`,
  `production`, …). Actions injects them into the EC2 host at deploy
  time — never baked into the image.
- No Doppler, no Secrets Manager, no `.env` files committed, no secret
  values ever printed in logs.

### 9. AWS auth: OIDC, no long-lived keys

GitHub Actions never stores `AWS_ACCESS_KEY_ID`. Instead, each job
assumes its environment's IAM role via OpenID Connect:

- AWS trusts GitHub's OIDC provider (`token.actions.githubusercontent.com`).
- Each environment has one IAM role whose trust policy allows only
  `repo:OWNER/REPO:environment:<env>` (exact match — never wildcarded).
- The workflow requests a short-lived token and assumes the role for
  that run only. Leaked credentials expire in minutes, not never.

The very first `terraform apply` (which creates the OIDC provider and
roles) runs locally with your own AWS access — after that, CI owns it.

### 10. CI/CD: GitHub Actions with Environments

`.github/workflows/deploy.yml` runs one job per environment:

```
test → assume AWS role (OIDC) → build image → push to ECR
     → deploy Compose bundle to EC2 → run migrations → verify /healthz
```

A relaunch is a workflow re-run, not a runbook. Teardown is a separate
manually-dispatched workflow (`.github/workflows/teardown.yml`) with a
confirmation gate and a backup-first step — destructive by design.

## How a request flows (examples)

**Page load:** browser → EC2 → Express serves `index.html` + assets →
Angular boots → calls `GET /api/v1/...` on the same origin.

**Signup:** browser → `/api/auth/sign-up` → Better Auth validates,
writes user + session rows to Postgres → returns session cookie →
browser stores it, sends it on later requests.

**Creating a record:** browser → `POST /api/v1/widgets` with JSON →
drizzle-zod validates → Drizzle inserts into Postgres → API returns
the created object (or `{ error: { code, message } }` on failure).

**Slow side effect:** API writes the record, enqueues a BullMQ job
("send confirmation email"), returns immediately → worker picks up
the job from Redis and sends the email.

## Environments

`staging` and `production` are the conventional pair, but the pattern
is "whatever the plan calls for." Each environment is: a GitHub
Environment (secrets), an IAM role (trust-scoped to that environment),
and its own Terraform state + infra (or at minimum its own host and
database). Adding an environment means adding those three — the
workflow matrix handles the rest.

## Local development

```
cp .env.example .env   # fill in local values
docker compose up      # app + Postgres + Redis, fully local
```

No AWS access needed. The stack only touches AWS if you explicitly
pass an override — and the agent must ask before doing that.

## Design principles to keep

1. **Stateless app** — sessions in Postgres, files in S3, nothing on
   local disk, no in-process cron. Any copy can serve any request.
2. **Same Compose shape everywhere** — local and prod differ in
   connection strings, not architecture.
3. **Postgres + Redis only** — one system of record, one queue store.
4. **Stable JSON API** — a future iOS app consumes the same `/api/v1`.
5. **Boring until proven otherwise** — no load balancer, ECS, custom
   VPC, or Secrets Manager until a plan demands them.
