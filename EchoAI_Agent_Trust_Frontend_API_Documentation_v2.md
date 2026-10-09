# EchoAI Agent Trust — Frontend API Contract

**Audience:** Frontend engineers implementing the Agent Health Center UI  
**Contract basis:** The Agent Trust backend package and the Agent Trust frontend API client inspected on 9 October 2026.  
**Purpose:** Define the screens, endpoints, request bodies, response examples, error handling, and integration rules needed to implement the accepted UI.

> **Important distinction:** Endpoint paths, request models, and response fields below are based on the supplied backend source. JSON examples are illustrative values matching those shapes, not production data. Where the current frontend client does not match the backend contract, this document explicitly identifies the required correction. Do not assume a frontend TypeScript interface changes the backend response.

---

## 1. Product and screen contract

Agent Trust is presented as **Agent Health Center**. Keep the primary navigation focused on three screens:

1. **Agents** — discover agents, trust state/score, review freshness, usage and cost.
2. **Agent health** — one agent's summary, findings/improvements, checks and evidence, policies, functional tests, identity/security, usage/cost and timeline. Tabs/sections inside this screen are not separate top-level products.
3. **Finding detail** — evidence, expected versus observed behavior, impact, recommendation and verification instructions.

The frontend must not edit an agent's underlying prompt/configuration from Agent Trust. Recommendations are suggestions. The user changes the agent in the normal Agents editor and then reruns the review.

### Trust fields

- `trust_score`: nullable numeric score from 0 to 100. A missing score is not zero.
- `trust_state`: `UNREVIEWED`, `TRUSTED`, `NEEDS_REVIEW`, or `HIGH_RISK`.
- `stale`: whether some results need to be refreshed after the agent definition or governance setup changes.
- Check statuses: `pass`, `partial`, `fail`, `not_configured`, `not_verified`.

The backend owns score calculation. The frontend renders the score and the supplied breakdown; it must not recalculate trust scores from check statuses.

---

## 2. Base URL, authentication and conventions

### Base URL

Use the platform's configured API base URL (`ENV.API_BASE_URL`). Agent Trust routes are under `/agent-trust`.

Example full route: `{API_BASE_URL}/agent-trust/agents`.

### Authentication

The current frontend client uses Axios with JSON headers and `withCredentials: true`. Preserve the application's existing authenticated session mechanism; do not introduce a second authentication scheme just for Agent Trust. Backend routes shown here require an authenticated EchoAI user with `require_admin_or_above` unless otherwise noted.

### IDs and timestamps

- `agent_id` may be supplied in the platform's accepted UUID representation; the backend normalizes it internally. Treat it as an opaque string in the frontend.
- Policy IDs and suite IDs are UUID-like strings; URL-encode path values when appropriate.
- Timestamps are ISO 8601 strings. Render them in the user's local timezone, but retain the original value in data/state.
- Monetary fields ending in `_usd` are USD.

### Scope and pagination

`scope=mine` is the default. `scope=all` is super-admin only. Agent list `limit` defaults to 50 and has a maximum of 200; `offset` defaults to 0.

### Error shape

FastAPI errors generally look like:

```json
{ "detail": "Agent not found" }
```

Validation errors (HTTP 422) may return `detail` as an array of objects with `loc`, `msg`, and `type`. The current frontend `extractApiError()` should display `detail`, `message`, or `default` where available.

| Status | Meaning / UI handling |
|---|---|
| `400` | Invalid ID, invalid query, or invalid request. Show a useful error. |
| `401` | Authentication expired or missing. Let the app's auth flow handle it. |
| `403` | User lacks permission; explain access is restricted. |
| `404` | Agent/policy/suite/review-run not found or not visible to this user. |
| `422` | Request body/query does not match the backend schema. Correct the payload; do not blindly retry. |
| `5xx` | Backend failure. Show a retry option and retain the correlation ID when available. |

---

## 3. Endpoint map for the accepted UI

