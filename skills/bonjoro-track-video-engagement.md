---
name: Track engagement on sent Bonjoro videos
description: Poll Bonjoro's results, events and transmission surfaces to learn whether a personal video was delivered, opened, watched, clicked or replied to.
api: openapi/bonjoro-api-v2-openapi.yml
base_url: https://www.bonjoro.com/api/v2
operations:
  - getGreetResults
  - getGreetResultsStats
  - getResults
  - getResultStats
  - getGreetEvents
  - getEvents
  - getTransmissions
  - getReplies
generated: '2026-08-12'
method: generated
source: openapi/_original/bonjoro-api-v2-openapi-original.json
---

# Track engagement on sent Bonjoro videos

Bonjoro publishes **no AsyncAPI and no webhook payload schema**, so the machine-readable path to
engagement data is polling the REST surface. (The push path exists — 11 events via the first-party
Zapier app and a help-centre webhook article — but is undocumented as a contract; see
`asyncapi/bonjoro-webhooks.yml`.)

Authenticate first as in `bonjoro-send-personal-video.md` (`authenticate`, then
`Authorization: Bearer <token>`).

## Roll-up view — `getGreetResults`

`GET /api/v2/greets/results` returns the paginated per-recipient outcome list. Query parameters:
`sort`, `assignee_id`, `profile_id`, `limit`, `simple`. The response uses the Laravel page-number
envelope — follow `next_page_url` until it is null, and note that `prev_page_url` is (incorrectly)
typed as an integer in the published schema.

`GET /api/v2/greets/results/stats` (`getGreetResultsStats`) returns the aggregate counters for the same
set.

## Single greet + recipient — `getResults`

`GET /api/v2/results/{greet_id}/{profile_id}` returns the `result` object, which carries the greet, the
sending user, the replies, and a `greetProfileStats` block. `GET
/api/v2/results/{greet_id}/{profile_id}/stats` (`getResultStats`) is the stats-only variant.

`PUT /api/v2/results/{greet_id}/{profile_id}` (`putResult`) writes back to a result — use it only when
you intend to mutate state.

## Event streams — `getGreetEvents` / `getEvents`

- `GET /api/v2/greets/{greet_id}/events` — activity for one greet.
- `GET /api/v2/events` — account-wide event feed. Each `event` is
  `{id, type, source, message, occured_at}`; `type` is a free-text discriminator (example `DemoEvent`)
  and Bonjoro publishes **no enum of event types**, so treat it as an open vocabulary and match
  defensively.

Timestamps are `Y-m-d H:i:s` strings (`2021-01-19 23:45:45`), **not** RFC 3339 — parse them as naive
local strings, do not assume a timezone offset is present.

## Deliverability — `getTransmissions`

`GET /api/v2/transmisions` (note: the path is misspelled in the published contract — send it exactly as
written) returns transmission records, each carrying `transmissionEvent[]` with
`{event_type, message, created_at}`. `event_type` examples include `sent`. This is where a bounce shows
up.

## Replies — `getReplies`

`GET /api/v2/replies` lists inbound recipient replies; `POST /api/v2/replies` (`postReply`) posts one.

## Polling discipline

`x-ratelimit-limit: 600` was observed on live responses with no reset header and no published window.
Poll on a fixed interval, read `x-ratelimit-remaining` on every response, and stop before zero — the
429 behaviour is undocumented and no `Retry-After` was observed. Prefer `getEvents` (one call) over
per-greet polling when watching many greets.
