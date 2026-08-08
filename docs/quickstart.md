# Connect Catalog Attribute Normalizer to your MCP client — 2-minute quickstart

The onboarding page: how a developer goes from "found it" to "first call". Every fact below is verified
against the live server. Remote server — nothing to install or self-host; you point your MCP client
at a URL and paste a key.

## 1. Get a free key (500 products/month, no card)

```sh
curl -X POST https://catalog-normalizer.acjlabs.com/v1/signup \
  -H "content-type: application/json" \
  -d '{"email":"you@example.com"}'
# -> {"ok":true,"apiKey":"cat_…","warning":"Store this key now — it will not be shown again."}
```

The key is **shown once** — copy it now. It's a plain bearer token; treat it like a password.

## 2. The only three facts any MCP client needs

| | |
| --- | --- |
| **Endpoint** | `https://catalog-normalizer.acjlabs.com/mcp` |
| **Transport** | Streamable HTTP (remote MCP) |
| **Auth** | header `Authorization: Bearer cat_…` (or `x-api-key: cat_…` if your gateway reserves `Authorization`) |

`https://acjlabs-catalog-normalizer.acjlabs.workers.dev/mcp` is an alternate endpoint for the same
deployment. If you connected before 2026-07-28 you already have it configured and nothing needs to change.

MCP client config formats evolve and differ between clients — but every MCP client can connect a remote
server from just those three facts. The per-client snippets below are the current common forms; if yours
isn't listed, plug the three facts into its "add remote/HTTP MCP server" flow.

## 3. Add it to your client

**Claude Code (CLI)**

```sh
claude mcp add --transport http catalog-normalizer \
  https://catalog-normalizer.acjlabs.com/mcp \
  --header "Authorization: Bearer cat_…"
```

**Cursor / Windsurf / VS Code** — in the client's `mcp.json`:

```json
{
  "mcpServers": {
    "catalog-normalizer": {
      "url": "https://catalog-normalizer.acjlabs.com/mcp",
      "headers": { "Authorization": "Bearer cat_…" }
    }
  }
}
```

**Claude Desktop** (stdio-only — bridge to the remote server with `mcp-remote`):

```json
{
  "mcpServers": {
    "catalog-normalizer": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote",
        "https://catalog-normalizer.acjlabs.com/mcp",
        "--header", "Authorization: Bearer cat_…"
      ]
    }
  }
}
```

## 4. Confirm it's connected

Restart your client and the tool **`normalize_catalog`** appears in its tool list. To check the server
directly, list its tools:

```sh
curl -s https://catalog-normalizer.acjlabs.com/mcp \
  -H "Authorization: Bearer cat_…" \
  -H "content-type: application/json" \
  -H "accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

A `200` (a `text/event-stream` reply) listing `normalize_catalog` means the server is reachable and the
tool is present. **`tools/list` is public discovery — it succeeds even without a valid key**, by design, so
registries and MCP clients can index the server before you authenticate. Your key is checked when you
actually **call** the tool: an invalid/missing key returns **401**, an exhausted monthly quota **402**, and
too many requests **429**. So the real "is my key working" test is making a `normalize_catalog` call from
your client (§5).

## 5. Make a call

`normalize_catalog` takes a batch of products and returns, per product and in order: canonicalized
attributes (with `canonicalized` vs `extracted` provenance) and category-path mappings into the taxonomies
you request. Each product needs `title`, `description`, and `raw_attributes` (all required); `target_taxonomies`
is a top-level sibling of `products` (one of `google`, `shopify`, `amazon`):

```json
{
  "products": [
    {
      "title": "Men's Merino Wool Crew Neck Sweater — Charcoal, L",
      "description": "Soft merino wool crew-neck jumper, charcoal grey, size large",
      "raw_attributes": { "size": "L", "color": "Charcoal" }
    }
  ],
  "target_taxonomies": ["google"]
}
```

The differentiator: `google`/`shopify` `category_paths` are **retrieval-grounded against the real
taxonomy file** (measured 22 of 24 checks exact path+leaf_id — a 12-product evaluation set, each
product checked against both the Google and Shopify taxonomies), and each entry's `confidence` +
`leaf_id` (null when not confident) are honest signals — so it catches the plausible-but-wrong IDs a
generic LLM invents. See the [comparison page](comparison-page.md) and [FAQ](faq-seo-draft.md).

## Limits & notes

- **Free:** 500 products/month. See [pricing](pricing.md).
- The key is show-once; if you lose it, sign up again.
- Source & issues: <https://github.com/acjlabs/catalog-normalizer>.
