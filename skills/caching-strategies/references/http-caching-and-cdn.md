# HTTP Caching and CDNs

Reference for the HTTP cache semantics defined by RFC 9111 (June 2022) and the operational discipline of running CDN-edge caches above origin services.

## RFC 9111 in one sentence

`Cache-Control` is both a request and response header. Response directives control whether and how a cache stores and reuses a response; request directives constrain what cached responses the client is willing to accept. RFC 9111 obsoletes RFC 7234 and is the current HTTP caching spec.

## `Cache-Control` directives that matter

- **`public`** — storable in shared caches even if `Authorization` was present in the request.
- **`private`** — storable only in user-agent-private caches (the browser). Shared caches (CDN, proxy) must not store.
- **`no-cache`** — must revalidate with origin before reuse. **Not** "do not cache" — that's `no-store`. The most-confused directive.
- **`no-store`** — do not store in any cache. The actual "do not cache" directive.
- **`max-age=N`** — fresh for N seconds.
- **`s-maxage=N`** — `max-age` for shared caches only; overrides `max-age` and `Expires` for them. Useful when the browser should hold for a short time but the CDN should hold for longer.
- **`must-revalidate`** — once stale, cannot be served without revalidation. Disables stale-fallback.
- **`stale-while-revalidate=N`** — RFC 5861 extension recognized by RFC 9111's stale-response model. Serve stale up to N seconds after expiry while refreshing in background. The simplest stampede mitigation at the cache layer.
- **`stale-if-error=N`** — RFC 5861. Serve stale up to N seconds if origin returns 5xx or is unreachable. Availability boost when the source is down.
- **`immutable`** — RFC 8246, September 2017. Tells the client not to revalidate during freshness lifetime even on user-initiated reload. Designed for content-hashed asset URLs.
- **`no-transform`** — proxies must not modify the content (relevant for image-resizing CDNs).

## Validators and conditional requests

When a cached entry expires, the cache sends a conditional request to origin:

- **`ETag`** (opaque, strong by default) on the response, then `If-None-Match: <etag>` on the conditional request. Origin returns `304 Not Modified` if unchanged. The cache extends the entry's freshness without re-downloading the body.
- **`Last-Modified`** (HTTP-date) with `If-Modified-Since`. Older mechanism; lower precision (seconds), and hits clock-skew issues. Use `ETag` when both are available.

`304` saves bandwidth, not a round-trip. The benefit is on large-payload, rarely-changing resources (images, large JSON documents).

## The `Vary` header

`Vary` names request headers that must match for a cached response to be reused. The cache key is effectively `(URL, Vary-named header values)`.

- **`Vary: Accept-Encoding`** — almost always needed when serving compressed and uncompressed variants.
- **`Vary: Accept-Language`** — when serving content negotiated by language.
- **`Vary: Cookie`** — disables shared caching for any response that depends on session cookies. Usually correct for personalized pages; the bug is omitting it.
- **`Vary: Authorization`** — required when you intentionally allow shared caching of a response whose content depends on the `Authorization` header. RFC 9111 blocks shared caching of Authorization-bearing requests by default unless `public`, `s-maxage`, or `must-revalidate` explicitly allows it; once you allow it, you must partition the cache key by user or authorization context.

`Vary` interactions with cache key matter: each unique combination of `Vary`-named header values is a separate cache entry. `Vary: Cookie` on a long-tail-of-cookies response explodes the cache keyspace; usually pair with `Cache-Control: private, no-store` if there's no shared-cache benefit anyway.

## Immutable assets pattern

The cleanest invalidation strategy for static assets:

1. Build emits content-hashed filenames: `app.a1b2c3.js`, `vendor.f9e8d7.css`.
2. HTML references them by hash.
3. Serve with `Cache-Control: public, max-age=31536000, immutable`.
4. New build = new hash = new URL = new cache entry. Old entries age out.

Invalidation is automatic and atomic: the deploy *is* the invalidation. No purge step required. The HTML page itself, which references the hashed assets, is served with a short TTL or `no-cache` so new deploys reach users quickly.

## CDN purge semantics

Most CDNs support two purge modes:

- **Soft purge.** Mark the entry stale; serve until refreshed; refresh on next request or via `stale-while-revalidate`. Cheap; doesn't cause origin spikes.
- **Hard purge.** Evict the entry; next request goes to origin. Can cause origin spike if many keys purge at once.

Tag-based purge (Fastly Surrogate-Key, Cloudflare cache tags) lets one purge invalidate a group. Useful when invalidation logic is "every cached response that involves user 42" — tag responses with `user-42`, purge that tag.

Purges should not be on the correctness path. If correctness depends on a purge succeeding within seconds, the design has the cache too low in the stack — move to event-driven invalidation or shorter TTLs.

## `Cache-Status`

RFC 9211 (June 2022). Standardized response header for caches to annotate their handling of a response. Replaces ad-hoc `X-Cache: HIT` / `X-Served-By` headers. Format:

```
Cache-Status: ExampleCache; hit
Cache-Status: ExampleCDN; fwd=miss; stored, ExampleCache; hit
```

Multiple caches in the chain each append. Valuable for debugging: when a response is unexpectedly stale or unexpectedly fresh, `Cache-Status` shows which cache made the decision. Increasingly supported; check vendor docs.

## Origin signaling and degradation

A few CDN-era patterns worth knowing:

- **`Surrogate-Control`** (RFC 5861 era, supported by Fastly and others) — cache directives that apply only to surrogate caches (CDN), not propagated to clients. Lets origin tell the CDN to cache for 1 hour while the browser caches for 10 seconds.
- **Stale-on-error fallback** via `stale-if-error` plus a long-ish max-stale window — the CDN serves last-good content during origin outages.
- **Cache key normalization** — strip irrelevant query parameters (`utm_*`, session debug flags) from the cache key at the CDN to avoid keyspace fragmentation.

## Common HTTP-caching antipatterns

- `Cache-Control: no-cache` written when `private, max-age=0` was meant, or when `no-store` was meant. The three mean different things.
- Personalized response explicitly made shared-cacheable without `Vary`, per-user keying, or another partitioning mechanism (auth-bypass via shared cache).
- `max-age` without `s-maxage` on a response that the browser should hold briefly but the CDN longer (or vice versa).
- Long-TTL'd `5xx` errors pinning an outage in place.
- `immutable` on a URL that does in fact change (e.g., the HTML page itself).
- Asset URLs without content hashes, requiring purge on every deploy.
- `Vary: User-Agent` (extremely high cardinality; almost never what you want).
- Trusting `X-Cache: HIT` / `X-Served-By` headers as a contract; they're vendor-specific. Use `Cache-Status` if available.

## Sources

- RFC 9111 *HTTP Caching* (June 2022). `https://www.rfc-editor.org/rfc/rfc9111.html`.
- RFC 5861 *HTTP Cache-Control Extensions for Stale Content* (May 2010). `https://www.rfc-editor.org/rfc/rfc5861.html`.
- RFC 8246 *HTTP Immutable Responses* (September 2017). `https://www.rfc-editor.org/rfc/rfc8246.html`.
- RFC 9211 *The Cache-Status HTTP Response Header Field* (June 2022). `https://www.rfc-editor.org/rfc/rfc9211.html`.
- MDN web docs on `Cache-Control`. `https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control`.
- Fastly surrogate-key documentation; Cloudflare cache-tag documentation.
