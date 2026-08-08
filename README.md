# Catalog Attribute Normalizer

**Taxonomy-grounded catalog attribute normalizer** — verified against the real Google Product Taxonomy, so
it catches the plausible-but-wrong category IDs a generic LLM invents. Messy multi-source catalogs — titles,
descriptions, images — in; consistent attributes, units, and grounded category mappings out. For
merchant-ops teams and feed-tool developers wrangling a Shopify export, a supplier CSV, and a marketplace
scrape that each spell "size" or "material" differently.

## What it does

- Accepts a batch of raw products (`title`, `description`, `raw_attributes`), returns normalized
  attributes (canonicalized sizes/colors/units) plus category mappings for whichever taxonomies you
  request (Google Product Taxonomy, Shopify, Amazon).
- Works as an MCP server (`normalize_catalog` tool) or a plain HTTPS API.

### Example (MCP tool call)

```json
{
  "tool": "normalize_catalog",
  "input": {
    "products": [{ "title": "...", "description": "...", "raw_attributes": { "size": "Lrg", "Color": "Navy Blue" } }],
    "target_taxonomies": ["google", "shopify"]
  }
}
```

```json
{
  "results": [
    {
      "schema_version": "2.0",
      "source_title": "...",
      "category_paths": {
        "google": { "path": ["Apparel & Accessories", "Clothing", "Shirts & Tops"], "leaf_id": null, "confidence": 0.9 },
        "shopify": { "path": ["Apparel & Accessories", "Clothing", "Tops"], "leaf_id": "aa-3-1", "confidence": 0.85 }
      },
      "attributes": {
        "size": { "value": "L", "provenance": "canonicalized" },
        "color": { "value": "navy", "provenance": "canonicalized" }
      }
    }
  ]
}
```

`attributes` is keyed by a controlled vocabulary (`size`, `color`, `material`, `gender`, `sleeve_length` —
unrecognized keys are dropped, not passed through under a model-chosen name). Each value's `provenance`
is `"canonicalized"` when it came from your own `raw_attributes` input for that product (deterministic
cleanup only, no recall) or `"extracted"` when the model inferred it from the title/description and it
wasn't in your input — treat `"extracted"` values as a suggestion, the same way you'd treat a
low-confidence `category_paths` entry.

**Attributes are reliable by construction** — canonicalizing values already in your input, not recall.
**Category classification is retrieval-grounded**, not recalled from memory: candidates are retrieved
from the real, current Google and Shopify taxonomy files and offered to the model as suggestions, so a
`leaf_id` almost always names a node that actually exists — measured against a 12-product evaluation set,
each product checked against both the Google and Shopify taxonomies: **22 of 24 checks (91.7%) exact
path + leaf-ID matches**. Still treat `confidence` and a `null` `leaf_id` as "worth a quick check," not
a guarantee — a `leaf_id` is only ever returned when the path independently verifies against the real
taxonomy file, so a `null` there is an honest "check this" signal, never a fabricated ID. Amazon has no
comprehensive public taxonomy reference file to retrieve candidates from, so it stays best-effort
(recall from memory) rather than retrieval-grounded.

## Pricing

The free tier is self-serve today; paid plans are opening soon.

- **Free** — 500 products/month, no card required. Self-serve today.
- **Pay-as-you-go** *(opening soon)* — $0.01/product, no minimum.
- **Pro** *(opening soon)* — $29/month for 5,000 products (~$0.0058/product effective).

Want a paid plan now?
[Register your interest](https://catalog-normalizer-signup.acjlabs.com/#pro-intent) and we'll set you up first.

**What counts as one unit:** one product, not one call. A `normalize_catalog` call carrying 40 products
uses 40 of your monthly allowance; batching is a convenience, not a discount. A batch is all-or-nothing
— if it is larger than your remaining balance, the whole call is refused with `402` (the body names
`required` and `remaining`) and nothing is classified, so you are never billed for a partial result
that looks complete. A call that fails outright costs nothing.

## Getting a key

The MCP server is live at `https://catalog-normalizer.acjlabs.com/mcp` (the alternate
`https://acjlabs-catalog-normalizer.acjlabs.workers.dev/mcp` address reaches the same deployment and
keeps working, so existing configurations need no change). Free-tier keys
self-serve — `POST /v1/signup` with `{ "email": "you@example.com" }` returns your key directly in the
response, good for 500 products/month, no card required. **Store it immediately — it is shown once and
cannot be recovered if lost** (re-signup for a new one). See
**<https://acjlabs-catalog.pages.dev>** for the same steps plus pricing and a comparison against
alternatives. Paid keys provision the same way via a
one-time claim link once Pro/pay-as-you-go billing is live.

## Connecting it to an MCP client

With Claude Code:

```bash
claude mcp add --transport http catalog-normalizer \
  https://catalog-normalizer.acjlabs.com/mcp \
  --header "Authorization: Bearer YOUR_API_KEY"
```

Any other MCP client that supports a remote HTTP server with a custom header (Cursor, Cline, VS Code,
etc.) works the same way: point it at the URL above with an `Authorization: Bearer YOUR_API_KEY`
header. A gateway or scanner that can only forward the raw key value (no `Bearer` prefix) also works —
both forms authenticate. Tool discovery (`initialize`/`tools/list`) doesn't require a key at all; only
calling `normalize_catalog` does.

See [docs/quickstart.md](docs/quickstart.md) for a full copy-paste walkthrough (getting a key, per-client
configs, confirming the connection with `curl`) if you'd rather follow one linear guide.

## npm client

Prefer a typed function over hand-rolling MCP JSON-RPC calls?
[`@acjlabs/catalog-attribute-normalizer-client`](https://www.npmjs.com/package/@acjlabs/catalog-attribute-normalizer-client)
(live on npm) wraps the `normalize_catalog` tool call:

```sh
npm install @acjlabs/catalog-attribute-normalizer-client
```

```ts
import { createCatalogNormalizerClient } from "@acjlabs/catalog-attribute-normalizer-client";

const client = createCatalogNormalizerClient({
  baseUrl: "https://catalog-normalizer.acjlabs.com",
  apiKey: "...",
});
const results = await client.normalizeCatalog(products, ["google", "shopify"]);
```

## Why not just use a category classifier?

Category-classification APIs tell you *what* a product is. They don't touch the messier problem:
standardizing *attributes* across sources that each spell them differently. The vendors that do take on
the broader job are enterprise sales-led — demo request, annual contract, no self-serve signup and no
public price. See the [full comparison](docs/comparison.md), or read
[why "clean product data" is actually two different problems](docs/attribute-normalization-vs-classification.md)
(canonicalization vs. classification, and why they fail differently).

## Source availability & support

This repository hosts the documentation for the hosted service. The service implementation is not open source. Bug reports and feature requests are welcome in this repo's Issues; you can also reach us at <contact@acjlabs.com>.
