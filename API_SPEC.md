# ChatGPT Auth API Surface (Codex)

## Scope and intent

This document is a repo‑grounded description of the HTTP API surface used when Codex is authenticated via **ChatGPT** (not API key auth).
It is intentionally limited to what the code in this repository proves and avoids speculation about undocumented server behavior.
Every term of art is defined in plain language, and every endpoint listed here appears in the code paths that run under ChatGPT auth.

## Key definitions (plain language)

**ChatGPT auth** means the user signs in with ChatGPT credentials (via `codex login`), not an OpenAI API key.
In this mode, requests use a **ChatGPT backend base URL** and include ChatGPT‑specific headers like `ChatGPT-Account-Id` and a bearer token.
The base URL defaults to `https://chatgpt.com/backend-api/`, which is a ChatGPT backend host rather than the public OpenAI API host.【F:codex-rs/core/src/config/mod.rs†L339-L341】【F:codex-rs/core/src/config/mod.rs†L1582-L1585】

**WHAM** is the path namespace used by the ChatGPT backend for Codex tasks, usage, and environment discovery.
The code chooses WHAM whenever the base URL contains `/backend-api`, and then constructs paths like `/wham/tasks` or `/wham/usage` under that base URL.
There is no in‑repo expansion of the acronym; it is simply the path prefix that signals the ChatGPT backend API surface in these clients.【F:codex-rs/backend-client/src/client.rs†L22-L37】【F:codex-rs/backend-client/src/client.rs†L161-L253】

**Task** is a unit of work (a Codex “job” or conversation turn) created by a `POST /wham/tasks` call and then retrieved or listed via other task endpoints.
Tasks have identifiers, can be listed, and may include multiple turns; sibling turns can be fetched for a task’s turn, implying a task can have multiple attempt branches.【F:codex-rs/backend-client/src/client.rs†L172-L279】

**Environment** is a backend resource that represents a Codex execution workspace or context.
The code fetches environments from the ChatGPT backend to select where tasks should run, with repo‑specific lookups and a fallback to listing all environments.
Environments are described by fields such as `id`, `label`, `is_pinned`, and `task_count` in the response payloads the client expects.【F:codex-rs/cloud-tasks/src/env_detect.rs†L6-L107】

**Responses API** and **Chat Completions API** are two OpenAI‑compatible wire protocols.
Codex requires a provider to declare which wire API it expects.
In ChatGPT auth mode, the provider base URL switches to a ChatGPT backend host, but the same wire API concepts still apply: `Responses` corresponds to `/v1/responses` and `Chat` corresponds to `/v1/chat/completions`.
The repo defaults OpenAI providers to the Responses wire API, with Chat as the legacy fallback.【F:codex-rs/core/src/model_provider_info.rs†L34-L52】【F:codex-rs/core/src/model_provider_info.rs†L136-L166】

## Authentication APIs (ChatGPT login)

These endpoints are used to obtain or refresh ChatGPT credentials that then authorize calls to the ChatGPT backend APIs described later in this document.
They are part of the login flow in `codex login`, not part of task execution or model inference.

### OAuth authorize and token exchange (browser flow)

- `GET {issuer}/oauth/authorize?...`
  - Built by the login server to start the OAuth authorization‑code flow.
  - Includes parameters like `client_id`, `redirect_uri`, PKCE values, and optional `allowed_workspace_id` for forced workspace restrictions.【F:codex-rs/login/src/server.rs†L380-L418】

- `POST {issuer}/oauth/token`
  - Exchanges an authorization code for tokens (`id_token`, `access_token`, `refresh_token`) using `application/x-www-form-urlencoded` body data.
  - Also used for token exchange to obtain an API‑key access token (grant type `urn:ietf:params:oauth:grant-type:token-exchange`).【F:codex-rs/login/src/server.rs†L510-L555】【F:codex-rs/login/src/server.rs†L700-L743】

### Device code login (ChatGPT device flow)

- `POST {issuer}/api/accounts/deviceauth/usercode`
  - Requests a device code and polling interval for device‑code login.【F:codex-rs/login/src/device_code_auth.rs†L55-L97】

- `POST {issuer}/api/accounts/deviceauth/token`
  - Polls the device authorization endpoint until an authorization code is issued or the flow times out.【F:codex-rs/login/src/device_code_auth.rs†L99-L149】

- `GET {issuer}/codex/device`
  - Verification URL shown to the user for entering the device code in a browser.【F:codex-rs/login/src/device_code_auth.rs†L167-L176】

- `GET {issuer}/deviceauth/callback`
  - Redirect URL used after the device code flow to exchange an authorization code for tokens.【F:codex-rs/login/src/device_code_auth.rs†L195-L214】

### Local callback endpoints (login server)