| Screen / action | Method and path | Notes |
|---|---|---|
| Agents portfolio summary | `GET /agent-trust/dashboard` | Query `scope=mine` or authorized `all`. |
| Agent list | `GET /agent-trust/agents` | Search, state filter, pagination. |
| Agent health overview | `GET /agent-trust/agents/{agent_id}` | Score, checks, improvements, usage, where used. |
| Agent health detail | `GET /agent-trust/agents/{agent_id}/review` | Policies, suites, security and credentials. |
| Finding/check evidence | Usually from overview `checks[].detail` and `improvements[]` | Do not invent missing evidence. Use supplied detail fields. |
| Timeline | `GET /agent-trust/agents/{agent_id}/timeline` | Cursor-based activity history. |
| Usage & cost | `GET /agent-trust/agents/{agent_id}/usage` | `window=24h\|7d\|30d\|all`. |
| Start review | `POST /agent-trust/review` | Returns HTTP 202 and `correlation_id`. |
| Check review progress | `GET /agent-trust/reviews/{correlation_id}` | Poll manually from the Check Progress button. |
| Stream review progress (optional) | `GET /agent-trust/reviews/{correlation_id}/events` | Server-Sent Events; use fetch streaming if cookie/auth setup requires it. |
| Policy catalog | `GET /agent-trust/policies` | Administrative policy catalog. |
| Policies assigned to agent | `GET /agent-trust/agents/{agent_id}/policies` | Agent-centric view. |
| Policies assigned to agent (legacy/catalog route also present) | `GET /agent-trust/policies/agents/{agent_id}/policies` | Current frontend client uses this path; backend policies router defines it. Prefer one canonical path after confirming router mount/version. |
| Assign policy | `POST /agent-trust/policies/agents/{agent_id}/policies/{policy_id}` | No request body is required by the route. |
| Unassign policy | `DELETE /agent-trust/policies/agents/{agent_id}/policies/{policy_id}` | No request body. |
| Recommend policies | `POST /agent-trust/policies/agents/{agent_id}/policies/recommend` | **Send `{ "force_refresh": false }`**; body model is required. |
| Accept policy recommendation | `POST /agent-trust/policies/agents/{agent_id}/policies/recommend/{recommendation_id}/accept` | Explicit user confirmation only. |
| Create functional suite | `POST /agent-trust/agents/{agent_id}/functional-suites` | Saves user-approved tests. |
| Recommend functional suite | `POST /agent-trust/agents/{agent_id}/functional-suites/recommend` | **Send `{ "question_count": 1 }`** to honor the one-question design. |
| Accept AI functional recommendation | `POST /agent-trust/agents/{agent_id}/functional-suites/recommend/{recommendation_id}/accept` | Explicit user confirmation. |
| Delete functional suite | `DELETE /agent-trust/agents/{agent_id}/functional-suites/{suite_id}` | Refresh review/overview after success. |

The backend package may expose additional governance/testing routes via other routers, but they are not needed for the three primary screens unless the application explicitly mounts and uses them. Check the main application's router registration before integrating any additional `/testing`, `/cost`, `/audit`, or `/governance` route.

---

## 4. Agents screen

### 4.1 Dashboard summary

`GET /agent-trust/dashboard?scope=mine`

Example response:

```json
{
  "total_agents": 12,
  "average_trust_score": 78,
  "by_trust_state": {
    "UNREVIEWED": 2,
    "TRUSTED": 6,
    "NEEDS_REVIEW": 3,
    "HIGH_RISK": 1
  },
  "needs_attention": [
    {
      "agent_id": "a1b2c3d4e5f6478899aabbccddeeff00",
      "name": "Customer Support Agent",
      "trust_score": 58,
      "trust_state": "NEEDS_REVIEW",
      "reason": "Functional testing is incomplete"
    }
  ],
  "needs_attention_total": 1,
  "trend": [
    { "date": "2026-10-08", "average_trust_score": 74, "reviews": 4 },
    { "date": "2026-10-09", "average_trust_score": 78, "reviews": 3 }
  ]
}
```

All values above are examples. Render empty arrays/zero counts naturally. `average_trust_score` may be `null` when no scored agents exist.

### 4.2 Agent list

`GET /agent-trust/agents?scope=mine&limit=50&offset=0`

Optional query parameters:

- `scope`: `mine` (default) or `all` (super-admin only)
- `search`: free-text agent search
- `trust_state`: `UNREVIEWED`, `TRUSTED`, `NEEDS_REVIEW`, `HIGH_RISK`
- `limit`: 1–200, default 50
- `offset`: non-negative integer, default 0

