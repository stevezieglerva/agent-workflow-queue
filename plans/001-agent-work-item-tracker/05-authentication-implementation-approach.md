# Authentication Implementation Approach: Automated Tests and ECS Agents

## Decision

Use Cognito OAuth 2.0 `client_credentials` for non-interactive automated tests in MVP and for ECS-hosted coding agents after MVP.

The dev E2E runner reads the dedicated test app-client credentials from `.env.test` or CI secret storage. ECS-specific delivery through Secrets Manager and the ECS task role is deferred until containers are part of the deployed agent workflow.

## Request Flow

```text
Dev E2E runner or future ECS agent
  │  1. Read Cognito client ID and secret
  │     from .env.test/CI now, or Secrets Manager later
  ▼
Cognito token endpoint
  │  2. Validate client ID and secret
  │  3. Return short-lived access token
  ▼
Dev E2E runner or future ECS agent
  │  4. Cache token in memory until near expiry
  │  5. Call REST API with Authorization: Bearer <access-token>
  ▼
API Gateway HTTP API
  │  6. Cognito JWT authorizer validates issuer, client, expiry, and scopes
  ▼
Work-item service
```

The frontend uses the same REST endpoints with an owner Cognito JWT. No separate frontend API or agent API is required.

## MVP Credential Handling

- Create a dedicated dev/test Cognito app client with a client secret and `agent:read` / `agent:write` scopes.
- Store local test credentials in an ignored `.env.test` file; store CI values in the CI secret store.
- Never commit `.env.test`, print the client secret, or place it in the frontend bundle.
- Keep access tokens only in test-process memory; do not write them to logs, files, databases, or test reports.
- Refresh the access token before expiry and retry one time after a 401 response.
- Use this client-credentials flow for real Behave E2E tests against the deployed dev stack.

## Post-MVP ECS Credential Handling

- Store the Cognito app client secret in AWS Secrets Manager.
- Give each ECS task definition an IAM task role that can read only the required secret and decrypt it if a customer-managed KMS key is used.
- Retrieve the secret at task startup through ECS secret injection or the AWS SDK.
- Keep the container image free of client secrets, AWS credentials, and access tokens.
- Rotate the client secret without rebuilding the image.

## Cognito Configuration

- Reuse the Slack 2nd Brain Cognito User Pool and Google identity provider for the web owner.
- Create a tracker-specific web app client with production and localhost callback/logout URLs.
- Create a dedicated M2M app client with a client secret and enable the client-credentials grant.
- Configure resource-server scopes `agent:read` and `agent:write`.
- Allow only the scopes required by the test/client app.
- Use short-lived access tokens; agents and tests do not need refresh tokens.

## API Authorization

- Human web requests use Cognito user authentication and owner JWTs.
- Automated test and future ECS requests use Cognito access tokens issued through `client_credentials`.
- API Gateway HTTP API validates the token before invoking the Work Item Lambda.
- The application enforces scope rules:
  - `agent:read` permits reading projects and work items.
  - `agent:write` permits creating, claiming, editing, moving, commenting on, linking, closing, and reopening work items.
- Do not add per-human collaboration permissions in v1; valid clients operate within the owner’s account.
- Start with one shared dev/test machine client. Split clients by agent type or project only when individual revocation or audit detail becomes valuable.

## Example MVP Test Behavior

```text
1. Load COGNITO_CLIENT_ID, COGNITO_CLIENT_SECRET, and COGNITO_TOKEN_URL from .env.test.
2. POST grant_type=client_credentials to the Cognito token URL.
3. Cache the returned access_token and expires_in value in memory.
4. Call real /v1 endpoints in the deployed dev stack with Authorization: Bearer <access_token>.
5. Refresh before expiry; never open a browser.
```

## Acceptance Checks

- The real dev E2E runner can obtain a Cognito access token through `client_credentials`.
- A valid token with `agent:read` can read a project board.
- A valid token with `agent:write` can create and move a work item.
- A read-only token cannot mutate a work item.
- An expired or invalid token receives HTTP `401`.
- The client secret is absent from source control, frontend assets, and test reports.
- Token values do not appear in application or test logs.
- After MVP, an ECS task can use the same flow with Secrets Manager/task-role delivery.
