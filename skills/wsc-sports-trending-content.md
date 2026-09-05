---
name: wsc-sports-trending-content
description: Retrieve the currently trending Stories or Moments from the WSC Sports Blaze Feed API and render them in the right order. Use when building a trending rail, a "what's hot" surface, or any ranked content feed driven by WSC Sports recommendations.
api: WSC Sports Blaze Feed API
base_url: https://blazefeed.clipro.tv
operations:
  - getTrendingRecommendations
generated: '2026-09-04'
method: generated
source: openapi/wsc-sports-blaze-feed-api-reference.json + https://dev.wsc-sports.com/docs/blazefeed-api-recommendations
---

# Get trending content from WSC Sports

`GET /v1/recommendations/trending` returns what WSC Sports currently ranks as trending. It behaves
differently from every other endpoint on this API, and each difference is a place integrations go wrong.

## The call

```
GET https://blazefeed.clipro.tv/v1/recommendations/trending?ContentType=Moment&ApiKey={FEED_KEY}
```

| Parameter | Required | Notes |
|---|---|---|
| `ApiKey` | yes | The **Feed** key, not the Experiences app key |
| `ContentType` | yes | **One value per call.** Only `Story` and `Moment` are supported — not `Video` |
| `Geo` | no | ISO 3166 two-letter country code, e.g. `US`. Omit for globally trending content |

## The four things that differ from the catalog endpoints

1. **The order *is* the ranking.** Results come back highest-trending first. Trending scores are not
   exposed and **no field marks an item as trending**. If you re-sort the array — by date, by title, by
   anything — you have thrown away the entire product of this endpoint. Render in received order.
2. **No pagination.** `PageNum` and `PageSize` do not apply. The response is capped at **100 items**.
3. **One content type per call.** To build a mixed rail, call twice (`Story` and `Moment`) and interleave
   deliberately; there is no cross-type ranking to inherit.
4. **The envelope is different.** You get `RecommendationsResponseDTO` — `lastUpdated` plus
   `recommendations[]` — not the `totalItems` / `result` pagination envelope.

## Handle the empty result as normal, not as an error

A `200` with an empty `recommendations` array is expected and documented. It means either trending is not
enabled for that app and content type, or there is nothing trending right now. Render your fallback rail;
do not retry in a loop and do not surface an error.

## Use `lastUpdated`

`lastUpdated` tells you when the ranking was computed. Cache against it rather than polling tightly — the
ranking does not change per request, and this endpoint is subject to the same undocumented rate limit as
the rest of the API. Read `X-RateLimit-Remaining` and back off before it hits zero; on `429` there is no
`Retry-After`, so use `X-RateLimit-Reset`.

## Rendering the items

Each recommendation carries `id`, `type`, `title`, `subtitle`, `description`, `duration`, `status`,
`aspectRatios`, `thumbnails[]`, `geoTargeting`, `metadata` and `labels[]`.

- Pick a thumbnail by matching `aspectRatio` to your surface rather than taking `thumbnails[0]`. The
  published example returns `TwoToThree`, `Square` and a `Vertical` GIF in one item.
- Honour `geoTargeting.includedGeos` / `excludedGeos` if you cache and re-serve results across regions.
  Passing `Geo` gets you the right ranking, but the per-item targeting still applies.
- `id` is the content ID you would pass to `/v1/stories/ids` or `/v1/moments/ids` to fetch the full object.
