# Project instructions

## Scope and decision rules

You are the build agent. Follow the locked stack and conventions below.

- For the initial application build, complete Phase 1 and Phase 2 before writing implementation code. Missing deployment credentials block initial implementation as well as launch.
- Read-only investigation, reviews, planning, and documentation edits do not require the build phases. Preparing placeholder configuration and the approved bootstrap prerequisites is also allowed before implementation.
- For subsequent features and fixes, reuse approved decisions. Ask about and obtain approval for material changes to scope, architecture, data handling, or design; do not restart discovery for routine maintenance.
- Before every launch, recheck Phase 2 for the target environment.
- An approved build plan does not authorize deployment or teardown. Deploy when the user requests it. Teardown requires the separate confirmation in Phase 5.
- If requirements conflict or a material decision is missing, explain the specific conflict and ask a focused question. Do not silently substitute technologies or relax security rules.
- Never commit secrets or print secret values. Inspect secret names, configuration keys, and presence only.

## Phase 1 — Discover and approve the plan

Ask focused questions, grouped by topic. Reuse answers already given. Resolve decisions that affect scope, architecture, security, or acceptance criteria; propose defaults for minor implementation details instead of asking indefinitely.

Cover:

- Purpose, users, roles, and core user flows. The application is single tenant.
- Features, stored data, relationships, retention, and deletion behavior.
- File types, sizes, access rules, and upload/download flows.
- Background work, recurring work, time zones, retry behavior, and failure handling.
- Pages, navigation, and visual design.
- Deploy environments, AWS account and region, domain and HTTPS, and launch acceptance criteria.

For design, suggest that the user ask their Muse agent to explore directions using the Figma connector and provide the resulting design or Figma link. Implement the supplied design. If none is supplied, ask for one or for explicit authorization to propose a design; do not invent a full visual design unprompted.

Write a reviewable plan covering:

1. Features and acceptance criteria.
2. Data model, indexes, and authorization rules.
3. REST API endpoints, validation, pagination, and errors.
4. Pages and the approved design reference.
5. BullMQ queues and recurring jobs, including retries and idempotency.
6. Infrastructure changes beyond the base topology, if any.
7. Local file behavior, deployment transport to EC2, migration execution, HTTPS, and rollout/recovery procedure.
8. Required configuration and secret names for local development and each environment.
9. Bootstrap ownership and the launch checklist.

Obtain the user's explicit approval of the plan before implementation. Record the approved decisions in repository documentation without secrets.

## Phase 2 — Verify launch prerequisites

Secrets live in a gitignored `.env` for local development and GitHub Environment secrets for deployment. AWS deployment authentication uses GitHub Actions OIDC; never store long-lived AWS keys.

Verify before initial implementation and again before launch:

- `.env.example` is committed and contains placeholders only. `.env` exists locally, is gitignored, and has the required local values. Do not display its contents.
- Every approved deploy target has a GitHub Environment and the required secret names. Check with `gh secret list --env <name>`; listing names does not verify secret values.
- The GitHub OIDC provider `token.actions.githubusercontent.com` exists in the expected AWS account.
- Each environment has its own IAM role with the trust policy and permissions below.
- Local AWS access works through SSO / IAM Identity Center or a named profile using temporary credentials. Use `aws sts get-caller-identity` to verify the expected account. Never commit AWS credentials.
- The Terraform state bucket exists and the approved backend configuration identifies the correct bucket, region, and environment-specific state key.

If environments, the OIDC provider, roles, or required configuration are missing, report exactly what is missing and stop implementation and launch. Ask the user to create missing environments/provider/roles; do not create them implicitly. If access prevents verification, report the check as unverified rather than treating it as passed.

### OIDC trust and permissions

Each role must allow `sts:AssumeRoleWithWebIdentity` for the GitHub OIDC provider with `StringEquals` conditions:

- `token.actions.githubusercontent.com:aud` = `sts.amazonaws.com`
- `token.actions.githubusercontent.com:sub` = `repo:OWNER/REPO:environment:<env>`

Never wildcard the `sub` claim. Actions deployment jobs must declare their GitHub Environment, use `permissions: { id-token: write, contents: read }`, and assume that environment's role, for example with `aws-actions/configure-aws-credentials` and `role-to-assume`. Verify that CI assumes the expected account and role.

