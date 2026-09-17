# API Contract

Base URL (local): `http://localhost:8080`

All request and response bodies are JSON with `snake_case` keys. Timestamps are
RFC 3339 in UTC (`2026-09-15T16:53:54Z`). Dates are `YYYY-MM-DD` with no time
or zone.

Auth arrives in Phase 6. Until then endpoints are unauthenticated and storage is
in memory. Once auth lands, every endpoint except `/health` requires
`Authorization: Bearer <jwt>` and is scoped to the token's user.

## Conventions

| Situation | Status |
|-----------|--------|
| Read succeeded | 200 |
| Resource created | 201 |
| Deleted | 204, no body |
| Malformed or invalid input | 400 |
| Missing or invalid token (Phase 6) | 401 |
| Not found, or owned by another user | 404 |
| Conflicts with current state | 409 |
| Well-formed but unprocessable (extraction) | 422 |
| Upstream fetch failed | 502 |

Requests for another user's resource return 404, never 403.

## Error Shape

```json
{
  "error": {
    "code": "validation_error",
    "message": "one or more fields are invalid",
    "fields": [
      {"field": "deadline", "message": "must be YYYY-MM-DD"},
      {"field": "salary_max", "message": "must be >= salary_min"}
    ]
  }
}
```

`fields` is present only for `validation_error`.

| Code | Status | Meaning |
|------|--------|---------|
| validation_error | 400 | Input failed validation |
| invalid_json | 400 | Body is not valid JSON |
| invalid_status_transition | 400 | Transition not allowed from current status |
| unauthorized | 401 | Missing, expired, or invalid token |
| not_found | 404 | No such resource for this user |
| cannot_delete_submitted | 409 | Delete attempted on a non-saved application |
| duplicate_application | 409 | This user already has an application for this URL |
| draft_already_accepted | 409 | Draft was already accepted |
| extraction_failed | 422 | URL fetched but no usable fields found |
| upstream_fetch_failed | 502 | Could not fetch the listing URL |
| internal_error | 500 | Unexpected failure |

## Enums

| Enum | Values |
|------|--------|
| status | saved, applied, oa, interview, offer, rejected, archived |
| category | swe, data, pm, hardware, other |
| salary_period | hour, month, year, unknown |

## The Application Object

```json
{
  "id": "a3f1c2d4-5e6f-4a7b-8c9d-0e1f2a3b4c5d",
  "company_name": "Acme Corp",
  "title": "Software Engineer Intern",
  "category": "swe",
  "category_label": "Backend",
  "locations": ["Austin, TX", "Remote"],
  "salary_min": 45.00,
  "salary_max": 55.00,
  "salary_currency": "USD",
  "salary_period": "hour",
  "deadline": "2026-11-01",
  "status": "applied",
  "source_url": "https://example.com/jobs/123",
  "notes": "Referred by Sam.",
  "created_at": "2026-09-15T16:53:54Z",
  "updated_at": "2026-09-16T09:12:00Z"
}
```

`company_name` is flat in the API. The server resolves it to a `companies` row
internally (find-or-create on a normalized name). The client never sees a
company id.

### Field Rules

| Field | Type | Required on create | Rules |
|-------|------|--------------------|-------|
| id | string (uuid) | server-set | Read-only |
| company_name | string | yes | 1-200 chars, trimmed, non-empty |
| title | string | yes | 1-200 chars, trimmed, non-empty |
| category | string enum | no | Defaults to `other` |
| category_label | string or null | no | Max 60 chars |
| locations | string array | no | Defaults to `[]`, max 10 entries, each 1-120 chars |
| salary_min | number or null | no | >= 0, max 2 decimal places |
| salary_max | number or null | no | >= 0, must be >= salary_min when both present |
| salary_currency | string or null | no | Exactly 3 uppercase letters, defaults to `USD` when a salary is present |
| salary_period | string enum | no | Defaults to `unknown` when a salary is present |
| deadline | string or null | no | `YYYY-MM-DD`, must parse as a real date |
| status | string enum | no | Defaults to `saved` on create |
| source_url | string or null | no | http or https, max 2000 chars |
| notes | string | no | Defaults to `""`, max 5000 chars |
| created_at | string | server-set | Read-only |
| updated_at | string | server-set | Read-only |

Unknown fields in a request body are rejected with `validation_error`, so typos
fail loudly instead of being silently ignored.

## PATCH Semantics

Standard JSON Merge Patch rules:

- **Omitted key** means leave the field unchanged.
- **Explicit `null`** means clear the field (only for nullable fields).
- **A value** means set the field.

```json
{"deadline": null, "notes": "Withdrew after the OA."}
```

Clears the deadline, replaces the notes, changes nothing else.

Sending `null` for a non-nullable field (`company_name`, `title`, `status`,
`locations`, `notes`) returns `validation_error`.

