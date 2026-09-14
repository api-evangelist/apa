---
name: apa-read-leadership-roster
description: Read APA Corporation's executive and board roster from the custom post type APA registered for it, and handle the id-scoping trap that makes this collection different from stock WordPress.
api: APA Corporation Leadership API
operations:
  - listLeadership
  - getLeadershipProfile
  - listContentTypes
---

# Read the APA Corporation leadership roster

APA registers a `leaderships` post type of its own — it is not stock WordPress — carrying the executive team and board-of-director profiles rendered on the corporate governance pages. There were 25 published entries on 2026-09-14.

## Discover it first

```
GET https://apacorp.com/wp-json/wp/v2/types
```

This is the document that proves `leaderships` exists and gives its `rest_base` and `rest_namespace`. Three other APA-registered types appear in the same response — `pp_video_block` (Media Hub, `rest_base` **presto-videos**), `visualizer` and `wprss_feed_item`. Note the pattern: **the post type slug is not always the REST base.** Always read `rest_base` from this document rather than assuming the slug.

## List the roster

```
GET https://apacorp.com/wp-json/wp/v2/leaderships?per_page=100&orderby=menu_order&order=asc
```

`orderby=menu_order` reproduces the ordering the site itself renders, which for a leadership page is usually seniority rather than alphabetical or chronological. `orderby=date` would give you the order the records were created, which is meaningless here.

At 25 entries one page of 100 covers the whole collection. Confirm with `X-WP-Total` rather than assuming it stays that size.

## The id trap

Leadership ids are **site-unique WordPress post ids, not a per-collection sequence.** Verified live: `GET /wp/v2/leaderships/1` returns `404 rest_post_invalid_id` — id 1 is a real WordPress id on this site, but it is not a `leaderships` record.

So:

- Never iterate ids from 1.
- Never carry an id from one collection to another.
- Always take ids from the collection response itself.

A `404 rest_post_invalid_id` here means "not a leadership record", which is a different fact from "does not exist".

## Fetch one profile

```
GET https://apacorp.com/wp-json/wp/v2/leaderships/{id}
```

Use an id returned by the list call. `title.rendered` and `content.rendered` are HTML — render or strip them, do not treat them as plain text. An `acf` object is present on this type and carries the site's custom fields for the profile.

## Headshots

The profile carries `featured_media` as an integer attachment id, or `0` when none is set. Resolve it with `_embed` on the list call rather than a per-record lookup:

```
GET https://apacorp.com/wp-json/wp/v2/leaderships?per_page=100&_embed
```

## Handling the people data

These are published corporate biographies of named executives and directors — already public on apacorp.com. Reproduce them as published and attribute them to APA. Do not enrich, cross-reference or combine them with other sources to build a profile of an individual; that is outside what this surface is for. The same caution applies to the five public author records on `/wp/v2/users`.

## No credential, no context=edit

Everything above works anonymously. `context=edit` returns `401 rest_forbidden_context` and there is no application password a third party can obtain (`authentication/apa-authentication.yml`).
