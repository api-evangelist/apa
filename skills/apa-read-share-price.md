---
name: apa-read-share-price
description: Read APA Corporation's delayed share price from the one endpoint APA built itself, and present it with the staleness the cache actually implies.
api: APA Corporation Ticker API
operations:
  - getApaShareQuote
  - getTickerNamespaceIndex
---

# Read the APA Corporation share price

APA Corporation (Nasdaq: APA) registered exactly one REST route of its own on apacorp.com. It returns the delayed quote that drives the ticker on the corporate site, it needs no credential, and it is CORS-open.

## Call it

```
GET https://apacorp.com/wp-json/apa-ticker/v1/quote
```

No headers are required. No API key exists to send.

## What comes back

```json
{"symbol":"APA","price":45.09,"change":0.06,"percent":0.13,"is_up":true,"source":"nasdaq"}
```

Six fields, nothing else. `change` and `percent` are measured against the previous close. `is_up` is a convenience boolean that restates the sign of `change` — do not derive one from the other and get a different answer; use `is_up` as published.

## The rule that matters: this is not a real-time price

The response is served `Cache-Control: max-age=600, must-revalidate` through Cloudflare, and an `Age` header up to 600 has been observed on a cache HIT. **The value you receive can be ten minutes old, and you cannot tell how old from the body.**

- Read the `Age` response header. Report freshness as "as of up to `Age` + 600 seconds ago" rather than as "now".
- Never label this a live or real-time quote.
- Never use it to make or justify a trading decision. It is a corporate-site widget feed, not a market data service, and APA publishes no accuracy or availability commitment for it (see `lifecycle/apa-lifecycle.yml` — no SLA, no status page, no deprecation policy).
- `source` has been observed as `nasdaq`. Carry it through to the user so the upstream is attributed.

## Polling

Do not poll faster than the cache. One call per ten minutes is the most that can return new information; anything faster burns requests for an identical body. There is no published rate limit and no rate-limit header on this surface (`rate-limits/apa-rate-limits.yml`), so there is no runtime signal to tell you when you have gone too far — you would simply be cut off by an edge rule you cannot see.

## Confirm the route still exists

```
GET https://apacorp.com/wp-json/apa-ticker/v1
```

Returns the namespace registration listing the routes under `apa-ticker/v1`. Use it as a cheap existence check before reporting a failure. APA publishes no deprecation policy and returns no `Sunset` header, so this index is your only warning mechanism if the route is ever retired.

## Errors

A wrong path under this namespace returns `404 {"code":"rest_no_route"}`. There are no other error shapes on this route — it takes no parameters, so there is nothing to get wrong. See `errors/apa-problem-types.yml`.
