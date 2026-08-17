# Product Brief: Agent Work Item Tracker

## Problem
GitHub Projects is useful for organizing work, but GitHub outages interrupt the primary workflow for coordinating coding agents and personal AI assistants. A small, personally operated tracker is needed as a dependable, low-cost fallback that agents can update headlessly and that provides a live web board.

## Users
- **Owner** — The single human user who creates projects, reviews work, and monitors all boards.
- **AI assistants and coding agents** — Automated clients that create, claim, update, move, comment on, and complete work items through an authenticated REST API.

## Value
The tracker provides a GitHub-Projects-like board that remains available independently of GitHub, while staying cheap and simple enough for one person to maintain. Agents can use stable REST endpoints and human-friendly item numbers, and the web board reflects changes within a few seconds.

## Scope — In
- Multiple personal projects, with approximately ten projects and ten items per project as the initial operating target.
- A shared six-column workflow: Todo, Elaborate, Ready For Agent, In Progress, Reviewing, and Done.
- Work items with a Jira-style project key plus sequential number such as `AFT-134`, title, description, status, optional priority, labels, assignee/agent, optional external source reference, optional PR URL, timestamps, comments, and dependency links.
- REST API operations for create, read, list, filter, claim, update, move, comment, link a PR, close/reopen, and delete where appropriate.
- Cognito-authenticated web access for the owner.
- Headless client access through Cognito OAuth 2.0 `client_credentials`; dev E2E tests use a dedicated test app client, while ECS secret delivery is deferred until after MVP.
- A live board website with project switching, drag-and-drop movement, item creation/editing, filtering, item detail, and activity history.
- Serverless-first hosting and persistence with a target cost of a few dollars per month or less.

## Scope — Out
- Multi-user collaboration, invitations, teams, or per-agent authorization policies.
- GitHub Issues/Projects synchronization or `bd` beads integration; external issue and PR references remain optional links.
- Bulk update operations.
- Custom workflows or per-project column configuration in v1.
- Enterprise availability, audit/compliance features, high-volume scaling, or offline editing.
- Dependence on GitHub for core board availability.

## Success Signals
- Authenticated REST mutations appear on the live board within five seconds under normal conditions.
- The owner can manage at least ten projects with roughly ten items each without noticeable UI degradation.
- A coding agent can complete the core lifecycle—create, claim, update, move, comment, and close—using documented REST calls without browser automation.
- Monthly infrastructure cost remains within a few dollars for the expected personal workload.
- A small implementation can be deployed and maintained without introducing a dedicated server or always-on compute service.

## Primary Risk
Implementation time and effort could exceed the value of replacing GitHub Projects. Mitigation: keep v1 single-owner, fixed-workflow, REST-first, serverless, and deliberately omit integrations, custom workflows, bulk actions, and advanced permissions.
