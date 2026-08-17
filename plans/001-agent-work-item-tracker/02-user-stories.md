# User Stories: Agent Work Item Tracker

## US-001 — Agent Authenticates and Reads Work
**As a** coding agent or personal AI assistant
**I want** to authenticate through the Cognito OAuth 2.0 `client_credentials` flow and read project work items
**So that** I can operate headlessly from an ECS container without browser automation or human login

**Priority:** P0
**Acceptance Criteria:**
- An ECS agent can obtain a short-lived Cognito access token using runtime credentials from AWS Secrets Manager.
- A token with the `agent:read` scope can list projects and read a project board through the REST API.
- An expired or invalid token receives HTTP `401`.
- A valid token without the required scope receives HTTP `403`.

## US-002 — Agent Creates a Work Item
**As a** coding agent or personal AI assistant
**I want** to create a work item with context and optional metadata
**So that** new work enters the correct project queue without human data entry

**Priority:** P0
**Acceptance Criteria:**
- An authenticated agent can create an item with a title, description, and project key.
- The service assigns the next sequential numeric issue number and returns a Jira-style identifier such as `AFT-134`.
- The item may include priority, labels, assignee, optional source reference, optional PR URL, and dependency links.
- A repeated request with the same idempotency key returns the original item instead of creating a duplicate.
- An invalid project key or malformed request returns HTTP `400` or `404` with a useful error body.

## US-003 — Agent Claims Work Exclusively
**As a** coding agent
**I want** to claim an unassigned work item
**So that** multiple agents do not perform the same work simultaneously

**Priority:** P0
**Acceptance Criteria:**
- The first valid claim records the agent identity, claim timestamp, and assignee on the item.
- A concurrent claim for an already claimed item fails with HTTP `409 Conflict`.
- A successful claim can optionally move the item to `In Progress` in the same operation.
- The claim response includes the current item version or timestamp needed for safe subsequent updates.

## US-004 — Agent Advances and Completes Work
**As a** coding agent or personal AI assistant
**I want** to update, move, comment on, link, close, and reopen work items
**So that** the tracker records the full lifecycle of agent activity

**Priority:** P0
**Acceptance Criteria:**
- An agent can move an item between any workflow columns, including reopening a Done item.
- An agent can update the title, description, priority, labels, assignee, optional PR URL, and dependency links.
- An agent can append a comment and the system records the activity event with actor and timestamp.
- Comments and system activity are append-only in v1.
- An agent can close or reopen an item without requiring a GitHub issue or PR.
- A stale update is rejected with HTTP `409 Conflict` rather than silently overwriting newer data.

## US-005 — Human Watches a Live Board
**As a** project owner
**I want** to see agent changes appear on the board automatically
**So that** I can monitor work without refreshing or polling manually

**Priority:** P0
**Acceptance Criteria:**
- The owner can open a project board and see all six workflow columns and their current counts.
- A move, claim, edit, comment, or completion made through the REST API appears in the open board within five seconds.
- The board can switch between projects and display each project’s Jira-style item identifiers.
- Local drag/drop updates optimistically and rolls back with an error message if the REST API rejects the change.
- The board remains usable with one human user and approximately ten projects with ten items each.

## US-006 — Human Manages Items from the Website
**As a** project owner
**I want** to create, edit, filter, move, and inspect work items in the website
**So that** I can intervene when an agent needs clarification or when work is not agent-driven

**Priority:** P1
**Acceptance Criteria:**
- The owner can create an item in a selected project and workflow column.
- The owner can edit item details and move an item to any column.
- The owner can search by Jira-style identifier, title, description, label, or assignee.
- The owner can filter to priority items or items assigned to the owner.
- The item detail view shows comments, system activity, timestamps, optional source reference, and optional PR URL.

## US-007 — Human Provisions Agent Access
**As a** project owner
**I want** to manage the Cognito app client and agent secret lifecycle
**So that** ECS agents can authenticate without exposing credentials in container images

**Priority:** P1
**Acceptance Criteria:**
- The deployment creates or documents the Cognito resource server scopes and client-credentials app client.
- The client secret is stored in AWS Secrets Manager and is not present in the repository or built image.
- The ECS task role can read only the required secret.
- Rotating or revoking the client secret prevents future token acquisition without changing the container image.