Example response:

```json
{
  "total": 1,
  "limit": 50,
  "offset": 0,
  "items": [
    {
      "agent_id": "a1b2c3d4e5f6478899aabbccddeeff00",
      "name": "Customer Support Agent",
      "owner": { "user_id": "u123", "name": "Agent Owner", "email": "owner@example.com" },
      "environment": "production",
      "trust_score": 78,
      "trust_state": "TRUSTED",
      "last_reviewed_at": "2026-10-09T09:30:00Z",
      "stale": false,
      "total_runs": 1250,
      "total_tokens": 810000,
      "total_cost_usd": 21.4,
      "last_used_at": "2026-10-09T09:20:00Z",
      "updated_at": "2026-10-09T09:25:00Z"
    }
  ]
}
```

Note: `trust_score`, `last_reviewed_at`, `environment`, and some timestamps can be `null` depending on the model/data state. Do not treat a missing score as 0.

---

## 5. Agent health overview

### `GET /agent-trust/agents/{agent_id}`

Example response shape (values are illustrative):

```json
{
  "agent_id": "a1b2c3d4e5f6478899aabbccddeeff00",
  "name": "Customer Support Agent",
  "description": "Handles customer support requests",
  "owner": { "user_id": "u123", "name": "Agent Owner", "email": "owner@example.com" },
  "environment": "production",
  "tags": ["support"],
  "trust_score": 78,
  "trust_state": "TRUSTED",
  "last_reviewed_at": "2026-10-09T09:30:00Z",
  "stale": false,
  "created_at": "2026-09-10T11:00:00Z",
  "updated_at": "2026-10-09T09:25:00Z",
  "checks": [
    {
      "key": "identity",
      "label": "Identity",
      "status": "pass",
      "points_earned": 20,
      "points_possible": 20,
      "points_available": 0,
      "summary": "Identity checks passed",
      "fix": null,
      "detail": { "evidence": [] }
    },
    {
      "key": "functional",
      "label": "Functional testing",
      "status": "not_verified",
      "points_earned": 0,
      "points_possible": 20,
      "points_available": 20,
      "summary": "No verified functional result is available",
      "fix": "Run the functional test suite",
      "detail": {}
    }
  ],
  "improvements": [
    {
      "key": "functional_suite",
      "label": "Improve functional coverage",
      "points": 20,
      "status": "not_verified",
      "reason": "Functional behavior has not been verified",
      "action": "Run functional tests"
    }
  ],
  "open_alerts": 0,
  "usage": {
    "window": "all",
    "total_runs": 1250,
    "input_tokens": 610000,
    "output_tokens": 200000,
    "total_tokens": 810000,
    "total_cost_usd": 21.4,
    "cost_is_estimated": false,
    "last_used_at": "2026-10-09T09:20:00Z"
  },
  "used_in": {
    "workflows": [{ "workflow_id": "wf123", "name": "Customer Support", "status": "active" }],
    "apps": [{ "application_id": "app123", "link": "via_workflow" }]
  }
}
```

`checks[].detail` is flexible evidence data; its internal keys can vary by check. The frontend should render known fields where available and show a safe JSON/evidence fallback for unknown detail keys. The actual backend schema for `used_in.apps` uses `application_id` and `link`; it does not guarantee an app `name` field.

---

## 6. Agent health review workspace

### `GET /agent-trust/agents/{agent_id}/review`

Example response:

```json
{
  "agent_id": "a1b2c3d4e5f6478899aabbccddeeff00",
  "name": "Customer Support Agent",
  "trust_score": 78,
  "trust_state": "TRUSTED",
  "last_reviewed_at": "2026-10-09T09:30:00Z",
  "stale": false,
  "checks": [],
  "improvements": [],
  "policies": [
    {
      "policy_id": "policy-uuid",
      "name": "Privacy protection",
      "framework": "Internal",
      "severity": "high",
      "assigned_by": "human",
      "assigned_at": "2026-10-08T12:00:00Z"
    }
  ],
  "test_suites": [
    {
      "suite_id": "suite-uuid",
      "name": "Core functional test",
      "description": "One grounded question",
      "test_count": 1,
      "source": "ai",
      "last_run": {
        "status": "passed",
        "total": 1,
        "passed": 1,
        "failed": 0,
        "results": [],
        "at": "2026-10-09T09:28:00Z"
      }
    }
  ],
  "security": {
    "total_probes": 1,
    "blocked": 1,
    "last_scan_at": "2026-10-09T09:27:00Z",
    "probes": [
      { "attack_type": "privacy", "attack_name": "Unauthorized data request", "blocked": true, "verdict": "pass", "rationale": "No unauthorized data was returned" }
    ]
  },
  "credentials": { "evaluated_at": "2026-10-09T09:27:00Z", "declared": [], "drift": [] }
}
```

