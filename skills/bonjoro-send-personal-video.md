---
name: Send a personal Bonjoro video to a new customer
description: Authenticate against the Bonjoro API V2, create or reuse a recipient profile, create the greet (video task), attach a recording, and mark it complete.
api: openapi/bonjoro-api-v2-openapi.yml
base_url: https://www.bonjoro.com/api/v2
operations:
  - authenticate
  - postProfile
  - postGreet
  - getGreetUploadUrl
  - sendNotification
generated: '2026-08-12'
method: generated
source: openapi/_original/bonjoro-api-v2-openapi-original.json
---

# Send a personal Bonjoro video

A "greet" is the Bonjoro unit of work: a task assigned to a team member to record and send a personal
video to one or more recipient profiles. Everything below is grounded in operationIds that exist
verbatim in `openapi/bonjoro-api-v2-openapi.yml`.

## Before you start

- Base URL is `https://www.bonjoro.com/api/v2`. Do **not** use `api.bonjoro.com` — it sits behind a
  Cloudflare interactive challenge and is not the API host.
- REST API access is a **Company-tier** entitlement (see `plans/bonjoro-plans-pricing.yml`). If the
  account is on Free/Starter/Pro/Grrrowth, expect `401`/`403` regardless of correct code.
- There is **no idempotency key** on any operation. If a create call times out, do not blind-retry —
  read back with `getGreets` first, or you will duplicate the task. See
  `conventions/bonjoro-conventions.yml`.

## 1. Get a token — `authenticate`

`POST /api/v2/oauth/2/token` with a JSON body carrying `grant_type` (`password`,
`client_credentials`, or `authorization_code`), plus the credentials for that grant. The response is
`{token_type: "Bearer", expires_in, access_token, refresh_token}`. Send it on every subsequent call as
`Authorization: Bearer <access_token>`.

The published spec declares **no** `components.securitySchemes`, so a generated client will not wire
this header for you — add it manually.

## 2. Create the recipient — `postProfile`

`POST /api/v2/profiles` with `{"email": "...", "first_name": "...", "last_name": "..."}`. Only `email`
is required. Returns the profile with a UUID `id`. If the profile already exists, look it up with
`getProfiles` (supports `search`, `limit`, `sort`) rather than creating a duplicate.

## 3. Create the greet — `postGreet`

`POST /api/v2/greets` with:

- `profiles` — array of up to **200** entries, each an email address *or* a profile UUID
- `assignee_id` — UUID of the team member who will record (nullable)
- `campaign_id` — UUID of the workspace/campaign the task belongs to (nullable)
- `template_id` — UUID of the message template to apply (nullable)
- `note` — free text shown to the assignee, e.g. `"New signup"`

For a list rather than a single recipient, use `bulkCreateGreets` (`POST /api/v2/greets/create`) with
`lines[]` of `{email, first_name, last_name, reason}` plus `assignee_id`, `campaign_id`, `reason` and
`sync` (`1`/`0`).

To reassign later, `putGreet` (`PUT /api/v2/greets/{greet_id}`) takes only `assignee_id`.

## 4. Attach the video — `getGreetUploadUrl`

`GET /api/v2/greets/{greet_id}/upload-url` returns a presigned S3 upload URL; PUT the media to that URL
directly. For standalone recordings the equivalent is `getUploadURL`
(`GET /api/v2/recordings/{recording_id}/upload-url`), which **requires** the `part_number` query
parameter — it is a multipart upload.

## 5. Send it — `sendNotification`

`POST /api/v2/greets/{greet_id}/notifications` with `{"via": ["email"]}` marks the greet complete and
delivers it. `email` is the only enum value the contract publishes.

## Error handling

| Status | Envelope | What to do |
|---|---|---|
| 400 | `{"error":{"message","status_code"}}` | Validation failure — fix the body, do not retry as-is |
| 401 | `{"message":"Unauthorized."}` — **different shape** | Refresh the token and retry once |
| 403 | `{"error":{...}}` | Token valid, resource not owned by this team — do not retry |
| 404 | `{"error":{...}}` | Bad UUID or out-of-team resource — do not retry |
| 500 | `{"error":{...},"message":...}` | Undeclared in the spec but reachable; retry with backoff |

Parse `401` separately: it is the one status that does **not** use the `error` wrapper. Full catalogue:
`errors/bonjoro-problem-types.yml`.

## Rate limits

`x-ratelimit-limit: 600` and `x-ratelimit-remaining` are returned on live responses (observed
2026-08-12); no window, no reset header, and no documented policy. Read `x-ratelimit-remaining` and back
off before it reaches zero — the exhaustion response is undocumented. See
`rate-limits/bonjoro-rate-limits.yml`.
