## How to run this project (read this first)

You are the build agent. Do not write any code until you have completed Phase 1 and Phase 2 below. The user wants a rigorous planning stage before any implementation.

### Phase 1 — Questions and planning

1. Ask the user every question you need answered to build this project: what the app does, who uses it, the core features, what data it stores, what files it handles, what background jobs and scheduled jobs it needs, and what the design should look like. Keep asking until nothing material is unanswered.
2. Write up a plan covering features, data model, API endpoints, pages, background and scheduled jobs (BullMQ queues and repeatable jobs), infrastructure changes (only if the Terraform needs anything beyond the base topology), and a launch checklist. Get the user's explicit approval on the plan before building anything.
3. Design: suggest the user ask their Muse agent to explore design directions using the Figma connector, then hand you the resulting design or Figma link to implement. Do not invent a full visual design unprompted. Implement the design you are given, or ask for one.

### Phase 2 — Launch credentials check

Secrets live in two places: a gitignored `.env` file for local dev, and GitHub Environment secrets for deploys. AWS auth uses OIDC (GitHub Actions assumes an IAM role), with no long-lived AWS keys stored anywhere. Verify all of these before writing any launch code:

- `.env.example` exists in the repo (committed, placeholder values only) and `.env` exists locally (gitignored, real local values).
- GitHub Environments exist for each deploy target (e.g. `staging`, `production`) with the required secrets set.
- The AWS OIDC provider for GitHub (`token.actions.githubusercontent.com`) exists, plus one IAM role per environment whose trust policy allows only that repo and environment, with the permissions below. Each role's trust policy uses `sts:AssumeRoleWithWebIdentity` with a `StringEquals` condition on `token.actions.githubusercontent.com:aud` = `sts.amazonaws.com` and `token.actions.githubusercontent.com:sub` = `repo:OWNER/REPO:environment:<env>`. Never wildcard the `sub` claim.
- Local AWS access works for one-off Terraform/CLI use (AWS SSO / IAM Identity Center or a named CLI profile). Never commit AWS keys.

Check GitHub secret names with `gh secret list --env <name>` (secret names only, never print values). Actions jobs use `permissions: id-token: write, contents: read` and assume the environment's role (e.g. via `aws-actions/configure-aws-credentials` with `role-to-assume`) to get short-lived credentials.

If the environments, OIDC provider, or roles are missing, stop and tell the user to create them, showing the permission list below. Do not attempt a launch without them.

When the AWS roles are missing or insufficient, show the user this exact permission list so they can create proper IAM roles (AWS console, then IAM, then Identity providers, then add GitHub as an OIDC provider, then Roles, then Create role for Web identity, scoped via the trust policy to this repo and environment, then attach a customer-managed policy with):

- EC2: full (instances, security groups, key pairs, Elastic IPs). The Terraform operates in the account's default VPC — do not build custom networking unless a plan explicitly calls for it.
- RDS: full (Postgres instances, subnet groups, parameter groups).
- S3: full (app file buckets, plus the Terraform state bucket).
- ECR: full (image repositories).

The user can sanity-check local AWS access by running `aws sts get-caller-identity` — it should succeed and show the expected account. In CI, the OIDC assume-role step should succeed and show the expected account and role.

### Context7 — look up current API docs

Libraries change. Before writing code that touches any library API, look up its current documentation with Context7 instead of relying on memory. This applies to everything in the locked stack: Express, Drizzle, drizzle-zod, Better Auth, BullMQ, React, Vite, Tailwind, Terraform (AWS provider), the AWS EC2/RDS/S3/ECR APIs, and Docker Compose.

1. Check whether Context7 is available in your environment (MCP tools named `resolve-library-id` and `get-library-docs`).
2. If it is not available, tell the user it is missing and how to add it: `claude mcp add --transport http context7 https://mcp.context7.com/mcp`. Then continue, flagging anything you could not verify.
3. Workflow: call `resolve-library-id` with the library name to get its Context7 ID, then call `get-library-docs` with that ID and the topic you need. Do this before writing Express routes, Drizzle queries and migrations, BullMQ workers, Better Auth configuration, Terraform, and Compose files.
4. When the docs disagree with your assumptions, the docs win.

### Phase 3 — Build

The stack (locked, do not substitute):

