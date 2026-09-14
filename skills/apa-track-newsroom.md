---
name: apa-track-newsroom
description: Page APA Corporation's newsroom archive correctly, detect new press releases without re-reading the archive, and resolve authors and media in one request instead of three.
api: APA Corporation Newsroom API
operations:
  - listNewsroomPosts
  - getNewsroomPost
  - listAuthors
  - searchContent
---

# Track the APA Corporation newsroom

APA's press releases, community stories and operational updates are published through the WordPress REST API on the corporate host. It is public, anonymous and CORS-open. There were 51 published posts on 2026-09-14.

## Start here

```
GET https://apacorp.com/wp-json/wp/v2/posts?per_page=100&orderby=date&order=desc
```

Read these response headers before anything else:

- `X-WP-Total` — the true size of the collection.
- `X-WP-TotalPages` — how many pages exist at your `per_page`.
- `Link` — RFC 8288, carrying `rel="next"` until the last page.

`per_page` is capped at **100**. Asking for more returns `400 rest_invalid_param` with the exact bound in `data.details.per_page` — the server tells you the constraint, so read it rather than guessing.

## Stop at the end, not past it

Paging past the last page returns `400 rest_post_invalid_page_number`, not an empty array. Either follow `Link rel="next"` until it disappears, or bound your loop with `X-WP-TotalPages`. Do not treat that 400 as a failure of the API — it is the documented end of the collection.

## Detect new posts without re-reading the archive

```
GET https://apacorp.com/wp-json/wp/v2/posts?after=2026-09-01T00:00:00&orderby=date&order=asc
```

`after`, `before`, `modified_after` and `modified_before` all take ISO 8601. Store the `date_gmt` of the newest post you have seen and pass it as `after` on the next run. This is the correct incremental pattern here; there is no webhook, no event surface and no changelog to subscribe to (`asyncapi/` is absent for exactly this reason).

## Resolve authors and images in one request, not three

```
GET https://apacorp.com/wp-json/wp/v2/posts?per_page=20&_embed
```

`_embed` inlines the author record, terms and featured media under `_embedded`, collapsing what would otherwise be a lookup per post against `/wp/v2/users/{id}` and `/wp/v2/media/{id}`. Combine it with `_fields` to keep responses small:

```
GET https://apacorp.com/wp-json/wp/v2/posts?per_page=20&_fields=id,date,link,title,excerpt,author
```

## Do not rely on categories, tags or search

Measured on 2026-09-14: APA has **1** category (the default "uncategorized") and **0** tags, and every one of the 51 posts sits under that single term. Classification carries no information on this surface — do not build a topic filter on it.

Cross-resource search is also thin: `GET /wp/v2/search?search=energy` returned `X-WP-Total: 0`. Use the per-collection `search` parameter instead:

```
GET https://apacorp.com/wp-json/wp/v2/posts?search=Suriname
```

## Cache discipline

Responses carry `Cache-Control: max-age=600`. Nothing in a corporate newsroom changes faster than that. Poll at most every ten minutes, and prefer the `after` filter over re-reading the archive.

## Errors

`404 rest_post_invalid_id` for an unknown id, `400 rest_invalid_param` for a bad parameter, `401 rest_forbidden_context` if you ask for `context=edit` (you cannot — there is no credential a third party can obtain). Branch on `code`, not on the message text, and note the envelope is **not** RFC 9457 problem+json. Full catalogue in `errors/apa-problem-types.yml`.
