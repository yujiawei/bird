# Bird v0.8.0 — HomeTimeline / Bookmarks / Likes 401 Authentication Failure Analysis

## Problem Summary

bird v0.8.0 (last npm release of `@steipete/bird`) can authenticate and use some X GraphQL endpoints but fails with HTTP 401 on others:

| Endpoint | Status | Notes |
|---|---|---|
| SearchTimeline | ✅ Works | search, mentions |
| TweetDetail | ✅ Works | read, replies, thread |
| account/settings.json | ✅ Works | whoami, getCurrentUser |
| HomeTimeline | ❌ 401 | "Could not authenticate you" |
| HomeLatestTimeline | ❌ 401 | Same |
| Bookmarks | ❌ 401 | Same |
| Likes | ❌ Hangs/fetch failed | getCurrentUser() slow + possible 401 |

## Auth Flow

All requests use the same auth mechanism:
- Cookie: `auth_token=xxx; ct0=xxx`
- Header: `authorization: Bearer AAAAAAAAAAAAAAAAAAAAANRILgAAAAAAnNwIzUejRCOuH5E6I8xnZz4puTs%3D1Zv7ttfk8LF81IUq16cHjhLTvJu4FA33AGWWjCpTnA`
- Header: `x-csrf-token: <ct0>`
- Header: `x-twitter-auth-type: OAuth2Session`

This is the standard X web app auth. Same headers work for SearchTimeline but not HomeTimeline.

## Query IDs

Query IDs have been refreshed via `bird query-ids --fresh` and manually updated from x.com's `main.ede5acfa.js` bundle. Current correct IDs confirmed:

- HomeTimeline: `Ly0idwoXvMotg0ArhGnnow`
- HomeLatestTimeline: `MR2EaMHFNTqPEFIodfclng`
- Likes: `a5tnuYtSDnZ9rdUBSqS5Og`
- SearchTimeline: `099UqLkXma7fhT81Jv4n9g`

Note: `Bookmarks` operation was NOT found in the main bundle. Only `BookmarkSearchTimeline` (`5kB8iO1n19yXfcxM4e30Nw`) exists, suggesting X renamed/replaced the endpoint.

## Hypothesis: Client Transaction ID Signing

X's anti-automation system likely validates `x-client-transaction-id` differently for "sensitive" endpoints (HomeTimeline, Bookmarks, Likes) vs "public" ones (SearchTimeline).

Looking at bird's code in `twitter-client-base.js`:
```js
createTransactionId() {
    return randomBytes(16).toString('hex');
}
```

This generates a random transaction ID. However, X's current web app may require a **signed** transaction ID derived from the page's JavaScript context. SearchTimeline may not enforce this, but HomeTimeline/Bookmarks/Likes do.

## Hypothesis: Missing/Changed Features

The features object sent with the request may need updated fields. X adds new required feature flags periodically. The `buildHomeTimelineFeatures()` in bird v0.8.0 may be missing fields that the current API requires.

## Hypothesis: Bookmarks Endpoint Renamed

The `Bookmarks` GraphQL operation no longer exists in x.com's JS bundles. It may have been replaced by `BookmarkSearchTimeline`. The code needs to be updated to use the new operation name and query structure.

## Investigation TODOs

1. **Capture real x.com requests** — Use browser DevTools to capture the actual headers/payload that x.com sends for HomeTimeline, then compare with what bird sends
2. **x-client-transaction-id signing** — Reverse-engineer how x.com generates this header; check if it uses a HMAC or other signing mechanism
3. **Feature flags audit** — Extract the exact features object x.com sends for each endpoint from the JS bundles
4. **Bookmarks migration** — Determine if `BookmarkSearchTimeline` is the replacement and what its query variables look like
5. **Likes timeout** — Debug why `getCurrentUser()` is extremely slow (15-20s) causing likes to appear hung

## Environment

- Mac: arm64, macOS 26.3.1
- Node: v26.0.0
- bird: v0.8.0 (npm @steipete/bird)
- birdclaw: v0.6.0
- Cookie auth verified working (whoami: @jwyu10, uid 1229323301555601408)

## Files

Key source files (compiled JS, no TypeScript source available):
- `dist/lib/twitter-client-base.js` — Base class, headers, auth, fetchWithTimeout
- `dist/lib/twitter-client-home.js` — HomeTimeline/HomeLatestTimeline
- `dist/lib/twitter-client-timelines.js` — Likes, Bookmarks
- `dist/lib/twitter-client-features.js` — Feature flag builders
- `dist/lib/twitter-client-constants.js` — Query IDs, fallbacks
- `dist/lib/runtime-query-ids.js` — Runtime query ID refresh from x.com bundles

