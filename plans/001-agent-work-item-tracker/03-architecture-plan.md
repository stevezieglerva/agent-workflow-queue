# Architecture Plan: Agent Work Item Tracker

## Architecture Decisions

- Deploy the backend with AWS SAM into Steve’s personal AWS account.
- Use Python 3.14 on AWS Lambda; AWS Lambda currently supports the 3.14 runtime.
- Use API Gateway HTTP API rather than API Gateway REST API. HTTP API pricing is approximately `$0.97 per million` requests versus `$3.50 per million` for REST API requests, and HTTP API supports Cognito JWT authorization. At this workload the dollar difference is small, but HTTP API is the simpler fit.
- Host the frontend with AWS Amplify Hosting at `https://app.agent-queue.nerdthoughts.net`.
- Reuse the existing Slack 2nd Brain Cognito User Pool and Google identity provider, but create a new app client and callback/logout URLs for this tracker.
- Use separate DynamoDB tables because this is a small personal system where obvious table boundaries are easier to maintain than a single-table design.
- Use three-second conditional polling for live updates. The browser updates without manual refresh and uses `ETag`/`If-None-Match` or an equivalent `updated_since` cursor to avoid rewriting unchanged board data.

## Components

| Name | Responsibility | Language/Runtime | Inputs | Outputs |
|---|---|---|---|---|
| Amplify Web App | Serves the authenticated board UI, project switcher, filters, item detail, and optimistic drag/drop | React + TypeScript, Amplify Hosting | Cognito session, REST responses | HTML/JS/CSS, REST requests |
| Cognito User Pool | Authenticates the owner through the existing Google identity provider and issues web JWTs | Managed AWS service | Google login, web OAuth requests | ID/access tokens |
| Cognito M2M App Client | Issues short-lived access tokens for ECS agents using `client_credentials` | Managed AWS service | Client ID and secret, requested scopes | `agent:read` / `agent:write` access tokens |
| API Gateway HTTP API | Public REST edge, CORS, throttling, JWT validation, and routing | Managed AWS service | HTTPS requests with Cognito JWT | Lambda invocations, HTTP responses |
| Work Item Lambda | Implements REST use cases, authorization scopes, optimistic concurrency, idempotency, and table joins | Python 3.14 | API Gateway event, Cognito claims | JSON responses and DynamoDB writes |
| Projects Table | Stores project identity, Jira-style key, metadata, and issue-number counter | DynamoDB on-demand | Project CRUD commands | Project records |
| Work Items Table | Stores work items, status, metadata, version, and stable project relationship | DynamoDB on-demand | Item CRUD and transition commands | Item records and board query results |
| Activity Table | Stores append-only comments and system activity events | DynamoDB on-demand | Item event commands | Activity history |
| Idempotency Table | Prevents duplicate creates and repeated mutation side effects | DynamoDB on-demand with TTL | Client ID, idempotency key, request hash | Original response/resource reference |
| Secrets Manager | Stores the Cognito M2M app client secret for ECS agents | Managed AWS service | ECS task-role read | Runtime secret value |
| ECS Agent Task | Runs coding agents that call the REST API headlessly | Existing ECS container runtime | Task role, Cognito token, work instructions | REST mutations and reads |
| CloudWatch Logs | Captures structured application and access logs without token values | Managed AWS service | Lambda/API logs | Debugging and operational history |
| SAM Stack | Defines and deploys Lambda, API Gateway, DynamoDB, IAM, and configuration | AWS SAM/CloudFormation | Template and parameter files | AWS resources per environment |
| Route 53 + Amplify Domain | Routes the application hostname to Amplify Hosting and manages certificate validation | Managed AWS services | DNS records | `app.agent-queue.nerdthoughts.net` |

## Data Entities

| Name | Fields | Persistence |
|---|---|---|
| Project | `owner_id`, `project_id`, `project_key`, `name`, `description`, `issue_counter`, `created_at`, `updated_at`, `archived_at` | `projects` DynamoDB table; project key is unique and never reused |
| WorkItem | `project_id`, `project_key`, `issue_number`, `public_id`, `title`, `description`, `status`, `priority`, `labels[]`, `assignee`, `claimed_at`, `source_ref`, `pr_url`, `dependency_ids[]`, `version`, `created_at`, `updated_at`, `closed_at`, `deleted_at` | `work-items` DynamoDB table; issue number is permanent and never reused |
| ActivityEvent | `event_id`, `public_id`, `project_id`, `actor_type`, `actor_id`, `event_type`, `comment`, `changes`, `created_at` | `activity` DynamoDB table; append-only |
| IdempotencyRecord | `client_id`, `idempotency_key`, `request_hash`, `resource_type`, `resource_id`, `response_status`, `response_body`, `created_at`, `expires_at` | `idempotency` DynamoDB table with TTL |
| AuthPrincipal | `principal_type`, `principal_id`, `owner_id`, `scopes`, `client_id` or Cognito `sub` | Cognito claims plus SAM configuration; no local user table in v1 |

## DynamoDB Join Strategy

The Lambda layer performs application-level joins using stable identifiers; no relational database is required.

