# Build Plan: Agent Work Item Tracker

## Build Assumptions

- The repository contains `sam/` for AWS SAM infrastructure and Python Lambda code, `frontend/` for the React/TypeScript Amplify app, and `deploy.sh` for local deployment.
- End-to-end Behave tests live under `sam/tests/e2e/` and run against real deployed dev infrastructure. They do not use mocks.
- The frontend, real dev E2E runner, and future agent clients call the same `/v1` REST API. Frontend-specific behavior is limited to Cognito browser login, CORS, polling, optimistic updates, and presentation.
- Dev and prod can both be seeded with the following projects: `2ndbrain`, `2ndbrain-dev`, `software-factory-dev`, `agent-workflow-queue`, and `example`.
- Cognito client-credentials support is built now for non-interactive E2E tests and future machine clients. Deployed-agent secret delivery is post-MVP; the tracker backend itself does not require agent runtimes.

## Steps

| Step ID | Title | Depends on | What gets built | Definition of Done |
|---|---|---|---|---|
| B-001 | Bootstrap SAM repository and local commands | — | `sam/template.yaml`, `sam/src/`, `sam/tests/unit/`, `sam/tests/e2e/`, `sam/samconfig.toml`, `frontend/`, `deploy.sh`, environment parameter files, and developer README | `sam validate` passes; `deploy.sh --help` describes `dev`, `prod`, `seed`, and `test-e2e`; Python unit-test discovery works under `sam/tests/unit/` |
| B-002 | Configure Cognito web and machine authentication | B-001 | SAM parameters for the existing Slack 2nd Brain Cognito User Pool, tracker web app client, dedicated dev E2E M2M app client/resource-server scopes, and HTTP API JWT authorizer | The real dev E2E client obtains a token from the Cognito token endpoint using `.env.test`/CI secrets; `agent:read` and `agent:write` claims are visible; no secret is committed |
| B-003 | Create DynamoDB tables and repository layer | B-001 | Projects, work-items, activity, and idempotency tables; Python repositories; conditional issue-counter, claim, and version writes; TTL configuration | `pytest sam/tests/unit/` passes; a deployed dev stack exposes all four table names as SAM outputs; conditional writes return the conflict result used by HTTP `409` responses |
| B-004 | Add shared API foundation and project endpoints | B-002, B-003 | Python 3.14 Work Item Lambda, request validation, principal normalization, CORS, structured errors, `GET /health`, `GET /v1/projects`, and `POST /v1/projects` | `@test-env` Behave scenarios can call the real dev API with both an owner JWT and a Cognito client-credentials token; `GET /v1/projects` and `POST /v1/projects` return documented JSON |
| B-005 | Implement core item creation and board reads | B-004 | `POST /v1/projects/{projectKey}/items`, `GET /v1/projects/{projectKey}/board`, `GET /v1/projects/{projectKey}/items/{issueNumber}`, Jira-style IDs, project joins, and `Idempotency-Key` handling | Repeating the same idempotent create returns one item such as `AFT-134`; the board endpoint groups real DynamoDB items by all six statuses; `GET /board` supports conditional unchanged responses |
| B-006 | Implement claims, updates, and flexible status movement | B-005 | `POST .../claim`, `PATCH .../items/{issueNumber}`, version checks, all-column movement, assignee/agent fields, priorities, labels, source references, PR URLs, and dependency IDs | Two real claim requests against the dev stack produce one success and one HTTP `409`; stale `PATCH` returns HTTP `409`; an item can move from any column to any other column |
| B-007 | Implement activity, comments, close/reopen, and soft deletion | B-006 | Activity event writer, append-only comments, activity reads, close/reopen endpoints, dependency summaries, and soft-delete tombstones | `@test-env` scenarios verify append-only activity, close/reopen from Done, optional PR/source links, activity ordering, and permanent issue-number retention |
| B-008 | Build Amplify board against the shared REST API | B-004, B-005 | Cognito/Google login using the tracker app client, Amplify environment configuration, project switcher, six-column board, item cards, counts, and item detail view | Amplify dev build authenticates with the existing Cognito User Pool and reads the same `/v1/projects/{projectKey}/board` response used by agent tests |
| B-009 | Add live polling and optimistic browser mutations | B-006, B-008 | Three-second conditional board polling, ETag/updated cursor handling, drag/drop status mutation, optimistic update rollback, search, filters, create/edit forms, and activity display | A real API mutation made outside the browser appears in the open dev board within five seconds; a rejected drag/drop restores the prior column and displays an error |
| B-010 | Add seed data, end-to-end suite, and local deployment workflow | B-007, B-009 | `sam/tests/e2e/` Behave features and steps, `.env.test`/`.env.prod` templates, `deploy.sh` build/deploy/seed/test commands, and seed data for the five named projects | `cd sam/tests/e2e && behave --tags=@test-env --dry-run` has no undefined steps; live `@test-env` runs pass against dev; `./deploy.sh dev seed` creates the five projects in dev and `./deploy.sh prod seed` can create them in prod |
| B-011 | Configure Amplify production domain and release hardening | B-010 | Amplify Hosting app, existing Cognito callback/logout URLs, `app.agent-queue.nerdthoughts.net`, Route 53 records, CloudWatch structured logs, CORS allowlist, and prod parameter validation | `https://app.agent-queue.nerdthoughts.net` authenticates through Google/Cognito and loads the production board; production smoke tests pass; logs contain request IDs and public item IDs but no tokens or secrets |

## Deployment Order

1. Run `./deploy.sh dev` to build and deploy the SAM backend and dev data resources.
2. Run `./deploy.sh dev seed` to create the five baseline projects.
3. Run `./deploy.sh test-e2e` or execute Behave from `sam/tests/e2e/` against the dev stack.
4. Deploy the Amplify frontend against the dev API URL and validate the shared REST contract.
5. Run production SAM deployment and seed only after dev acceptance tests pass.
6. Configure the Amplify custom domain and production Cognito callbacks.
7. Run the production smoke scenarios; then leave ongoing changes to the local `deploy.sh` workflow.

## Definition of Done for the Build Plan

- All P0 agent lifecycle stories have real `@test-env` coverage against API Gateway, Lambda, Cognito, and DynamoDB.
- The frontend and automated clients use the same documented REST endpoints; future agent clients can reuse the machine-auth flow after MVP.
- Dev and prod deployments are repeatable from local `deploy.sh` commands.
- The five seed projects can be created without GitHub, `bd`, or manual database edits.
- The production domain and Cognito callback configuration are documented and verified.

## Post-MVP Step

| Step ID | Title | Depends on | What gets built | Definition of Done |
|---|---|---|---|---|
| P-001 | Add deployed-agent secret delivery | B-011 | Secrets Manager secret, runtime identity policy, token-client configuration, and optional per-agent client attribution | A deployed agent obtains a short-lived Cognito token without interactive login and calls the same `/v1` endpoints; no credential is baked into the image |