---

## Resolution (2026-05-26)

Empirical re-test against the live X API with the same cookies on @jwyu10 showed:

| Endpoint | Actual status |
|---|---|
| HomeTimeline / HomeLatestTimeline | ✅ HTTP 200 (returns tweet data) once `bird query-ids --fresh` populated the runtime cache; the 401 reported in the original investigation reproduces only when stale query IDs are used. |
| Likes | ✅ HTTP 200 (returns tweet data). |
| Bookmarks | ❌ HTTP 200 with empty `entries` (cursor-only) — **not a 401 either**. `BookmarkSearchTimeline` with `rawQuery: "filter:bookmarks"` returned no tweets for the affected accounts; the legacy `Bookmarks` operation (queryId `RV1g3b8n_SGOHwkqKYSCFw`) is still served by the X GraphQL backend and continues to return the unfiltered bookmark timeline. |

### Browser capture (YUJ-2079)

Captured headers from `x.com`'s web client side-by-side with bird's request: `x-csrf-token`, `cookie` (`auth_token=…; ct0=…`), `authorization: Bearer …`, and a randomly-generated `x-client-transaction-id` are sufficient to reach 200 from a residential or server IP. Adding the full Camofox cookie jar (`twid`, `kdt`, `guest_id`, `personalization_id`, `lang`, `d_prefs`, `__cuid`, `guest_id_ads`, `guest_id_marketing`) produces the same status codes — the original 401 hypotheses (transaction-id signing, browser-fingerprint cookies, `sec-ch-ua` headers) were red herrings.

### Root cause for Bookmarks empty result

YUJ-2078 switched the Bookmarks call to `BookmarkSearchTimeline` with `rawQuery: "filter:bookmarks"`, on the assumption that the operation had been renamed. In reality both operations exist and serve different purposes:

- `Bookmarks` (`RV1g3b8n_SGOHwkqKYSCFw`) — the unfiltered bookmark timeline, used by `/i/bookmarks` page loads. Variables: `count`, `includePromotedContent`, `withDownvotePerspective`, `withReactionsMetadata`, `withReactionsPerspective`. Response under `data.bookmark_timeline_v2.timeline.instructions`.
- `BookmarkSearchTimeline` (`5kB8iO1n19yXfcxM4e30Nw`) — the search-box backend. Requires a `rawQuery`. With `rawQuery: "filter:bookmarks"` it returns 200 with cursor-only entries (no tweets) for accounts whose bookmark search index has not been populated. Response under `data.search_by_raw_query.bookmarks_search_timeline.timeline.instructions`.

Two bugs compounded:

1. The previous fetch loop hard-coded `BookmarkSearchTimeline` in the URL even when iterating over the legacy `Bookmarks` queryIds, so `…/RV1g3b8n_SGOHwkqKYSCFw/BookmarkSearchTimeline` was tried (incorrect operation pairing).
2. The `rawQuery: "filter:bookmarks"` payload returns no tweets for the target account, so even the `5kB8iO1n19yXfcxM4e30Nw` queryId could not produce data.

### Fix applied (YUJ-2079)

`dist/lib/twitter-client-timelines.js`:

- `getBookmarksQueryIds()` now returns `{queryId, operation}` pairs, with the legacy `Bookmarks` op tried first and `BookmarkSearchTimeline` as fallback.
- `getBookmarksPaged.fetchPage`:
  - URL: `…/${queryId}/${operation}` so each queryId is paired with the matching operation name.
  - `variables`: `rawQuery: "filter:bookmarks"` is only sent when calling `BookmarkSearchTimeline`; the legacy `Bookmarks` op gets the unfiltered shape.
  - 4xx (other than 404) on one candidate now skips to the next candidate so a legacy 422 cannot mask a working fallback.
  - When `BookmarkSearchTimeline` returns 200 with no tweets and no cursor on the first page, the loop falls through to the next candidate instead of reporting an empty timeline.
  - Response parsing reads `data.bookmark_timeline_v2.timeline.instructions` first, then falls back to `data.search_by_raw_query.bookmarks_search_timeline.timeline.instructions`.

### Verification

```
bird home -n 3 --json       → 3 tweets
bird likes -n 3 --json      → 3 tweets
bird bookmarks -n 3 --json  → 3 tweets
```

Verified on the same Mac (Node v26.0.0, bird v0.8.0 + this patch) with the cookies in `~/.zshrc`.

The original 401 hypotheses (transaction-id signing, missing feature flags) turned out not to be needed for any of the three endpoints.
