# Inline Redirects: Explainer

Status: early draft, not on any standards track. Author: Mateusz Krzeszowiak (Shopify).

Feedback is welcome. Please [open an issue](https://github.com/krzksz/inline-redirects/issues).

## Problem

When a user navigates to a URL that redirects to another URL on the same origin, the browser makes two requests before it can start parsing HTML:

1. The client requests `GET /old`.
2. The server responds with a `3xx` and `Location: /new`.
3. The client requests `GET /new`.
4. The server responds with `200` and the HTML.

The server that answers step 2 is the same origin that answers step 4. In many deployments it already knows what `/new` will return. The client still waits for a full roundtrip, plus its own redirect handling, before the first byte of the document arrives.

Same-origin redirects are common and often unavoidable for the site: locale or market prefixes (`/products/boots` to `/en-ca/products/boots`), URLs that changed after a migration or a rename, canonical URL enforcement, and old links that live on other sites, in ads, and in email.

## Goals

- Let a server deliver the destination document in the response to the redirected request, so the browser needs no second request.
- Give the committed document the same URL, history entry and state it would have after a normal redirect.
- Break nothing for clients and intermediaries that do not support the feature. They see a normal redirect.

## Non-goals

- Cross-origin redirects, including same-site ones such as `http` to `https` or apex to `www`.
- Subresource requests and `fetch()`. The mechanism is defined generically, but this proposal only specifies document navigations.
- Non-`GET` navigations, including Post/Redirect/Get. See Future work.

## Evidence

For eligible navigations (between 3-4% of page navigations on Shopify storefronts), collapsing the redirect into one response would save the following time before the first byte of the destination HTML:

| | p50 | p75 | p90 |
| --- | ---: | ---: | ---: |
| Redirect duration today (`redirectEnd - redirectStart`) | 260 ms | 427 ms | 663 ms |
| Time saved by an inline redirect | 109 ms | 168 ms | 248 ms |
| Saved share of the redirect duration | 50% | 70% | 81% |

The saving is the client turnaround (one client-to-edge roundtrip plus the browser's redirect handling) and one edge-to-origin leg. Server processing of the redirect and connection setup on the first request remain.

Method: mobile real-user navigations with exactly one same-origin redirect hop. The time saved is measured per navigation from the CDN timestamps.

## Proposal

One new HTTP response field, `Inline-Redirect`. It is a Structured Fields Item (RFC 9651) whose value is a boolean. No parameters are defined, and a recipient ignores parameters it does not know. A value of `?0` is treated like an absent field.

A server sends `Inline-Redirect: ?1` on a `301`, `302`, `303`, `307` or `308` response whose content is the representation of the `Location` target.

```http
GET /products/old-boots HTTP/2
Host: shop.example

HTTP/2 301 Moved Permanently
Location: /en-us/products/boots
Inline-Redirect: ?1
Content-Type: text/html; charset=utf-8
Cache-Control: private, max-age=3600
Content-Security-Policy: default-src 'self'

<!doctype html>...the document for /en-us/products/boots...
```

A client that does not know the response header sees an ordinary redirect and follows `Location`. A `3xx` response is already allowed to carry content (RFC 9110), and clients ignore it today. The cost is the bandwidth for content that is discarded.

### Server requirements

A server sends `Inline-Redirect: ?1` only when all of these hold:

- `Location` resolves to a URL with the same origin as the request URL, and differs from the request URL.
- The content is what the server would return with status `200` for a `GET` of the `Location` URL, given the same request headers.
- Apart from `Location`, `Inline-Redirect`, and HTTP caching headers (`Cache-Control`, `Expires`, `Age`), every response header is the one the server would send on that `200` response. The response carries no other headers that are meant for the redirect alone.

A chain of same-origin redirects can be collapsed. The server sets `Location` to the final URL and sends that URL's document. Loop detection, hop limits, and timeouts for internal chain resolution are the server's responsibility. If the server detects a loop, exceeds its hop limit, or times out, it should fall back to returning a plain `3xx` without `Inline-Redirect` (letting the client apply its normal redirect limit) or return an appropriate `5xx` error.

The server decides when to inline. A client that does not support the feature discards the content, so a wrong decision wastes bandwidth and server work and breaks nothing. A server can limit the waste by inlining only for browsers it knows support the feature, for example from the brand and version in `Sec-CH-UA`.

### Client processing

This proposal applies to `GET` document navigations: top-level, nested, and navigational prefetch and prerender. An implementation may support a subset of these.

On a `301`, `302`, `303`, `307` or `308` response to such a navigation, the client treats the response as an inline redirect when all of these hold:

1. The response carries `Inline-Redirect: ?1` and a valid `Location`.
2. The `Location` URL has the same origin as the request's current URL, and is not equal to it.
3. The response has content. A response that ends with no content is a normal redirect.
4. Every check the client applies when it follows a redirect to the `Location` URL passes. This includes URL reputation checks, enterprise and extension URL rules, and the initiator's Content Security Policy. The client can start these checks when the response headers arrive and buffer the content until they finish.
5. The cookies the client would send to the `Location` URL are the same set it sent on this request.
6. No service worker with a fetch handler would handle a request for the `Location` URL, and none handled this request.

If any condition fails, the client ignores `Inline-Redirect`, discards the content, and follows `Location` as it does today. On HTTP/2 and HTTP/3 it can reset the stream.

If all conditions hold, the client proceeds as if it had followed the redirect and received the response from the `Location` URL:

- The response URL is the `Location` URL. The request's fragment carries over unless `Location` has one, as with any redirect.
- The destination document's response status is `200`. The client still knows the status code it received and can use it for redirect caching and in developer tools.
- Apart from `Location`, `Inline-Redirect`, and HTTP caching headers (`Cache-Control`, `Expires`, `Age`), all response headers apply to the destination document. There is no separate redirect response to which headers such as `Referrer-Policy`, `Cross-Origin-Opener-Policy` or `Set-Cookie` apply first. Defaults that depend on the response URL, such as a cookie's default path, use the `Location` URL.
- The document's URL, base URL and origin, the address bar, and the session history entry are set from the `Location` URL before any content is parsed.
- History behaves as for a normal redirect. The original URL does not get its own entry.

### Caching

- The client caches the redirect as usual, based on the status code and the response's caching headers (`Cache-Control`, `Expires`, `Age`, or normal heuristics for `301` and `308` when no explicit freshness is present).
- The client does not store the inline response body in its HTTP cache, under either URL. A later navigation to the `Location` URL fetches and caches it as usual.
- The client uses an inline response even when it already holds a stored copy of the destination. The inline response is at least as fresh.
- A cache that does not know this feature stores what it sees, a `3xx` with content under the original URL, according to the `3xx`'s caching headers. Temporary redirects such as `302` are therefore not cached unless the server marks them cacheable. It may serve a stored response to any client; a client that does not support inline redirects follows `Location` and ignores the content, which costs bandwidth only. Servers behind an unaware shared cache that serve dynamic HTML should use `private` or `no-store` so the shared cache does not hold onto the attached body.
- A shared cache that knows the feature should not store the inline response. It can produce one itself from its stored redirect and its stored copy of the destination.

### Timing and script-visible state

After commit, `document.URL`, `location`, `document.referrer` and `Referer` on subresource requests are what they would be after a normal redirect.

Navigation Timing reports the navigation as direct: `redirectCount` is 0, and `redirectStart` and `redirectEnd` are 0. Navigation Timing needs no change. A site that wants to count inline redirects can add its own `Server-Timing` entry to inline responses.

## Alternatives considered

**`Content-Location`.** On a `200` response, RFC 9110 §8.7 explicitly states that `Content-Location` is representation metadata and not a replacement for the target URI. A `200` response also fails open: unaware browsers, crawlers, and HTTP clients stay on the original URL instead of redirecting. Existing sites already emit `Content-Location` on `200` navigations in ways that would break if browsers navigated to it. In the November 2026 HTTP Archive mobile crawl, 0.032% of HTML main documents carry `Content-Location` (94.5% with `Vary`), including internal portal state URLs, Apache `MultiViews` filenames (such as `index.html.en` or `*.php`), internal backend paths, and malformed relative hostnames. Using `Content-Location` equal to `Location` on a `3xx` response avoids those problems (0 occurrences in top-100k `3xx` responses), but duplicating the full URL introduces a mismatch edge case and leaves no room for Structured Field parameters.

**A `200` response with a header that replaces the URL.** A client or crawler that does not know the header would treat the content as belonging to the original URL. Reusing `3xx` means every unaware party follows `Location`. The proposal is in effect this alternative with a status code that fails safely.

**A new `3xx` status code.** A client treats an unknown `3xx` as `300` (RFC 9110). Fetch follows only `301`, `302`, `303`, `307` and `308`, so an unaware browser would not redirect.

**A request header that signals support.** A client could send a header such as `Sec-Inline-Redirect: ?1`, and a server would inline only on request. Clients that do not support the feature would then never receive the content. It adds a header to every navigation request, and it would be costly to extend to subresources. A client hint requested with `Accept-CH` avoids the per-request cost but misses the first navigation to a site. Without any request signal, the worst case is wasted bandwidth and server work.

**Internal rewrite with `history.replaceState()`.** The server returns the destination with `200` at the original URL, and script fixes the URL. The address bar, base URL, relative URL resolution, preload scanning and `Referer` all see the original URL until script runs, and clients without script never see the new one.

**Internal rewrite with `rel="canonical"`.** Whether sent via `<link rel="canonical">` or the `Link: rel="canonical"` HTTP header (RFC 8288), this addresses indexing only. Browsers do not update the address bar, `document.URL`, base URL, or history, and unaware non-browser clients do not redirect.

**Speculation Rules and prefetch.** These help when the site controls the link and the browser predicts the click. They do not help the first navigation from another site.

**`103 Early Hints` on the redirect.** It can warm connections and subresources but does not deliver the document.

**HTTP/2 Server Push.** It could deliver the destination alongside the redirect. It has been removed from Chromium.

## Security considerations

**Origin.** The destination must have the same origin as the request URL. A server can already put any content at any URL of its own origin, so an inline redirect gives it no new authority over URLs.

**Shared-path origins.** Some origins host different parties under different paths. A party that controls responses for one path could use an inline redirect to show its content under another path's URL. That is already possible with `history.replaceState()`, since path is not a security boundary. Because clients do not store the inline body under the destination URL, nothing persists beyond a normal redirect.

**Path-scoped state.** Cookies with a `Path` and service worker scopes can differ between the two URLs. Conditions 5 and 6 make the client fall back to a normal redirect in those cases, so the destination never renders with state it would not have had. The client does the cookie comparison because a server cannot see cookies it did not receive.

**URL-keyed checks.** Condition 4 ensures that no check keyed on the destination URL is skipped because the client made no request for it.

**Document policies.** The destination document's `Content-Security-Policy`, `X-Frame-Options`, `Cross-Origin-Opener-Policy`, `Cross-Origin-Embedder-Policy` and `Referrer-Policy` come from the inline response and are enforced as for any `200` document response.

**Intermediaries.** An intermediary that drops the content of a `3xx` produces a response with no content, which is a normal redirect under condition 3. One that truncates the content produces the same failure as a truncated `200`.

## Privacy considerations

The proposal adds no request header and exposes nothing new about the client. Because the destination is same-origin, no party receives information it would not get from the normal redirect. The server receives one request instead of two, so its logs have no separate entry for the destination URL.

## Open questions

1. **Service workers.** The current rule excludes any navigation where a service worker with a fetch handler is involved. A later version could let a worker see the inline response, or a synthetic fetch event for the destination.

## Future work

- `303` after `POST`, so Post/Redirect/Get takes one roundtrip.
- Subresources and `fetch()` with `redirect: "follow"`.
- Destination responses other than `200`, carried as a parameter on `Inline-Redirect`.
- Caching the inline response body under the `Location` URL.

## Standards venues

- IETF HTTP Working Group: semantics of content on a `3xx`, caching rules, and registration of the field.
- WHATWG Fetch and HTML: redirect handling, navigation commit, and service worker interaction.