The frontend should display these sections inside Agent Health: **Summary, Findings, Checks & Evidence, Policies, Functional Tests, Identity/Security, Usage & Cost, Timeline, Review Workspace**. The accepted primary navigation still consists of Agents, Agent Health, and Finding Detail.

Backend suite field names are `test_count` and nested `last_run.total/passed/failed/at`. Do not expect `total_tests`, `passed_tests`, `failed_tests`, or `last_run_at` directly; normalize these in the API adapter if convenient.

---

## 7. Finding detail

The current backend does not expose a separate canonical `GET /findings/{finding_id}` endpoint in the supplied router package. The finding detail screen should be populated from `checks[]`, `improvements[]`, and their `detail`/evidence payloads returned by the overview and review endpoints, unless a dedicated findings route is added.

Suggested UI model derived in the frontend (not a direct backend schema):

```json
{
  "key": "functional",
  "title": "Functional testing is not verified",
  "status": "not_verified",
  "observed": "No verified functional test result is available.",
  "expected": "The agent should answer grounded questions accurately and acknowledge uncertainty.",
  "impact": "Reliability has not been demonstrated by a completed test.",
  "recommendation": "Generate and review one agent-specific functional test, then run it.",
  "verification_steps": ["Review the proposed test", "Run the suite", "Inspect the result and trace"],
  "evidence": [],
  "evidence_available": false
}
```

The strings above are display examples, not facts that the backend will return. Only populate `observed`, `expected`, impact, and evidence with claims supported by the actual returned detail. When evidence is missing, display **Not verified / Evidence unavailable**, not PASS.

---

## 8. Start review and check progress

### 8.1 Start review

`POST /agent-trust/review` — HTTP `202 Accepted`.

Request body:

```json
{ "agent_ids": ["a1b2c3d4e5f6478899aabbccddeeff00"] }
```

The list must contain 1–25 agent IDs.

Response:

```json
{
  "reviewed": 0,
  "failed": 0,
  "correlation_id": "d3d0e11d9c2a4b4b8b2a0b5d4c3f2a10",
  "results": [],
  "status": "started"
}
```

The immediate response means the request was accepted, not that the review has completed. Store the `correlation_id` and show the Check Progress action.

### 8.2 Check progress

`GET /agent-trust/reviews/{correlation_id}`

Example running response:

```json
{
  "correlation_id": "d3d0e11d9c2a4b4b8b2a0b5d4c3f2a10",
  "status": "running",
  "agent_ids": ["a1b2c3d4e5f6478899aabbccddeeff00"],
  "created_at": "2026-10-09T09:26:00Z",
  "updated_at": "2026-10-09T09:26:15Z",
  "latest_progress": {
    "id": 4,
    "event": "progress",
    "correlation_id": "d3d0e11d9c2a4b4b8b2a0b5d4c3f2a10",
    "timestamp": "2026-10-09T09:26:15Z",
    "stage": "functional",
    "status": "running",
    "message": "Running functional checks",
    "agent_id": "a1b2c3d4e5f6478899aabbccddeeff00"
  },
  "result": null,
  "error": null
}
```

Example completed response uses `status: "completed"`, a non-null `result`, and `error: null`. Failed runs use `status: "failed"` and may include an error. Use only actual backend progress; do not manufacture percentage completion.

Recommended client behavior:

1. Disable the review button while that agent has a review marked started/running.
2. Save the correlation ID from the `202` response.
3. `Check Progress` performs the GET status request.
4. If `status` is `started` or `running`, keep the review in progress and allow another manual check.
5. If `completed`, refresh overview, review workspace, usage and timeline.
6. If `failed`, show the error and keep the correlation ID available for troubleshooting.
7. If status returns `404`, explain that progress may no longer be available and offer to start a new review.

