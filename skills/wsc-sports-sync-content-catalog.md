---
name: wsc-sports-sync-content-catalog
description: Perform the initial full sync of a WSC Sports Experiences content catalog (Stories, Moments and Videos) into a local store, then keep it current from content-change webhooks. Use when standing up a Blaze Feed integration or repairing a store that has drifted.
api: WSC Sports Blaze Feed API
base_url: https://blazefeed.clipro.tv
operations:
  - getStories
  - getStoriesByIds
  - get /v1/moments
  - get /v1/moments/ids
  - get /v1/videos
  - get /v1/videos/ids
generated: '2026-09-04'
method: generated
source: openapi/wsc-sports-blaze-feed-api-reference.json + https://dev.wsc-sports.com/docs/blazefeed-api-v1
---

# Sync a WSC Sports content catalog

WSC Sports positions webhooks as the way to avoid polling, but webhooks only carry content that changes
**after** the webhook is configured. A correct integration is therefore always two parts: one full pull to
seed the store, then webhooks to keep it current. Doing only the webhook half leaves the entire back
catalog missing, and this is stated plainly in the provider's own documentation.

## Before you start

- You need the **Feed API key**, issued by a WSC Sports account manager. It is **not** the Experiences app
  API key used to initialize the SDK. Sending the wrong one returns `401`.
- The key travels in the `ApiKey` **query parameter**, not a header. Keep request URLs out of logs.
- Every operation is a `GET`. Nothing here mutates WSC Sports state, so every call is safe to retry.

## Step 1 — Seed each content type

There is no polymorphic content endpoint. Run the same loop three times, once per content type:

| Content type | List endpoint | By-ID endpoint |
|---|---|---|
| Stories | `GET /v1/stories` | `GET /v1/stories/ids` |
| Moments | `GET /v1/moments` | `GET /v1/moments/ids` |
| Videos | `GET /v1/videos` | `GET /v1/videos/ids` |

Page with `PageNum` (**0-based**) and `PageSize`. The response envelope carries `totalItems` and `result`,
so loop until you have collected `totalItems` items.

```
GET https://blazefeed.clipro.tv/v1/stories?PageNum=0&PageSize=100&ApiKey={FEED_KEY}
```

Narrow the seed if you do not want the whole catalog. The useful filters on the list endpoints are
`Statuses` (`Draft` or `Active` — filter to `Active` unless you are building an editorial tool),
`LabelsFilterExpression`, `Geo`, `OnlyLive`, `Title`, and the `CreateTime.From` / `UpdateTime.From`
time-range parameters.

## Step 2 — Store the dual identifiers

Every item carries a `metadata` block mapping it to the sport domain (`competition`, `game`, `team`,
`player`, `round`, `season`). Each of those is a `{provider, providerId, wscId}` triple.

**Persist both `wscId` and `providerId`.** `wscId` is WSC Sports' integer id; `providerId` is your own or
your data vendor's id. Storing only one of them is the single most expensive mistake in this integration,
because the filter parameters let you query by either (`Entities.Team.WscId` or
`Entities.Team.ServiceProviderId`, selected with `Entities.IdType`) and you cannot reconstruct the mapping
later without a full re-sync.

## Step 3 — Keep it current from webhooks

Ask your account manager to configure a webhook against an HTTPS endpoint you control. You choose how WSC
Sports authenticates to you: Basic, an `x-api-key` header, or a Bearer token. OAuth2 is not supported.

Each delivery carries `Action`, `ActionTime`, `ContentId`, `ContentType` and the content object. Apply
creates and updates by `ContentId`, and honour deletes.

**There is no payload signature.** A shared credential proves the caller knows a secret; it does not prove
a given body came from WSC Sports or that it is not a replay. Treat `Version` and `UpdateTime` on the
content object as your ordering guard and ignore any delivery older than what you already hold.

## Step 4 — Reconcile

Webhook retry behaviour, ordering and dead-lettering are all undocumented, so do not assume exactly-once
delivery. Re-run Step 1 with `UpdateTime.From` set to your last successful sync on a schedule to catch
anything the webhook stream dropped.

## Error and rate-limit handling

| Status | Meaning | Do |
|---|---|---|
| `400` | Invalid request parameters | Fix the parameter; do **not** retry unchanged |
| `401` | Authentication required | Check you sent the Feed key, not the app key |
| `403` | Insufficient permissions | The key is valid but not entitled; contact the account manager |
| `404` | Resource not found | Verify path and content IDs |
| `429` | Rate limit exceeded | Back off (see below) |
| `500` | Server error | Retry with exponential backoff — safe, all GETs |

Read `X-RateLimit-Remaining` on every response and slow down before it reaches zero. On `429` there is
**no `Retry-After` header**; use `X-RateLimit-Reset` (seconds) or exponential backoff. WSC Sports publishes
no numeric limit, so discover it at runtime from the headers rather than hard-coding a rate.

No error response body is documented, and an unauthenticated call was observed returning `401` with a
zero-byte body. Branch on the status code, not on a parsed error object.
