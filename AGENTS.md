## How to run this project (read this first)

You are the build agent. Do not write any code until you have completed Phase 1 and Phase 2 below. The user wants a rigorous planning stage before any implementation.

### Phase 1 — Questions and planning

1. Ask the user every question you need answered to build this project: what the app does, who uses it, the core features, what data it stores, what files it handles, what background jobs and scheduled jobs it needs, and what the design should look like. Keep asking until nothing material is unanswered.
2. Write up a plan covering features, data model, API endpoints, pages, background and scheduled jobs (BullMQ queues and repeatable jobs), infrastructure changes (only if the Terraform needs anything beyond the base topology), and a launch checklist. Get the user's explicit approval on the plan before building anything.
3. Design: suggest the user ask their Muse agent to explore design directions using the Figma connector, then hand you the resulting design or Figma link to implement. Do not invent a full visual design unprompted. Implement the design you are given, or ask for one.

### Phase 2 — Launch credentials check

AWS keys live in Doppler dev secrets, not in the terminal. Verify all three of these before writing any launch code:

- DOPPLER\_TOKEN exists in the environment. This one cannot come from Doppler itself, so it must be set in the terminal.
- AWS\_ACCESS\_KEY\_ID exists in the Doppler dev config for this app's project.
- AWS\_SECRET\_ACCESS\_KEY exists in the Doppler dev config for this app's project.

Check the Doppler names with `doppler secrets` (secret names only, never print values). Terraform and the AWS CLI read these exact environment variable names, so run them under `doppler run --config dev -- <command>` — unlike other stacks, no name bridging is needed here.
Never put the keys in the terminal environment permanently; always fetch them from Doppler at runtime.

If DOPPLER\_TOKEN is missing, stop and tell the user to set it in the terminal. If either AWS key is missing from the Doppler dev config, stop and tell the user to add it there, showing the permission list below. Do not attempt a launch without all three.

When the AWS keys are missing or insufficient, show the user this exact permission list so they can create a proper IAM user (AWS console, then IAM, then Users, then Create user with programmatic access, then attach a customer-managed policy with):

- EC2: full (instances, security groups, key pairs, Elastic IPs). The Terraform operates in the account's default VPC — do not build custom networking unless a plan explicitly calls for it.
- RDS: full (Postgres instances, subnet groups, parameter groups).
- S3: full (app file buckets, plus the Terraform state bucket).
- ECR: full (image repositories).

The user can sanity-check the keys by running `doppler run --config dev -- aws sts get-caller-identity` — it should succeed and show the expected account.

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
- Secrets: Doppler only. No .env files anywhere, ever.
- CI/CD: GitHub Actions (test, then build image, then push to ECR, then deploy to EC2).
- Frontend: React plus Vite plus Tailwind CSS, built to static files and served by the same Express process as the API.

Conventions:

- One Node process serves everything: the Express API under /api/v1 as REST JSON, Better Auth at /api/auth/\*, and the frontend static assets for everything else with SPA fallback.
- The app is stateless: any number of copies must be able to run behind a future load balancer. Sessions live in Postgres (Better Auth), no local disk state, no in-process cron. The BullMQ worker runs from the same image as a separate Compose service with its own command.
- The Drizzle schema in src/db/schema.ts is the single source of truth for all tables, including Better Auth's tables. Derive Zod schemas from it with drizzle-zod. Never hand-write duplicate schemas.
- Validate every API input. Errors use the envelope { error: { code, message } }. Paginate lists with ?page= and ?per\_page=.
- Better Auth runs inside the Express process with its data in Postgres. Its secrets (BETTER\_AUTH\_SECRET, BETTER\_AUTH\_URL) come from Doppler and are injected at deploy time.
- Files go in S3. Use presigned URLs so browsers upload directly instead of proxying large files through the app.
- Background work goes on BullMQ queues. Recurring work goes on BullMQ repeatable jobs. Never put business logic on setInterval in the API process, and never build an always-on scheduler process outside the worker.
- Write efficient SQL: index every hot path and batch related writes.
- Local development is `doppler run -- docker compose up`, which gives a full local stack: the app, Postgres, and Redis. It runs fully local and never needs the AWS keys; it only goes out to AWS if the agent passes an override to do so, which it should not do without asking first.
- Keep the JSON API stable and documented. A future iOS app will consume this same API.

### Phase 4 — Launch

Prerequisites: DOPPLER\_TOKEN is set in the terminal and both AWS keys are in the Doppler dev config (Phase 2 verified). Run every Terraform, AWS CLI, and Docker command under `doppler run --config dev -- ...` so the keys resolve at runtime.

1. Install dependencies, run tests, and build the frontend.
2. Run `terraform init` and `terraform apply` to provision the EC2 host, RDS database, S3 buckets, and ECR repository.
3. Build the image, push it to ECR, and deploy the Compose bundle to the EC2 host with secrets injected from Doppler at deploy time.
4. Apply the Drizzle migrations to the RDS database.
5. Set the secrets in Doppler (generate BETTER\_AUTH\_SECRET, set BETTER\_AUTH\_URL to the final URL).
6. Verify on the live URL: /healthz responds, signup and login work, and one record can be created end to end through the API.

Encode these steps in scripts/launch.sh so a future agent can relaunch without rethinking them. The script must resolve AWS credentials from Doppler at runtime, and must fail with a clear message if any are missing.

### Phase 5 — Teardown (unlaunch)

Only do this when the user explicitly asks to take the project down. Teardown is destructive and permanent: deleted data cannot be recovered. Always get the user's explicit confirmation before running any of it, and never tear down as part of a launch, redeploy, or routine cleanup. Run every command here under `doppler run --config dev -- ...`.

1. Optional but recommended: back up data before deleting anything. Dump the RDS database with pg_dump. Download any S3 objects worth keeping. Ask the user if they want backups; if they skip it, proceed without.
2. If the app uses a custom domain, remove the DNS record pointing at the Elastic IP first.
3. Empty the S3 app buckets by deleting all of their objects (Terraform cannot destroy a non-empty bucket).
4. Run `terraform destroy` to remove the EC2 host, RDS database, ECR repository, and security groups.
5. Delete the Terraform state bucket only after confirming the destroy completed and no other project shares it.
6. In the Doppler dashboard, archive or delete the app's Doppler project so its secrets are gone too. Deactivate the IAM user keys used for this deployment.
7. Verify everything is gone: the AWS console should show no remaining EC2, RDS, S3, or ECR resources for this app. Hitting the old URL should fail to connect, not reach the application.

Encode these steps in scripts/teardown.sh so a future agent can tear down without rethinking them. The script must resolve AWS credentials from Doppler at runtime, ask for confirmation before deleting anything, and offer the backup step first.

## Rules

- Postgres plus Redis only. No .env files. No code generator CLI. Single tenant.
- Redis holds queue and scheduling data only, never system-of-record state.
- Never tear down without the user's explicit approval.
- TypeScript strict. Keep the code plain and simple, no clever abstractions.
- Never commit secrets. Never print tokens.