These are local HTTP endpoints exposed by the CLI on `localhost` during login.
They are not ChatGPT backend endpoints, but they are part of the end‑to‑end auth flow.

- `GET http://localhost:{port}/auth/callback`
  - Receives the OAuth authorization code and state from the browser flow and completes login.【F:codex-rs/login/src/server.rs†L112-L180】

- `GET http://localhost:{port}/cancel`
  - Used to signal a previously running login server to cancel if the port is already in use.【F:codex-rs/login/src/server.rs†L440-L466】

## ChatGPT base URLs and normalization

### Base URL configuration

- The **ChatGPT base URL** is stored in `Config.chatgpt_base_url` and defaults to `https://chatgpt.com/backend-api/`.【F:codex-rs/core/src/config/mod.rs†L339-L341】【F:codex-rs/core/src/config/mod.rs†L1582-L1585】
- When the auth mode is ChatGPT, the model provider base URL defaults to `https://chatgpt.com/backend-api/codex`, which is the root for OpenAI‑compatible model inference in ChatGPT auth mode (Responses or Chat Completions).【F:codex-rs/core/src/model_provider_info.rs†L136-L166】

### Normalization rules

When a base URL is created for backend requests, the client normalizes it so that ChatGPT hostnames always include `/backend-api`.
In practice this means `https://chatgpt.com` or `https://chat.openai.com` will be rewritten to `https://chatgpt.com/backend-api` (or the chat.openai.com equivalent) before request paths are appended.
This normalization is implemented both in the backend client and in cloud‑tasks utilities.【F:codex-rs/backend-client/src/client.rs†L50-L66】【F:codex-rs/cloud-tasks/src/util.rs†L14-L42】

## Headers and authentication in ChatGPT auth mode

ChatGPT‑auth requests typically include:

- `Authorization: Bearer <token>` (where the token comes from ChatGPT auth state)
- `ChatGPT-Account-Id: <account_id>` (derived from the auth record or extracted from the JWT when present)
- `User-Agent: <codex-cli>` (set to the Codex user agent)

The backend client and cloud‑tasks helpers explicitly construct these headers, and ChatGPT‑specific requests also add `Content-Type: application/json` where needed (e.g., task creation).【F:codex-rs/backend-client/src/client.rs†L109-L129】【F:codex-rs/cloud-tasks/src/util.rs†L72-L106】【F:codex-rs/chatgpt/src/chatgpt_client.rs†L30-L35】

## ChatGPT backend (WHAM) endpoints

All endpoints in this section are chosen when the base URL contains `/backend-api`, which is the default for ChatGPT auth.
The `PathStyle::ChatGptApi` branch in the backend client maps every task and usage endpoint into the WHAM path namespace.【F:codex-rs/backend-client/src/client.rs†L22-L37】【F:codex-rs/backend-client/src/client.rs†L161-L253】

### Usage and rate limits

- `GET {base_url}/wham/usage`
  - Used to fetch usage and rate limit snapshots for the signed‑in ChatGPT account.
  - The response is decoded into a rate‑limit snapshot that includes primary/secondary windows and credit details.【F:codex-rs/backend-client/src/client.rs†L161-L170】【F:codex-rs/backend-client/src/client.rs†L282-L303】

### Tasks

- `GET {base_url}/wham/tasks/list`
  - Lists tasks with optional query parameters: `limit`, `task_filter`, `cursor`, and `environment_id`.
  - The client expects a paginated response (`items` plus `cursor`).【F:codex-rs/backend-client/src/client.rs†L172-L205】

- `GET {base_url}/wham/tasks/{task_id}`
  - Retrieves task details for a specific task id.
    Used both in the backend client and the ChatGPT‑specific `get_task` flow that fetches task data directly from the ChatGPT backend API.
  - For the chatgpt crate, this is used to retrieve diff output items from the task’s current diff turn.【F:codex-rs/backend-client/src/client.rs†L213-L223】【F:codex-rs/chatgpt/src/get_task.rs†L37-L39】

- `POST {base_url}/wham/tasks`
  - Creates a new task (a new user turn). The client expects the response to include a task id either under `task.id` or top‑level `id`.
  - Requests are JSON (`Content-Type: application/json`).【F:codex-rs/backend-client/src/client.rs†L247-L279】

- `GET {base_url}/wham/tasks/{task_id}/turns/{turn_id}/sibling_turns`
  - Fetches sibling turns for a specific task turn, indicating multiple attempts or branches per turn.
  - The response is decoded into a `TurnAttemptsSiblingTurnsResponse`.【F:codex-rs/backend-client/src/client.rs†L227-L244】

### Environments

Environment endpoints are used to discover or list execution environments, with repo‑specific matching or fallback to the full list.

- `GET {base_url}/wham/environments/by-repo/{provider}/{owner}/{repo}`
  - Used to find environments that match a given repository (for example, a GitHub origin). The client builds this URL only when the base URL is a ChatGPT backend URL.

