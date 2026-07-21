# Research: HTTP status → Mule error type mapping (claimed by docs)

Ticket: [What mapping do MuleSoft docs claim? (status → errorType)](https://github.com/andersoal/mule-example-munit-http-error/issues/4)
Part of: [Wayfinder map: HTTP status → Mule error map, proven by MUnit](https://github.com/andersoal/mule-example-munit-http-error/issues/3)

## Connector version pinned

`pom.xml` declares `mule-http-connector` **1.10.5**. The closest published doc-source line is the `http/1.10` branch of `mulesoft/docs-connectors`.

## Headline finding

**MuleSoft's official documentation publishes the *set* of `HTTP:*` error types the Request operation can raise, but does not publish an explicit status-code → error-type mapping table, nor an explicit fallback rule for unmapped codes.** `docs.mulesoft.com` blocks automated fetching (403) from this environment; the GitHub-hosted AsciiDoc source for the same docs (`mulesoft/docs-connectors`, `http/1.10/modules/ROOT/pages/http-request-ref.adoc` and `http-documentation.adoc`) is fetchable and was used as the primary source instead — it mirrors the published docs 1:1 (MuleSoft builds docs.mulesoft.com from this repo).

This is itself the key fact this ticket exists to surface: **the mapping cannot be taken on faith from docs — it has to be proven empirically**, which is exactly what the MUnit matrix (the map's destination) is for. Everything below the error-type list is a **hypothesis**, not a documented fact, unless cited otherwise.

## Documented error types (HTTP Request operation)

Source: `http-documentation.adoc` (`mulesoft/docs-connectors`, `http/1.10` branch), corroborated by [Mule Errors | MuleSoft Documentation](https://docs.mulesoft.com/mule-runtime/latest/mule-error-concept) for the namespace/hierarchy concept.

The Request operation's documented possible errors (verbatim list from the doc source):

```
HTTP:BAD_REQUEST
HTTP:CLIENT_SECURITY
HTTP:CONNECTIVITY
HTTP:FORBIDDEN
HTTP:INTERNAL_SERVER_ERROR
HTTP:METHOD_NOT_ALLOWED
HTTP:NOT_ACCEPTABLE
HTTP:NOT_FOUND
HTTP:PARSING
HTTP:RETRY_EXHAUSTED
HTTP:SECURITY
HTTP:SERVICE_UNAVAILABLE
HTTP:TIMEOUT
HTTP:TOO_MANY_REQUESTS
HTTP:UNAUTHORIZED
HTTP:UNSUPPORTED_MEDIA_TYPE
HTTP:BAD_GATEWAY
HTTP:GATEWAY_TIMEOUT
```

17 named types. Notes on a few:

- `HTTP:SECURITY` vs `HTTP:CLIENT_SECURITY` — distinct types. Per [Mule Errors](https://docs.mulesoft.com/mule-runtime/latest/mule-error-concept), `HTTP:SECURITY` covers this app's own security failures (invalid credentials, expired token); `HTTP:CLIENT_SECURITY` covers a security error produced by the *external* entity the requester called. `HTTP:UNAUTHORIZED` has `MULE:CLIENT_SECURITY` as a parent in the error-type hierarchy (per the same source).
- `HTTP:CONNECTIVITY` and `HTTP:RETRY_EXHAUSTED` are framework-level types "always present because they are common to all connectors" (Mule SDK error-handling docs) — not tied to a status code at all, they fire on connection failure (refused, DNS, reset) rather than any HTTP response.
- `HTTP:PARSING` fires when the response body can't be parsed as the expected MIME type — also not status-code-driven.

## Hypothesis table (to be proven by the MUnit matrix)

No official source publishes this table directly. Rows are the **community-consensus / practitioner-documented convention** (HTTP semantics applied to the named types above), cross-checked against scattered MuleSoft help-portal answers found via search (e.g. [MULE 4, HTTP ERROR STATUS CODE](https://help.mulesoft.com/s/question/0D52T00004mXXK3SAO/mule-4-http-error-status-code) — page blocked from direct fetch, title/snippet only). **Every row not marked "doc-confirmed" is UNVERIFIED and is exactly what the matrix ticket must confirm or correct.**

| Status | Expected errorType | Confidence |
|---|---|---|
| 400 | `HTTP:BAD_REQUEST` | UNVERIFIED (name strongly implies it; no doc citation found) |
| 401 | `HTTP:UNAUTHORIZED` | UNVERIFIED (name match) |
| 403 | `HTTP:FORBIDDEN` | UNVERIFIED (name match) |
| 404 | `HTTP:NOT_FOUND` | doc-confirmed — [Mule Errors](https://docs.mulesoft.com/mule-runtime/latest/mule-error-concept) states explicitly: "an HTTP request fails with an `HTTP:NOT_FOUND` error (for a 404 status code)" |
| 405 | `HTTP:METHOD_NOT_ALLOWED` | UNVERIFIED (name match) |
| 406 | `HTTP:NOT_ACCEPTABLE` | UNVERIFIED (name match) |
| 408 | `HTTP:TIMEOUT` | UNVERIFIED — one help-portal thread title suggests a *lack of response in time* raises `CONNECTIVITY` rather than a literal 408 status; the two mechanisms (real 408 response vs. requester-side timeout) may not be the same code path. This ambiguity is a strong candidate for the special-families ticket. |
| 409 | none named | UNVERIFIED — no named type; presumed fallback |
| 410 | none named | UNVERIFIED — presumed fallback |
| 415 | `HTTP:UNSUPPORTED_MEDIA_TYPE` | UNVERIFIED (name match) |
| 422 | none named | UNVERIFIED — presumed fallback |
| 429 | `HTTP:TOO_MANY_REQUESTS` | UNVERIFIED (name match) |
| 500 | `HTTP:INTERNAL_SERVER_ERROR` | UNVERIFIED (name match) |
| 501 | none named | UNVERIFIED — presumed fallback |
| 502 | `HTTP:BAD_GATEWAY` | UNVERIFIED (name match) |
| 503 | `HTTP:SERVICE_UNAVAILABLE` | UNVERIFIED (name match) — this repo's own README (Option A example) already asserts this mapping informally by using 503 in its worked example, but does not cite a source either |
| 504 | `HTTP:GATEWAY_TIMEOUT` | UNVERIFIED (name match) |
| all other 4xx (402,406-not-covered-above,411,412,413,414,416,417,418,421,423,424,426,428,431,451, unregistered 4xx) | **fallback rule unknown** | **UNVERIFIED — no documented fallback found.** Candidates: falls back to `HTTP:BAD_REQUEST` (most cited informally in community posts), or a generic un-typed `MULE:UNKNOWN`/base `HTTP` error, or the response validator simply doesn't distinguish and the response is treated as a raw failure with no specific subtype. **This is the single most important thing the matrix must determine** — it decides how big the "equivalence class" rows actually are. |
| all other 5xx (506,507,508,509,510,511, unregistered 5xx) | **fallback rule unknown** | **UNVERIFIED**, same as above but for the 5xx fallback, which may differ from the 4xx fallback. |
| 1xx (100,101,102,103) | unknown — may never reach the flow as an error at all | UNVERIFIED. No doc found describing interim-response handling in the Request operation; these are typically consumed by the transport layer before the connector's response validator runs. Flagged for the special-families ticket. |
| 2xx (all) | no error — success path | doc-confirmed by definition (default validator "throws an error when the status code is 400 or above" per `http-request-ref.adoc`) |
| 3xx (all) | no error if followed; behavior if not followed is unknown | Partially doc-confirmed: `followRedirects` defaults to `true` per `http-request-ref.adoc` ("Specifies whether to follow redirects or not. Default value is true"). What happens with `followRedirects=false` and a 3xx response (does it throw, or return the 3xx as a non-error response attributes?) is UNVERIFIED. Flagged for the special-families ticket. |
| connection refused / DNS failure / TCP reset (no status code at all) | `HTTP:CONNECTIVITY` | doc-confirmed — [HTTP Connector Reference](https://docs.mulesoft.com/http-connector/1.8/http-documentation) snippet: "HTTP:CONNECTIVITY... Indicates that there was a problem establishing a connection." |
| exhausted retries after a transient failure | `HTTP:RETRY_EXHAUSTED` | doc-confirmed by definition — framework-level type, common to all connectors per Mule SDK docs |

## Sources consulted

- [mulesoft/docs-connectors — http/1.10/modules/ROOT/pages/http-request-ref.adoc](https://github.com/mulesoft/docs-connectors/blob/latest/http/1.10/modules/ROOT/pages/http-request-ref.adoc) (fetched raw)
- [mulesoft/docs-connectors — http/1.10/modules/ROOT/pages/http-documentation.adoc](https://github.com/mulesoft/docs-connectors/blob/latest/http/1.10/modules/ROOT/pages/http-documentation.adoc) (fetched raw — source of the 17-type list)
- [mulesoft/docs-connectors — http/1.10/modules/ROOT/pages/http-error-status-reason-phrase-task.adoc](https://github.com/mulesoft/docs-connectors/blob/latest/http/1.10/modules/ROOT/pages/http-error-status-reason-phrase-task.adoc) (fetched raw — listener-side manual status setting, not requester-side mapping; confirmed not relevant to this question)
- [Mule Errors | MuleSoft Documentation](https://docs.mulesoft.com/mule-runtime/latest/mule-error-concept) (title/snippet via search; direct fetch blocked 403)
- [MULE 4, HTTP ERROR STATUS CODE (MuleSoft Help Portal)](https://help.mulesoft.com/s/question/0D52T00004mXXK3SAO/mule-4-http-error-status-code) (title/snippet only; direct fetch blocked 403)
- `docs.mulesoft.com` direct URLs (`http-connector/1.10/http-request-ref`, etc.) all return HTTP 403 from this environment's WebFetch — could not be fetched directly. Community mirrors and the GitHub doc-source repo were used as substitutes for the primary claims; anything not found there is marked UNVERIFIED above.
- `mulesoft/mule-http-connector` GitHub repository — the connector's own Java source (`HttpError.java` or equivalent) was searched for but not located; the connector implementation does not appear to be in a public repository at a fetchable path. If it becomes reachable later, it would be the strongest possible primary source for the fallback rule.

## What this means for the map

The fallback rule for unmapped 4xx and unmapped 5xx codes is the **highest-value unknown** in the whole matrix — it alone determines how many of the ~63 rows collapse into "same as their class's fallback" versus need individual named-type verification. The MUnit matrix (blocked on the parameterization ticket) should test at least one representative unmapped code per class (e.g. `410`, `422` for 4xx; `501`, `508` for 5xx) early, since the answer reshapes the rest of the ticket set.
