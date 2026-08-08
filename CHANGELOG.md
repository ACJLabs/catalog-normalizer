# Changelog

All notable, user-visible changes to the Catalog Attribute Normalizer service. Dates are UTC+10
(Melbourne).

## Unreleased

- **Quota is now metered in products, not calls.** The free tier has always been published as "500
  products/month," but the meter incremented once per `normalize_catalog` call while a call may carry up
  to 200 products — so it really allowed far more than the stated number. A call now consumes one unit
  per product in its `products` array, which is what the pricing has always said.
- **A batch that exceeds your remaining balance is refused whole**, with `402` and a body naming
  `required` and `remaining`; nothing is classified and nothing is charged. It is never silently
  truncated, because results are returned positionally and a partial batch would look like a complete
  one. A batch that fails outright still costs nothing; individual `NormalizeError` entries inside an
  otherwise-successful batch are charged, since those products were processed.
- Existing free-tier keys can therefore normalize fewer products per month than the previous
  implementation permitted. The published figure is unchanged — the code now matches it.
- **A request that never reaches the classifier is never charged**, whatever it asked for. Requests
  rejected at the transport layer — an `Accept` header the server can't satisfy, a malformed JSON-RPC
  envelope, a notification with no `id` — now cost zero rather than their stated batch size. Nothing a
  correctly-formed client does changes; this closes a way for malformed traffic to consume an
  allowance without normalizing a single product.
- **JSON-RPC batching (an array of requests in one body) is refused** with `-32600 Invalid Request`.
  MCP removed batching in spec revision 2025-06-18 and the official SDK client does not send it, so
  conforming integrations — including `@acjlabs/catalog-normalizer-client` — are unaffected. Send one
  request object per HTTP request.

## 0.2.0 — 2026-07-26

- **Breaking (output `schema_version` 1.0 → 2.0): `attributes` values are now `{value, provenance}`
  objects, not bare strings.** `provenance` is `"canonicalized"` when the value came from your own
  `raw_attributes` input for that product, or `"extracted"` when the model derived it from the
  title/description and it wasn't in your input — treat `"extracted"` values as a suggestion, the same
  way you'd treat a low-confidence `category_paths` entry. `attributes` is also now constrained to a
  controlled vocabulary (`size`, `color`, `material`, `gender`, `sleeve_length`) — any other key the
  model might have surfaced is dropped, not returned under an arbitrary name.
- **Category classification (google, shopify) is now retrieval-grounded**, not recalled from memory:
  real candidate paths and leaf IDs are retrieved from the current taxonomy files and offered to the
  model as suggestions before it classifies. Measured against a 12-product evaluation set, each product
  checked against both the Google and Shopify taxonomies: **22 of 24 checks (91.7%) exact path + leaf-ID
  matches** — up from 1–2 of 12 checks under the prior zero-shot approach, measured on the earlier
  6-product version of that set against the same two taxonomies (two different set sizes, so read the
  pair as a direction, not a delta; see `0.1.0`'s "best-effort" framing below, now superseded).
  `leaf_id` is still only ever returned when the path independently verifies against the real taxonomy
  file — a fabricated leaf ID could not slip through even under retrieval, since the existing
  verification step is unchanged. Amazon has no comparable public taxonomy reference to retrieve
  against, so it stays best-effort (recall from memory).
- A `leaf_id` that fails re-verification against the real taxonomy file now caps that entry's
  `confidence` at 0.5 (was previously left as whatever the model self-reported, which could read as a
  confident 0.98 next to a `null` leaf_id — a contradiction).
- MCP protocol hygiene pass (spec revision 2025-11-25): `normalize_catalog` now declares tool
  annotations (`readOnlyHint: true`, `openWorldHint: false`) so clients know it's a safe, read-only
  call; the server's `serverInfo` now includes a description and website URL; requests carrying an
  unrecognized browser `Origin` header are now rejected (HTTP 403) per the spec's DNS-rebinding
  protection — normal MCP client calls (no `Origin` header) are unaffected; rate-limited (429)
  responses now include a `Retry-After` header. All additive/hygiene, no breaking change to any
  existing request or response shape.
- **Billing fairness fix**: `initialize` and `tools/list` calls (routine connection setup every MCP
  client does) no longer count against your monthly quota — only an actual `normalize_catalog` call
  does. A failed classification (`isError: true`) also no longer counts as a billed call. No change to
  any request or response shape; this only corrects what was and wasn't billed.
- A request with no API key now gets a `WWW-Authenticate: Bearer realm="catalog-attribute-normalizer"`
  header on its 401 response, per the HTTP Bearer-token convention (RFC 6750) — MCP/HTTP client tooling
  that reads this header can now tell automatically that a bearer token is expected and how to label
  it. No change to the response body or to any other error case.
- **MCP discovery is now public**: `initialize` and `tools/list` no longer require an API key —
  automated scanners and clients can discover the tool and its schema before authenticating. Calling
  the tool itself (`tools/call`) still requires a valid key exactly as before. Also added
  `GET /.well-known/mcp/server-card.json`, a static fallback for clients that can't complete live
  discovery. Separately, `Authorization` headers are now accepted without the `Bearer` prefix (a bare
  key value) alongside the existing `Bearer <key>` form — some gateways forward just the raw key.

## 0.1.0 — 2026-07-21

Initial public release (registry listing to follow the quality work above).

- `normalize_catalog` MCP tool (streamable HTTP) and HTTPS API: batch attribute canonicalization plus
  category mapping to Google, Shopify, and Amazon taxonomies.
- Deterministic size/color canonicalization — the README's worked example is enforced by an automated
  test, so the docs and the service cannot drift apart.
- Anti-fabrication guard: a `leaf_id` is only returned when the category path verifies against the
  real taxonomy file; `null` means "check this suggestion", never an invented ID.
- Structured tool output: the tool declares an output schema and returns machine-readable
  `structuredContent` alongside text.
- Free tier: 500 products/month, self-serve signup, no card required. API keys are shown once at
  signup — store yours immediately.
- Pre-release hardening included in this version: fixed a runtime fault that broke authenticated tool
  calls on the deployed service; per-product error isolation so one bad product never fails a batch.