- Runtime: Node 22 plus TypeScript strict. Express (API), Drizzle (database access), Zod via drizzle-zod (validation), BullMQ plus Redis (background jobs and repeatable scheduled jobs), Better Auth (self-hosted logins).
- Data: Postgres 16 — a Docker Compose service locally, RDS in production. Redis holds queue and scheduling data only, never system-of-record state. No other datastores.
- Files: S3 via presigned URLs.
- Containers: Docker Compose locally and in production — the same Compose shape in both places.
- Infrastructure: Terraform, one flat root module (EC2 host plus RDS plus S3 buckets plus security groups, in the default VPC, remote state in S3 from day one). No Terraform modules, no ECS, no load balancer until a plan explicitly calls for them.
- Secrets: gitignored `.env` locally plus GitHub Environment secrets in CI/CD. Commit `.env.example` with placeholder values only. No real secrets in the repo, ever.
- CI/CD: GitHub Actions with Environments (test, then assume the environment's AWS role via OIDC, then build image, then push to ECR, then deploy to EC2). One job per environment (staging, production, whatever the plan calls for).
- Frontend: React plus Vite plus Tailwind CSS, built to static files and served by the same Express process as the API.

Conventions:

- One Node process serves everything: the Express API under /api/v1 as REST JSON, Better Auth at /api/auth/\*, and the frontend static assets for everything else with SPA fallback.
- The app is stateless: any number of copies must be able to run behind a future load balancer. Sessions live in Postgres (Better Auth), no local disk state, no in-process cron. The BullMQ worker runs from the same image as a separate Compose service with its own command.
- The Drizzle schema in src/db/schema.ts is the single source of truth for all tables, including Better Auth's tables. Derive Zod schemas from it with drizzle-zod. Never hand-write duplicate schemas.
- Validate every API input. Errors use the envelope { error: { code, message } }. Paginate lists with ?page= and ?per\_page=.
- Better Auth runs inside the Express process with its data in Postgres. Its secrets (BETTER\_AUTH\_SECRET, BETTER\_AUTH\_URL) come from `.env` locally and from GitHub Environment secrets in deploys, injected at deploy time (written to the host env file or passed to Compose — never baked into the image).
- Files go in S3. Use presigned URLs so browsers upload directly instead of proxying large files through the app.
- Background work goes on BullMQ queues. Recurring work goes on BullMQ repeatable jobs. Never put business logic on setInterval in the API process, and never build an always-on scheduler process outside the worker.
- Write efficient SQL: index every hot path and batch related writes.
- Local development is `docker compose up`, which reads `.env` and gives a full local stack: the app, Postgres, and Redis. It runs fully local and never needs AWS access; it only goes out to AWS if the agent passes an override to do so, which it should not do without asking first.
- Keep the JSON API stable and documented. A future iOS app will consume this same API.

### Phase 4 — Launch

Prerequisites: Phase 2 verified (GitHub Environments plus OIDC roles plus `.env.example`). All authenticated AWS and deploy steps run in GitHub Actions per environment, assuming that environment's IAM role via OIDC.

1. Install dependencies, run tests, and build the frontend (CI).
2. Run `terraform init` and `terraform apply` to provision the EC2 host, RDS database, S3 buckets, and ECR repository (via the CI OIDC role, or locally with SSO for first bootstrap — document which).
3. Build the image, push it to ECR, and deploy the Compose bundle to the EC2 host with secrets from the GitHub Environment injected at deploy time.
4. Apply the Drizzle migrations to the RDS database.
5. Set the per-environment secrets in GitHub (generate BETTER\_AUTH\_SECRET, set BETTER\_AUTH\_URL to the final URL).
6. Verify on the live URL: /healthz responds, signup and login work, and one record can be created end to end through the API.

Encode these steps in `.github/workflows/deploy.yml` (one job per environment) so a future agent can relaunch with a re-run instead of rethinking them. The workflow must fail with a clear message if any required secret or role is missing.

### Phase 5 — Teardown (unlaunch)

Only do this when the user explicitly asks to take the project down. Teardown is destructive and permanent: deleted data cannot be recovered. Always get the user's explicit confirmation before running any of it, and never tear down as part of a launch, redeploy, or routine cleanup. Run teardown via a manually-dispatched GitHub Actions workflow per environment (OIDC), or locally with SSO — document which.

1. Optional but recommended: back up data before deleting anything. Dump the RDS database with pg_dump. Download any S3 objects worth keeping. Ask the user if they want backups; if they skip it, proceed without.
2. If the app uses a custom domain, remove the DNS record pointing at the Elastic IP first.
3. Empty the S3 app buckets by deleting all of their objects (Terraform cannot destroy a non-empty bucket).
4. Run `terraform destroy` to remove the EC2 host, RDS database, ECR repository, and security groups.
5. Delete the Terraform state bucket only after confirming the destroy completed and no other project shares it.
6. In the GitHub repo settings, remove the app's Environment secrets for the torn-down environment. Remove or disable the OIDC IAM roles used for this deployment.
7. Verify everything is gone: the AWS console should show no remaining EC2, RDS, S3, or ECR resources for this app. Hitting the old URL should fail to connect, not reach the application.

Encode these steps in `.github/workflows/teardown.yml` (workflow_dispatch, environment-scoped) so a future agent can tear down without rethinking them. The workflow must ask for confirmation before deleting anything, and offer the backup step first.

## Rules

- Postgres plus Redis only. `.env` gitignored plus `.env.example` committed. No real secrets in the repo. No code generator CLI. Single tenant.
- Redis holds queue and scheduling data only, never system-of-record state.
- Never tear down without the user's explicit approval.
- TypeScript strict. Keep the code plain and simple, no clever abstractions.
- Never commit secrets. Never print secret values.