- `GET {base_url}/wham/environments`
  - Lists all environments as a fallback when repo‑specific environments do not resolve.

The environment responses are expected to be arrays of objects containing `id`, and optionally `label`, `is_pinned`, and `task_count`.
These fields are used to select an environment with preference for label matches, single results, pinned environments, and highest `task_count` when multiple are returned.【F:codex-rs/cloud-tasks/src/env_detect.rs†L6-L107】【F:codex-rs/cloud-tasks/src/env_detect.rs†L109-L176】

## ChatGPT task URLs for humans

The CLI builds browser‑friendly URLs for tasks that point to the ChatGPT web UI.
When the base URL is a ChatGPT backend host, it converts `https://chatgpt.com/backend-api` into `https://chatgpt.com/codex/tasks/{task_id}` for display to the user.
This is not an API endpoint, but it is directly derived from the ChatGPT backend base URL and reflects how tasks are viewed in the UI.【F:codex-rs/cloud-tasks/src/util.rs†L107-L121】

## ChatGPT inference endpoints at `/backend-api/codex`

When ChatGPT auth is active, the model provider default base URL switches to `https://chatgpt.com/backend-api/codex`.
This indicates that the OpenAI‑compatible inference endpoints are mounted under the ChatGPT backend host.
The repo does not define the full server‑side API, but it does define the wire protocol expectations:

- **Responses API** (`/v1/responses`) is the default wire protocol for OpenAI and ChatGPT providers.
- **Chat Completions API** (`/v1/chat/completions`) is the legacy wire protocol.

In practice, Codex builds request paths relative to the provider base URL.
For the default ChatGPT provider, this means the following paths are used under `https://chatgpt.com/backend-api/codex`:

- `POST /responses`
  - The core Responses API endpoint used for streaming and non‑streaming model output.
  - The streaming client selects `responses` when the provider’s wire API is Responses.【F:codex-rs/codex-api/src/endpoint/chat.rs†L52-L74】

- `POST /chat/completions`
  - Used when the provider’s wire API is Chat, maintaining compatibility with the legacy Chat Completions format.【F:codex-rs/codex-api/src/endpoint/chat.rs†L52-L74】

- `POST /responses/compact`
  - Used to compact response history for providers supporting the Responses wire API (or explicit Compact wire API).【F:codex-rs/codex-api/src/endpoint/compact.rs†L24-L40】【F:codex-rs/codex-api/src/endpoint/compact.rs†L42-L74】

- `GET /models?client_version=...`
  - Used to retrieve the available model list and optional ETag for caching.
  - The path is `models` relative to the provider base URL, with `client_version` appended as a query parameter.【F:codex-rs/codex-api/src/endpoint/models.rs†L33-L61】【F:codex-rs/codex-api/src/endpoint/models.rs†L63-L87】

The provider definition requires you to specify which wire API is used, and Codex cannot auto‑detect it.
That distinction is documented in the provider metadata and is explicitly referenced in comments and enums tied to these endpoints.【F:codex-rs/core/src/model_provider_info.rs†L34-L52】【F:codex-rs/core/src/model_provider_info.rs†L136-L166】

## What we cannot confirm from this repo

- There is no in‑repo description of what “WHAM” stands for, only its usage as a ChatGPT backend path prefix.
- The exact schemas for WHAM endpoints are partially inferred (e.g., `environment` fields, `task.id`), but the full server contract is not defined in this repository.
  We only list fields and behavior that the client expects or parses.

## How to verify in code

From the repo root:

- Find the ChatGPT base URL defaults:
  - Search for `chatgpt_base_url` in `codex-rs/core/src/config/mod.rs`.
- Confirm ChatGPT provider base URL and wire APIs:
  - Open `codex-rs/core/src/model_provider_info.rs` and read the `WireApi` enum and the `AuthMode::ChatGPT` default base URL.
- Confirm WHAM endpoints:
  - Search for `/wham/` in `codex-rs/backend-client/src/client.rs` and `codex-rs/cloud-tasks/src/env_detect.rs`.
- Confirm ChatGPT‑specific headers:
  - Inspect header construction in `codex-rs/backend-client/src/client.rs` and `codex-rs/cloud-tasks/src/util.rs`.
- Confirm direct ChatGPT task fetches in the `codex-chatgpt` crate:
  - Inspect `codex-rs/chatgpt/src/get_task.rs` and `codex-rs/chatgpt/src/chatgpt_client.rs`.

## Web research note

Web research was attempted, but the environment blocked outbound web search requests with HTTP 403 “CONNECT tunnel failed” responses when querying public search engines.
A simple unauthenticated probe of `https://chatgpt.com/backend-api/` also returned HTTP 403.
As a result, this spec is based entirely on the repository code as the source of truth.
