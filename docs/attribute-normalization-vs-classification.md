# "Clean product data" is two different problems — most tools only solve one

*A guide for merchant-ops teams and feed-tool developers dealing with catalog data from more than one
source (a Shopify export, a supplier CSV, a marketplace scrape).*

If you've ever asked an AI tool to "clean up" a product catalog and gotten back confident-looking
results that turned out to be wrong in a way you only caught by accident, you've run into a real
distinction most catalog tools blur together: **canonicalizing an attribute you already have** and
**classifying a category you don't** are not the same kind of problem, and they don't fail the same way.

## Cleaning a messy product catalog before a Shopify import — the short answer

If you're staring at a supplier CSV with inconsistent sizes, colors spelled three different ways, and no
category mapping at all, and wondering how to clean a messy product catalog before importing into
Shopify, here's the practical split: **first canonicalize what you already have (sizes, colors, units —
deterministic, one right answer), then map supplier titles to a Google/Shopify taxonomy path
separately (probabilistic — there's real judgment involved, and it can be confidently wrong)**. Treating
both as the same "AI cleanup" step is exactly how confident-looking but wrong output slips into a live
import. The two sections below explain why they're different problems and what to check either way.

## Problem 1: canonicalization — deterministic, and should say so

If a raw catalog says a shirt's size is `"Lg"`, `"lg"`, or `"large"`, there's exactly one right answer:
`"L"`. Same for `"2XL"` vs. `"XX Large"` vs. `"xxlarge"` → `"XXL"`, or `"med"` vs. `"Medium"` → `"M"`.
This is a lookup problem, not a judgment problem — the raw value is already in your data, and the job is
just mapping known synonyms to one controlled vocabulary. A tool that gets this right should get it right
*every time*, deterministically, not "usually."

The honest edge case: **unrecognized input should return nothing, not a guess.** If a size string doesn't
match any known synonym and isn't a plain number, the correct answer is "we don't know," not a
best-effort stab. The same logic applies to color — most color-name variants map cleanly (though even
here there are real judgment calls worth being upfront about: is "khaki" a tan/beige shade or a military
olive-green? Apparel convention says tan, but it's genuinely ambiguous outside that context — a tool that
picks silently and never mentions the call was made is hiding a real decision, not avoiding one).

**Test it yourself:** feed a tool a size or color value you're confident isn't in any standard
vocabulary (a made-up value, a typo, a vendor-specific code). If it returns a confident-looking
canonical value anyway instead of "unrecognized," it's guessing, not canonicalizing.

## Problem 2: category classification — probabilistic, and the failure mode is silent

Assigning a product to a taxonomy path (Google Product Taxonomy, Shopify's category tree) is a genuinely
different kind of problem: there's no lookup table that tells you a "vintage-style denim jacket" belongs
under *Apparel & Accessories > Clothing > Outerwear > Coats & Jackets* rather than three plausible
neighbors. This needs real judgment, which means it's probabilistic — and that means it can be
*confidently wrong* in a way canonicalization can't be.

Here's a concrete example of exactly that failure, found while building against a real taxonomy
reference file (Google's ~5,600-node public taxonomy): asked to classify a winter parka, a capable LLM
returned taxonomy ID `212` with high confidence — a real, valid-looking ID. Checked against the actual
taxonomy file: ID `212` is *Apparel & Accessories > Clothing > Shirts & Tops*. The correct ID for outerwear
is `5598`. The model wasn't confused or hedging — it was **confidently, specifically wrong**, and nothing
about its output looked like an error. It looked exactly like a correct answer.

That's the core problem with trusting classifier output at face value: a fabricated-but-plausible ID is
indistinguishable from a correct one unless you check it against something real. The fix isn't a smarter
prompt — it's **treating "the ID is real" and "the ID is right for this product" as two separate,
independently-checkable claims**, and actually checking the first one deterministically (does this ID
exist in the real taxonomy file? does its real path match what was claimed?) before trusting the second.

**Test it yourself:** pick a handful of taxonomy IDs a tool returns and check them against the taxonomy's
own public reference file (Google publishes theirs; Shopify's is in their API docs). If any returned ID
doesn't exist, or its real path doesn't match what the tool claimed, the tool is fabricating IDs — and if
you haven't checked, you don't know whether it does this occasionally or constantly.

## The tell: does it ever say "not confident enough"?

A classifier that returns a category for every single product, always with a plausible-looking
confidence number, is a classifier that's never been caught guessing — not one that's never wrong. The
honest version of this tool returns a `null` leaf ID alongside a lower confidence score when a
classification is genuinely uncertain, rather than picking the closest-sounding path and hoping.
Canonicalization and classification, done right, look different in the output: reliable fields that are
just *right*, and best-effort fields that are honest about when they're a guess.

## What actually fixes the fabrication problem — and what we measured after fixing it

The winter-parka example above isn't a one-off worth just disclosing — it's fixable, and worth explaining
*how*, not just flagging as a risk. The failure happens because the model is asked to recall a ~5,600-node
taxonomy from memory. The fix is to stop asking it to: retrieve the real candidate paths from the actual,
current taxonomy file first (via embedding similarity against the product's title/description), and let
the model choose among *those*, rather than generate a path from scratch. This doesn't remove the
existing verification step (a claimed leaf ID is still independently checked against the real taxonomy
file, and a leaf ID that fails that check now caps the reported confidence rather than leaving a
confident-looking number next to a `null` ID) — it's a second, complementary fix: retrieval narrows the
model toward real answers *before* verification catches whatever it still gets wrong.

We measured this against a 12-product evaluation set (both Google and Shopify taxonomies, 24 checks
total): **22 of 24 checks (91.7%) exact path + leaf-ID matches** — up from 1–2 of 12 checks before
retrieval-grounding was added, measured on the earlier 6-product version of that set against the same
two taxonomies. The set sizes differ, so the honest reading of the pair is a direction, not a delta;
we state both rather than putting the two fractions side by side. The two remaining misses were an
honest null (correct path, no leaf ID committed — the safe failure mode) and one genuine wrong-path
pick, not a fabrication. We're publishing the sample size and methodology alongside the number on
purpose: at least one competitor in this space advertises a specific accuracy figure (99%+) with no
stated sample size or test methodology attached — a number without a methodology isn't independently
checkable, which is the same complaint this whole guide makes about unverified taxonomy IDs. Amazon has no comparably complete public taxonomy reference file to
retrieve against, so classification there stays best-effort rather than retrieval-grounded.

**Test it yourself:** ask any classifier tool not just "how accurate are you" but "on how many products,
against which taxonomies, and what were the exact misses" — a real methodology survives that question;
a headline percentage with no attached sample usually doesn't.

## Why platform-native tools don't fully solve this either

Shopify and other platforms have started shipping their own AI-assisted attribute tooling. That's
genuinely useful — for data that already lives on that one platform. It doesn't solve the problem most
merchant-ops teams actually have: a catalog assembled from a Shopify export, a supplier's CSV with its
own inconsistent field names, and a marketplace scrape that spells "material" three different ways.
Platform-native tools can't reconcile data they never see. Cross-source consistency needs a tool built
for cross-source input, not one scoped to a single platform's own catalog.

[See how attribute normalization + verified taxonomy mapping compares to category-only classifiers and
enterprise catalog tools →](./comparison.md)

---

*This guide describes the real, disclosed limitations of automated category classification — it isn't a
claim that any tool (including ours) gets every classification right. The honest answer to "how accurate
is this" is: reliable for canonicalization; retrieval-grounded and measured (22 of 24 checks, 91.7% —
12 products × Google + Shopify) for Google and Shopify classification; best-effort, with an honest confidence
signal, for Amazon and any case retrieval can't ground — and worth verifying against the real taxonomy
file rather than trusting either the tool or its stated confidence blindly.*