1. Read the project by `owner_id + project_key` from `projects`.
2. Query `work-items` by `project_id`; group the returned items by `status` for the board response.
3. Return `public_id` values in the form `<PROJECT_KEY>-<ISSUE_NUMBER>`, such as `AFT-134`.
4. For item detail, query `activity` by `public_id` and order events by timestamp.
5. Store dependency references as public IDs. Resolve dependency summaries with a bounded `BatchGetItem` against `work-items` only when the detail view requests them.
6. Check `idempotency` before processing a create or other mutation that supplies an idempotency key; return the saved response when the request hash matches.

The initial workload is approximately 100 items, so a project board query can retrieve all active items for the project and group them in Lambda. A status GSI may be added later if the workload grows; it is not required for v1.

## REST API

Base path: `/v1`

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/v1/projects` | List owner projects |
| `POST` | `/v1/projects` | Create a project and reserve its immutable project key |
| `GET` | `/v1/projects/{projectKey}/board` | Read all active items grouped by status; supports `ETag`/conditional reads |
| `POST` | `/v1/projects/{projectKey}/items` | Create an item; accepts `Idempotency-Key` |
| `GET` | `/v1/projects/{projectKey}/items/{issueNumber}` | Read item detail and summary metadata |
| `PATCH` | `/v1/projects/{projectKey}/items/{issueNumber}` | Update fields or move to any status; requires expected version for mutation safety |
| `POST` | `/v1/projects/{projectKey}/items/{issueNumber}/claim` | Exclusively claim an item; returns `409` if already claimed |
| `POST` | `/v1/projects/{projectKey}/items/{issueNumber}/comments` | Append a comment and activity event |
| `GET` | `/v1/projects/{projectKey}/items/{issueNumber}/activity` | Read append-only activity and comments |
| `POST` | `/v1/projects/{projectKey}/items/{issueNumber}/close` | Close an item without requiring a PR |
| `POST` | `/v1/projects/{projectKey}/items/{issueNumber}/reopen` | Reopen an item, including from Done |
| `DELETE` | `/v1/projects/{projectKey}/items/{issueNumber}` | Soft-delete an item while retaining its number tombstone |

## Integrations

| Name | Direction | Protocol | Auth |
|---|---|---|---|
| Amplify Web App → Existing Cognito User Pool | inbound to app | OAuth 2.0 / OIDC | Google identity provider through Cognito |
| Amplify Web App → API Gateway HTTP API | inbound | HTTPS/JSON REST | Cognito owner JWT; CORS restricted to `app.agent-queue.nerdthoughts.net` and local dev origin |
| ECS Agent Task → Cognito token endpoint | inbound to Cognito | HTTPS `application/x-www-form-urlencoded` | M2M app client ID/secret from Secrets Manager |
| ECS Agent Task → API Gateway HTTP API | inbound | HTTPS/JSON REST | Cognito access token with `agent:read` or `agent:write` |
| API Gateway HTTP API → Work Item Lambda | inbound | AWS service integration | IAM service invocation |
| Work Item Lambda → DynamoDB tables | outbound | AWS SDK | Lambda execution role with least-privilege table access |
| ECS Task Role → Secrets Manager | outbound | AWS SDK | IAM task role scoped to the M2M secret ARN |
| Amplify Hosting → Route 53 | inbound DNS | DNS/HTTPS | Amplify-managed certificate and Route 53 records |

## Runtime

- **AWS account:** Steve’s personal AWS account.
- **Backend deployment:** AWS SAM, with separate `dev` and `prod` stacks.
- **API:** API Gateway HTTP API with Cognito JWT authorization and Lambda proxy integration.
- **Lambda:** Python 3.14, short-lived request handlers, no local state.
- **Persistence:** Four DynamoDB on-demand tables with TTL on idempotency records.
- **Frontend:** Amplify Hosting, using the existing Slack 2nd Brain Cognito User Pool and Google IdP with a tracker-specific app client.
- **Custom domain:** `app.agent-queue.nerdthoughts.net` for the website. The API uses its API Gateway URL in v1 and is supplied to Amplify as an environment variable; a custom `api.` hostname can be added later.
- **Agent runtime:** Existing ECS tasks; the tracker does not run an always-on ECS service.
- **Live updates:** Browser polls the board endpoint every three seconds with a conditional request.

## Non-functional Notes

- **Auth:** Cognito handles web owner login and ECS `client_credentials` access tokens. HTTP API validates JWTs; Lambda enforces `agent:read` and `agent:write` scopes.
- **Secrets:** Cognito M2M client secret is in Secrets Manager and read by ECS task role at runtime. No secret is baked into images, source, Amplify frontend code, or logs.
- **Persistence:** DynamoDB on-demand; immutable project keys and issue numbers; soft deletion preserves number history.
- **Concurrency:** DynamoDB conditional writes on `version`, claim state, and issue counter. Conflicts return HTTP `409`.
- **Idempotency:** `Idempotency-Key` plus request hash prevents duplicate creates and repeated side effects; records expire with DynamoDB TTL.
- **Availability:** No GitHub dependency for board reads or writes. AWS-managed services provide the runtime availability target.
- **Cost:** HTTP API, Lambda, DynamoDB on-demand, Amplify Hosting, and Cognito are pay-per-use. ECS runtime cost is external to the tracker and will dominate if agents run continuously.
- **Observability:** Structured CloudWatch logs with request ID, endpoint, principal type, project key, and public item ID; never log bearer tokens or client secrets.
- **CORS:** Allow only the Amplify production domain and explicitly configured localhost development origin.
