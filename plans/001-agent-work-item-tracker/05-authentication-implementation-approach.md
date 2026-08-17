# Authentication Implementation Approach: ECS Agents

## Decision

Use the Cognito OAuth 2.0 `client_credentials` grant for machine-to-machine authentication from ECS-hosted coding agents and personal AI assistants to the Agent Work Item Tracker REST API.

This keeps the API Cognito-authenticated without requiring an agent container to perform an interactive human login.

## Request Flow

```text
ECS agent
  │  1. Read Cognito client credentials at runtime
  │     from AWS Secrets Manager using its ECS task role
  ▼
Cognito token endpoint
  │  2. Validate client ID and secret
  │  3. Return short-lived access token
  ▼
ECS agent
  │  4. Cache token in memory until near expiry
  │  5. Call REST API with Authorization: Bearer <access-token>
  ▼
API Gateway / REST API
  │  6. Cognito JWT authorizer validates issuer, audience, expiry, and scopes
  ▼
Work-item service
```

## Credential Handling

- The container image contains no client secret, API token, or AWS credential.
- Store the Cognito app client ID and secret in AWS Secrets Manager.
- Give each ECS task definition an IAM task role that can read only the required secret and decrypt it if a customer-managed KMS key is used.
- Retrieve the secret at task startup through ECS secret injection or the AWS SDK. Prefer runtime retrieval over storing secrets in source control or the image filesystem.
- Keep access tokens only in process memory; do not write them to logs, files, databases, or task metadata.
- Refresh the access token before expiry and retry one time after a 401 response.
- Rotate the Cognito app client secret and update Secrets Manager without rebuilding the image.

## Cognito Configuration

- Create a Cognito User Pool and domain for the tracker.
- Configure a resource server with scopes such as `agent:read` and `agent:write`.
- Create an OAuth app client with a client secret and enable the client-credentials grant.
- Allow only the scopes required by the app client.
- Use short-lived access tokens; agents do not need refresh tokens for this flow.
- Use the Cognito access-token `client_id` and scope claims for service-level attribution and authorization.

## API Authorization

- Human web requests use Cognito user authentication and user JWTs.
- ECS agent requests use Cognito access tokens issued through `client_credentials`.
- API Gateway or the API edge validates the token before the application handler runs.
- The application enforces scope rules:
  - `agent:read` permits reading projects and work items.
  - `agent:write` permits creating, claiming, editing, moving, commenting on, linking, closing, and reopening work items.
- Do not add per-human collaboration permissions in v1; all valid clients operate within the owner’s account.
- Start with one shared agent app client for implementation simplicity. Split into separate clients by agent type or project later if individual revocation or audit detail becomes important.

## Example Agent Behavior

```text
1. Start ECS task.
2. Read COGNITO_CLIENT_ID, COGNITO_CLIENT_SECRET, and COGNITO_TOKEN_URL from runtime configuration.
3. POST grant_type=client_credentials to the Cognito token URL.
4. Cache the returned access_token and expires_in value in memory.
5. Call REST endpoints with Authorization: Bearer <access_token>.
6. Refresh before expiry; never ask a human to log in.
7. Stop the task and discard the in-memory token.
```

## V1 Boundaries

- No long-lived API keys embedded in images or source code.
- No username/password login from agents.
- No Cognito refresh-token storage for ECS agents.
- No per-agent user accounts or fine-grained project permissions initially.
- No custom token issuer; Cognito remains the identity authority.

## Acceptance Checks

- A newly started ECS task can obtain a Cognito access token using only its task role and Secrets Manager secret.
- A valid token with `agent:read` can read a project board.
- A valid token with `agent:write` can create and move a work item.
- A read-only token cannot mutate a work item.
- An expired or invalid token receives HTTP 401 from the API.
- The client secret is absent from the built container image and repository.
- Token values do not appear in application logs.