### 8.3 Optional SSE progress

`GET /agent-trust/reviews/{correlation_id}/events` returns `text/event-stream`. Event names include `review`, `progress`, `completed`, and `failed`. Each event's `data` is JSON containing an event ID, event name, correlation ID, timestamp and event-specific fields. `Last-Event-ID` can replay retained events while the same application process remains alive.

**Reliability limitation:** review status and event history are held in process memory. They are not durable across process restarts and are not shared across multiple workers/instances. For the accepted manual Check Progress UI, the status endpoint is sufficient within those limits.

---

## 9. Functional test suite

### 9.1 Generate a recommendation

`POST /agent-trust/agents/{agent_id}/functional-suites/recommend`

**A JSON request body is required.** For the agreed one-question experience, send:

```json
{ "question_count": 1 }
```

Allowed `question_count`: 1–5. Backend default is 3 if omitted from a valid body, but the frontend should explicitly send 1 for the agreed UI.

Response:

```json
{
  "recommendation_id": "recommendation-uuid",
  "agent_id": "a1b2c3d4e5f6478899aabbccddeeff00",
  "suggested_name": "Customer Support Core Test",
  "grounded_on": {
    "agent_name": ["Customer Support Agent"],
    "description": ["Handles customer support requests"],
    "connector_names": ["SupportKnowledge"],
    "tool_names": ["search_tickets"],
    "workflow_names": ["Customer Support"]
  },
  "test_cases": [
    {
      "name": "Handle insufficient information",
      "input": "Answer this request when the connected knowledge source has no relevant information.",
      "expected": "The agent acknowledges uncertainty instead of inventing unsupported facts.",
      "assertion_type": "semantic_judge",
      "keywords": [],
      "rationale": "Checks grounding and uncertainty handling.",
      "evidence_keys": ["description", "tool_names"]
    }
  ]
}
```

Display the generated test for review. Do not save it until the user accepts it or explicitly adds the suite.

### 9.2 Save a functional suite

`POST /agent-trust/agents/{agent_id}/functional-suites`

```json
{
  "name": "Customer Support Core Test",
  "description": "One grounded functional question",
  "source": "ai",
  "test_cases": [
    {
      "name": "Handle insufficient information",
      "input": "Answer this request when the connected knowledge source has no relevant information.",
      "expected": "The agent acknowledges uncertainty instead of inventing unsupported facts.",
      "assertion_type": "semantic_judge",
      "keywords": [],
      "rationale": "Checks grounding and uncertainty handling.",
      "evidence_keys": ["description", "tool_names"]
    }
  ]
}
```

Response:

```json
{
  "suite_id": "suite-uuid",
  "agent_id": "a1b2c3d4e5f6478899aabbccddeeff00",
  "name": "Customer Support Core Test",
  "description": "One grounded functional question",
  "test_count": 1,
  "source": "ai"
}
```

Each suite requires 1–50 test cases. Each case requires `name`, `input`, and `expected`. `assertion_type` is `semantic_judge` or `keyword_match`; `keywords` is used for keyword matching. `evidence_keys` is optional but useful for traceability.

### 9.3 Delete suite

`DELETE /agent-trust/agents/{agent_id}/functional-suites/{suite_id}`

Response:

```json
{ "status": "deleted", "suite_id": "suite-uuid" }
```

Refresh the review workspace and overview after successful create/delete.

---

## 10. Policies

### 10.1 List policy catalog

`GET /agent-trust/policies`

Returns a JSON array of policy objects from `Policy.to_dict()`. The persisted backend policy model supports fields such as `id`, `name`, `description`, `framework`, `severity`, `policy_prompt`, `judge_prompt`, `is_system_policy`, `is_active`, and `version` (plus timestamps if supplied by the model base). It does **not** define the frontend's proposed `category`, `rules`, or `enforcement_mode` as persisted model fields.

Illustrative response:

```json
[
  {
    "id": "policy-uuid",
    "name": "Privacy protection",
    "description": "Avoid unauthorized disclosure of private data.",
    "framework": "Internal",
    "severity": "high",
    "policy_prompt": "Check whether the agent discloses private data without authorization.",
    "judge_prompt": "Judge the tool trace and final answer for unauthorized disclosure.",
    "is_system_policy": false,
    "is_active": true,
    "version": 1
  }
]
```