`locations` is replaced wholesale, not merged.

In Go this requires pointer fields so an absent key and an explicit null are
distinguishable:

```go
type UpdateApplicationRequest struct {
    CompanyName *string   `json:"company_name"`
    Deadline    *string   `json:"deadline"`
    Locations   *[]string `json:"locations"`
}
```

A nil pointer means the key was absent. A non-nil pointer to a zero value means
it was present. For fields where `null` is meaningful, decode into
`json.RawMessage` or use a custom nullable type to tell null from absent.

---

## GET /health

No auth. Returns 200.

```json
{"status": "ok"}
```

## GET /applications

Lists the signed-in user's applications.

### Query Parameters

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| status | string, repeatable | all | Any status value. `?status=applied&status=oa` |
| category | string, repeatable | all | Any category value |
| location | string | none | Case-insensitive substring match against any location |
| deadline_before | date | none | Inclusive |
| deadline_after | date | none | Inclusive |
| q | string | none | Case-insensitive substring match on company_name and title |
| sort | string | updated_at | One of: updated_at, created_at, deadline, company_name, title, salary |
| order | string | desc | asc or desc |
| limit | int | 50 | 1-200 |
| offset | int | 0 | >= 0 |

`sort=salary` orders by a computed hourly equivalent (year / 2080, month / 173,
hour as-is). Rows with `salary_period` of `unknown` or no salary sort last
regardless of `order`. Rows with a null `deadline` sort last when
`sort=deadline`.

Invalid enum values, an unknown `sort`, or an out-of-range `limit` return
`validation_error`.

### Response 200

```json
{
  "data": [ { "...application object..." } ],
  "total": 42,
  "limit": 50,
  "offset": 0
}
```

`total` is the count matching the filters, ignoring limit and offset.

## POST /applications

Creates a manual application.

### Request

```json
{
  "company_name": "Acme Corp",
  "title": "Software Engineer Intern",
  "category": "swe",
  "locations": ["Austin, TX"],
  "salary_min": 45,
  "salary_max": 55,
  "salary_currency": "USD",
  "salary_period": "hour",
  "deadline": "2026-11-01",
  "status": "saved",
  "source_url": "https://example.com/jobs/123",
  "notes": ""
}
```

Only `company_name` and `title` are required.

Creating with a status other than `saved` is allowed for backfilling existing
applications, and writes a `created` event at that status.

### Responses

- **201** the application object
- **400** `validation_error`, `invalid_json`
- **409** `duplicate_application` when `source_url` matches an existing
  application for this user

## PATCH /applications/{id}

Edits fields other than status.

Including `status` in the body returns `validation_error` with the message that
status changes go through `POST /applications/{id}/status`. This keeps every
status change paired with an event.

### Responses

- **200** the updated application object
- **400** `validation_error`, `invalid_json`
- **404** `not_found`
- **409** `duplicate_application` when changing `source_url` to one already used

## DELETE /applications/{id}

Hard deletes an application. Only permitted while status is `saved`.

### Responses

- **204** no body
- **404** `not_found`
- **409** `cannot_delete_submitted`, with a message pointing at archiving instead

Anything past `saved` is moved to `archived` through the status endpoint.

## POST /applications/{id}/status

Changes status and appends an event.

### Request

```json
{"status": "interview", "note": "Phone screen scheduled for Friday."}
```

`status` is required. `note` is optional, max 1000 chars, and is stored on the
event, not on the application.

### Allowed Transitions

| From | To |
|------|-----|
| saved | applied, archived |
| applied | oa, interview, offer, rejected, archived |
| oa | interview, offer, rejected, archived |
| interview | interview, offer, rejected, archived |
| offer | rejected, archived |
| rejected | applied, oa, interview, offer, archived |
| archived | the status it held before archiving, recovered from event history |

Notes:

- `interview -> interview` is allowed for multiple rounds and writes an event
  each time.
- `rejected` can move back, since companies do reverse decisions.
- Transitioning to the same status (other than interview) returns
  `invalid_status_transition`.
- Un-archiving with no prior status in the event history falls back to `saved`.

### Responses

- **200** the updated application object
- **400** `validation_error`, `invalid_status_transition`
- **404** `not_found`

## Draft Endpoints

`POST /listing-drafts`, `GET /listing-drafts/{id}`,
`PATCH /listing-drafts/{id}`, and `POST /listing-drafts/{id}/accept` are
specified in Phase 5 alongside the extraction pipeline design. They use the same
error shape and the `extraction_failed`, `upstream_fetch_failed`,
`duplicate_application`, and `draft_already_accepted` codes.

## Deferred To Later Phases

| Endpoint | Phase |
|----------|-------|
| GET /applications/{id}/events | 8 |
| GET /analytics/summary | 7 |