When roles are missing or insufficient, direct the user to AWS console → IAM → Identity providers to add GitHub as an OIDC provider, then Roles → Create role → Web identity. Explain the exact repo/environment trust scope above and show this permission list verbatim for the customer-managed policy:

- EC2: full (instances, security groups, key pairs, Elastic IPs). The Terraform operates in the account's default VPC — do not build custom networking unless a plan explicitly calls for it.
- RDS: full (Postgres instances, subnet groups, parameter groups).
- S3: full (app file buckets, plus the Terraform state bucket).
- ECR: full (image repositories).

If the approved deployment transport or other planned resources require additional permissions, identify them in the plan and ask the user to provision them. Do not assume the list grants permissions to other services.

### Bootstrap exception

Before implementation, the agent may prepare `.gitignore`, a placeholder-only `.env.example`, and bootstrap documentation. Local secret values must be supplied or generated without printing them.

The state bucket must exist before the application's first `terraform init`. The approved plan must name who creates it and how. The user may create it, or explicitly authorize a one-time bootstrap with local SSO. Keep its lifecycle separate from the application Terraform root; document the bootstrap and cleanup procedure. Application Terraform uses S3 remote state from its first initialization.

Values available only after provisioning, such as an RDS endpoint, must be identified as deployment outputs in the plan. Their absence is not a missing pre-provisioning secret. Required user-supplied secrets must exist before provisioning/deployment; outputs must be resolved before starting the application.

## Documentation lookup before implementation

Before writing code that uses a library or infrastructure API, verify the relevant API against official documentation for the selected version. This applies to Express, Drizzle, drizzle-zod, Zod, Better Auth, BullMQ, React, Vite, Tailwind, Terraform and its AWS provider, AWS services, and Docker Compose.

1. Check for Context7 tools that resolve a library ID and retrieve documentation. Tool names can vary by integration.
2. Resolve the library, then retrieve documentation for the relevant topic and version before implementing it. Reuse verified documentation within the task unless the topic or version changes.
3. If Context7 is unavailable, tell the user and use official documentation as the fallback. For Claude Code, the installation command is `claude mcp add --transport http context7 https://mcp.context7.com/mcp`; other clients require their own MCP setup.
4. Flag anything that could not be verified. Do not invent APIs. Documentation determines API usage, but does not authorize changing the locked stack or approved plan.

## Phase 3 — Build

### Locked stack

- Node 22, TypeScript strict, Express, Drizzle, Zod with drizzle-zod, BullMQ with Redis, and self-hosted Better Auth.
- Postgres 16: Docker Compose locally, RDS in deployed environments. Postgres is the system of record. Redis stores queue and scheduling data only. No other datastores.
- S3 for deployed files, using presigned URLs for direct browser upload/download.
- Docker Compose locally and on EC2. Build the frontend with React, Vite, and Tailwind CSS; serve its static files from Express.
- Terraform in one flat application root module: EC2 host, RDS, S3 app buckets, ECR, and security groups in the account's default VPC. No child modules, ECS, custom networking, or load balancer unless an approved plan explicitly allows them.
- GitHub Actions with one deployment job per approved environment, using GitHub Environments and OIDC.

### Processes and local development

- One Node API process serves REST JSON under `/api/v1`, Better Auth under `/api/auth/*`, and frontend assets with an SPA fallback. Unknown API routes must return API errors rather than the SPA.
- Run the BullMQ worker as a separate Compose service using the same image with its own command. Do not run business schedules with `setInterval`, in-process cron, or a separate always-on scheduler process.
- Use BullMQ's documented recurring scheduling API for the selected version. Register schedules idempotently from worker startup.
- Keep the application stateless and ready for multiple copies. Better Auth sessions live in Postgres. No persistent application state on local disk.
- `docker compose up` reads `.env` and starts the full local stack: app, worker, Postgres, and Redis. It must not require AWS credentials or contact AWS by default.
- Share the app/worker/Redis service definitions between local and production Compose configurations. Use explicit profiles or overrides so production uses RDS and does not start a local Postgres service. Document both commands.
- By default, local S3-dependent features are disabled with a clear unavailable response/UI state. Do not silently store files on disk or add a storage emulator. If local upload testing is required, resolve and approve an explicit exception in the plan. Connecting local development to AWS requires the user's permission and an explicit override.

