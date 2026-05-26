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
| Bookmarks | ❌ HTTP 422 `GRAPHQL_VALIDATION_FAILED` — **NOT 401**. The renamed `BookmarkSearchTimeline` operation requires a `rawQuery` variable; calling `…/Bookmarks?…` against the new query ID is the only failure that needs source changes. |

### Root cause for Bookmarks 422

x.com's web bundle (`main.ede5acfa.js`) no longer ships an operation named `Bookmarks`. The unfiltered timeline is now served by `BookmarkSearchTimeline` (queryId `5kB8iO1n19yXfcxM4e30Nw`), which:

1. Validates a required string variable named `rawQuery`. Sending no rawQuery yields `must be defined` → 422. Sending an empty string yields `ERROR_EMPTY_QUERY` → 200 with `code:214` Validation error.
2. Returns the timeline under `data.search_by_raw_query.bookmarks_search_timeline.timeline.instructions` instead of the previous `data.bookmark_timeline_v2.timeline.instructions`.

Setting `rawQuery: "filter:bookmarks"` reproduces the legacy "all bookmarks" view.

### Fix applied

`dist/lib/twitter-client-timelines.js` (the only TS->JS source shipped in this fork):

- `getBookmarksQueryIds()` now resolves `BookmarkSearchTimeline` first, with `Bookmarks` and the literal `5kB8iO1n19yXfcxM4e30Nw` as fallbacks.
- `getBookmarksPaged.fetchPage`:
  - URL: `…/${queryId}/Bookmarks` → `…/${queryId}/BookmarkSearchTimeline`.
  - `variables`: added `rawQuery: "filter:bookmarks"`.
  - Response parsing accepts both `search_by_raw_query.bookmarks_search_timeline.timeline.instructions` and the legacy `bookmark_timeline_v2.timeline.instructions`.

`dist/lib/twitter-client-constants.js` and `dist/lib/query-ids.json` also gained a `BookmarkSearchTimeline` entry, and `dist/commands/query-ids.js` adds it to the refresh target list so `bird query-ids --fresh` discovers the live ID from x.com bundles.

### Verification

```
bird home -n 5 --json       → tweet data
bird likes -n 5 --json      → tweet data
bird bookmarks -n 5 --json  → []           # @jwyu10 has no current bookmarks; HTTP 200, no errors
```

The original 401 hypotheses (transaction-id signing, missing feature flags) turned out not to be needed for any of the three endpoints once the operation rename was addressed.