Do not require `category`, `rules`, `enforcement_mode`, `created_at`, or `updated_at` unless the deployed backend actually returns them. Render absent optional fields gracefully.

### 10.2 Create/update/delete policy

`POST /agent-trust/policies`

```json
{
  "name": "Privacy protection",
  "description": "Avoid unauthorized disclosure of private data.",
  "framework": "Internal",
  "severity": "high",
  "policy_prompt": "Check whether the agent discloses private data without authorization.",
  "judge_prompt": "Judge the tool trace and final answer for unauthorized disclosure.",
  "is_system_policy": false,
  "is_active": true
}
```

`PATCH /agent-trust/policies/{policy_id}` accepts any subset of those editable fields. `DELETE /agent-trust/policies/{policy_id}` returns `{ "status": "deleted" }`.

### 10.3 Assign/unassign policy

`POST /agent-trust/policies/agents/{agent_id}/policies/{policy_id}` — no JSON body required.

Response:

```json
{
  "policy_id": "policy-uuid",
  "agent_id": "a1b2c3d4e5f6478899aabbccddeeff00",
  "assigned_by": "human",
  "status": "assigned"
}
```

`DELETE /agent-trust/policies/agents/{agent_id}/policies/{policy_id}` removes the assignment. The frontend currently treats its response as unused; refresh the assigned policy list after success.

### 10.4 Recommend/accept policies

`POST /agent-trust/policies/agents/{agent_id}/policies/recommend`

Request body is required:

```json
{ "force_refresh": false }
```

The response is generated by the recommendation service and includes a recommendation ID plus suggested policy IDs/details; use the actual returned object rather than assuming the frontend's desired shape. The frontend currently types this as `{ agent_id, recommendations: [{ policy_id, name, description, why }] }`; verify/adapt that mapping against the deployed response.

Accept a recommendation only after explicit user action:

`POST /agent-trust/policies/agents/{agent_id}/policies/recommend/{recommendation_id}/accept`

Example response:

```json
{
  "recommendation_id": "recommendation-uuid",
  "agent_id": "a1b2c3d4e5f6478899aabbccddeeff00",
  "accepted": true,
  "policy_ids": ["policy-uuid"]
}
```

**Policy UI schema alignment:** do not send `category`, `rules`, or `enforcement_mode` in policy create/update payloads; the backend uses `framework`, `severity`, prompts, and active/system flags. Keep Cost Governance out of the LLM-generated policy-prompt recommendation flow, as specified by the product requirements.

---

## 11. Timeline and usage/cost

### 11.1 Timeline

`GET /agent-trust/agents/{agent_id}/timeline?limit=50`

Optional query parameters: `category` (repeatable), `cursor` (sequence number), `limit` (1–200, default 50). By default, run events are excluded because they are high volume.

Response shape:

```json
{
  "entries": [
    {
      "seq": 128,
      "event": "agent.reviewed",
      "category": "governance",
      "title": "Agent reviewed",
      "at": "2026-10-09T09:30:00Z",
      "actor": "owner@example.com",
      "outcome": "success",
      "severity": "info",
      "details": { "previous_score": 68, "new_score": 78 },
      "hash": "abc123",
      "parent_hash": "def456"
    }
  ],
  "next_cursor": 127,
  "has_more": false
}
```

### 11.2 Usage and cost

`GET /agent-trust/agents/{agent_id}/usage?window=30d`

Allowed windows: `24h`, `7d`, `30d`, `all` (default `all`).

```json
{
  "window": "30d",
  "total_runs": 1250,
  "input_tokens": 610000,
  "output_tokens": 200000,
  "total_tokens": 810000,
  "total_cost_usd": 21.4,
  "cost_is_estimated": false,
  "last_used_at": "2026-10-09T09:20:00Z"
}
```

If `cost_is_estimated` is true, label the amount as estimated. Keep normal agent usage/cost separate from the overhead of Agent Trust review runs if the backend provides review-run cost metadata; do not infer missing review costs.

---

## 12. Current frontend/backend mismatches to resolve

These are important implementation items observed in the supplied source packages:

1. **Functional recommendation request:** frontend currently posts without a body. Backend requires a body model. Send `{ "question_count": 1 }`.
2. **Policy recommendation request:** frontend currently posts without a body. Backend requires a body model. Send `{ "force_refresh": false }` (or true for explicit force refresh).
3. **Functional suite result normalization:** backend review data returns `test_count` and `last_run: { total, passed, failed, at }`; frontend interface expects `total_tests`, `passed_tests`, `failed_tests`, and `last_run_at`. Add an API adapter mapping.
4. **Policy object shape:** frontend type expects `category`, `rules`, `enforcement_mode`; backend model uses `framework`, `severity`, `policy_prompt`, `judge_prompt`, `is_system_policy`, `is_active`, `version`. Align the TypeScript types and forms to backend fields.
5. **App usage references:** backend overview returns `used_in.apps` entries with `application_id` and `link`; frontend interface currently expects `id` and `name`. Adapt the frontend to the actual schema or change the backend response contract intentionally.
6. **Policy route canonicalization:** the backend includes both `/agent-trust/agents/{agent_id}/policies` (read view) and `/agent-trust/policies/agents/{agent_id}/policies` (policy service). The frontend uses the latter. Keep it consistent with the deployed router and use one canonical path per operation.
7. **Review status durability:** status/SSE event history is process-local memory. It can be lost on restart or unavailable if requests reach another worker. The UI must show the real 404/status error, not imply the review passed.

Do not hide these mismatches behind loose `any` types. Normalize the response once in `api.ts` and keep components typed against a stable frontend view model.

---

## 13. Recommended frontend API adapter types

Keep backend wire types separate from UI view types when their shapes differ. For example:

```ts
export interface UiTestSuite {
  suite_id: string;
  name: string;
  description: string | null;
  total_tests: number;
  passed_tests: number;
  failed_tests: number;
  last_run_at: string | null;
  source: "user" | "ai";
}

export function mapSuiteFromReview(raw: {
  suite_id: string;
  name: string;
  description?: string | null;
  test_count: number;
  source: "user" | "ai";
  last_run?: { total: number; passed: number; failed: number; at?: string | null } | null;
}): UiTestSuite {
  return {
    suite_id: raw.suite_id,
    name: raw.name,
    description: raw.description ?? null,
    total_tests: raw.last_run?.total ?? raw.test_count,
    passed_tests: raw.last_run?.passed ?? 0,
    failed_tests: raw.last_run?.failed ?? 0,
    last_run_at: raw.last_run?.at ?? null,
    source: raw.source,
  };
}
```

This is frontend normalization only; it does not change the backend contract.

---

## 14. Minimum integration test checklist

Before demo or merge, test these with a real authenticated session against the deployed API:

- [ ] Dashboard loads; empty-state dashboard renders without crashing.
- [ ] Agent list loads, search works, state filter works, pagination works.
- [ ] Selecting an agent loads overview and review workspace.
- [ ] Null score, null timestamps, empty policies/suites, and empty evidence render correctly.
- [ ] Review request returns `202` and a correlation ID.
- [ ] Check Progress handles started/running/completed/failed and 404.
- [ ] Functional recommendation sends `{ "question_count": 1 }` and handles 422/errors.
- [ ] User can inspect and explicitly accept/save a generated test suite; saved suite count updates.
- [ ] Policy recommendation sends `{ "force_refresh": false }` and response is normalized correctly.
- [ ] Policy assignment/removal refreshes the policy list.
- [ ] Timeline pagination/filtering works.
- [ ] Usage windows `24h`, `7d`, `30d`, `all` work.
- [ ] 401/403/404/422/5xx errors display clear messages without fake success states.
- [ ] Verify the full frontend build and a real backend/database smoke test in the target environment.

---

## 15. Contract ownership and non-goals

- Backend is authoritative for persisted values, check verdicts, scores, costs, and status.
- Frontend owns presentation, loading/error states, response normalization, and explicit user confirmation for recommendations.
- Missing evidence is not a pass.
- Do not fabricate sample agents, findings, progress, timestamps, token counts, or costs in production UI.
- Agent Trust suggestions do not mutate the agent's original prompt/configuration.
- Cost Governance is not an LLM-generated policy prompt.
- Keep the accepted primary navigation at **Agents**, **Agent Health**, and **Finding Detail**.