### Data and API conventions

- `src/db/schema.ts` is the single source of truth for all tables, including Better Auth tables.
- Derive database-field validation from Drizzle using drizzle-zod. Do not duplicate table schemas. Define request-only schemas for pagination, filters, and action inputs as needed, composing derived schemas where applicable.
- Validate every API input and enforce authorization. Application API errors use `{ error: { code, message } }`. Preserve Better Auth's required protocol responses under `/api/auth/*`.
- Paginate lists with `?page=` and `?per_page=`; document defaults and limits.
- Index hot query paths and batch related writes. Use transactions when related changes must succeed together.
- Send background work to BullMQ queues; make retryable operations idempotent. Keep durable business state in Postgres.
- Keep the JSON API stable and documented for a future iOS client.
- Supply `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, and other runtime configuration through local `.env` or environment-scoped deployment configuration. Inject deployment secrets into the host environment file or Compose at deploy time, never into the image.
- Use plain, simple code. No code generator CLI.

## Phase 4 — Launch

Launch only when requested and after the target environment passes Phase 2. Routine authenticated AWS and deployment operations run in GitHub Actions using that environment's OIDC role. The only local bootstrap exception is the explicitly approved SSO procedure documented in Phase 2.

Encode the deployment in `.github/workflows/deploy.yml`, with one deployment job per approved environment:

1. Validate required configuration and secret presence without printing values; fail with a clear message naming missing keys or role configuration.
2. Install dependencies, run appropriate tests, and build the frontend.
3. Assume the environment's AWS role through OIDC and verify the expected account/role.
4. Run `terraform init`, review the planned changes, and apply the approved infrastructure. Resolve deployment outputs.
5. Build the image and push it to ECR.
6. Prepare the Compose bundle and runtime environment on EC2 using the transport specified in the plan. Ensure secrets, including `BETTER_AUTH_SECRET`, and the final `BETTER_AUTH_URL` are configured before startup.
7. Run Drizzle migrations against RDS from the execution location specified in the plan before activating code that requires them. Prevent concurrent migration runs and stop rollout on failure.
8. Activate the new app/worker services and verify `/healthz`, signup/login, and creation of a record through the API. Use designated smoke-test data and accounts.

Document deployment commands, migration compatibility, recovery steps, and any manual bootstrap. A workflow rerun must reuse existing infrastructure safely. Do not use teardown as a recovery strategy.

## Phase 5 — Teardown

Teardown is destructive and may permanently delete data. Never perform it during launch, redeployment, or routine cleanup.

Only proceed after the user explicitly requests teardown and confirms the exact environment and resources to delete. Offer database and file backups before any deletion; record whether the user accepts or skips them. Backups must be stored outside resources being destroyed.

Encode teardown in `.github/workflows/teardown.yml` using `workflow_dispatch`, an environment-scoped OIDC job, an explicit environment confirmation input, and a backup choice. Fail before deletion if confirmation is absent or mismatched. A requested backup must complete successfully before deletion proceeds.

1. If selected, dump RDS with `pg_dump` and copy S3 objects to the approved backup destination.
2. Remove the app's DNS record, if applicable, using the approved DNS procedure.
3. Empty the target app's S3 buckets, including versions and delete markers where applicable. Do not empty unrelated or backup buckets.
4. Review the destroy scope, then run `terraform destroy` for the target environment to remove its application resources. Account explicitly for RDS deletion settings and ECR images.
5. Delete the state bucket only after confirming destroy completed, no other project or environment shares it, and its deletion is included in the user's confirmation.
6. Remove the target environment's app secrets and remove or disable its deployment role after the AWS cleanup is complete. If the workflow lacks the necessary GitHub or IAM permissions, provide the user with the exact remaining manual steps; do not claim cleanup is complete.
7. Verify that the target application's AWS resources are gone and its old URL no longer serves the application. Report any remaining backups, shared resources, or manual cleanup.

Local teardown with SSO is allowed only when explicitly confirmed and documented instead of the Actions procedure. Apply the same scope, backup, and confirmation requirements.
