# Kemppi Search Prototype — Developer Documentation

## Overview

The prototype is a single self-contained HTML file that implements a client-side search experience for kemppi.com. It covers three interaction states — empty (previous searches), rich preview panel (while typing), and full results page — using a combination of deterministic prefix matching and fuzzy search via Fuse.js.

All product data is embedded in the file as JavaScript arrays. There is no backend call at runtime; product images are fetched on demand from Kemppi's Content Hub CDN.

---

## Product Hierarchy

The catalogue follows a three-tier structure:

```
Brand Group
  └── Product Family
        └── Individual Product Model
```

**Brand Groups** (`brandGroup` field) are the top-level groupings and map to the major product line names: `minarc`, `master`, `x5`. They exist only as a field on Family objects — there is no separate Brand Group array.

**Product Families** (`FAMILIES` array) represent ranges of related machines sharing a name and welding process. Each family has:

| Field | Description |
|---|---|
| `id` | Kebab-case slug, e.g. `minarc-t-dc` |
| `brandGroup` | Parent brand group ID |
| `name` | Display name, e.g. `Minarc T DC` |
| `weldType` | Process label, e.g. `TIG DC` |
| `url` | Relative URL to the family page |
| `desc` | Short marketing description |

**Individual Products** (`PRODUCTS` array) are specific purchasable models. Each product has:

| Field | Description |
|---|---|
| `id` | Kebab-case slug, e.g. `minarc-t-223-dc-gm` |
| `familyId` | Links to a `FAMILIES` entry |
| `name` | Full model name |
| `sku` | Product SKU code |
| `category` | Type label: `Power source`, `Welding machine`, or `Wire feeder` |
| `desc` | Short specification summary |

---

## Where the Data Comes From

### Product and family data (Algolia)

The product names, SKUs, descriptions, and family structure were sourced from the Kemppi Algolia search backend during the prototype build. The relevant Algolia details are:

- **Application ID:** `0L0VF8KN1P`
- **Public API key:** `d88f73217aa507303fcf78ad929f261c`
- **Endpoint:** `https://0l0vf8kn1p-dsn.algolia.net`
- **Indices used:**
  - `products` — 1,817 individual product records
  - `product_families` — 54 family records
  - `all_query_suggestions` — 1,518 query suggestion records

To query a live Algolia index (e.g. to refresh data in future):

```js
fetch('https://0L0VF8KN1P-dsn.algolia.net/1/indexes/products/query', {
  method: 'POST',
  headers: {
    'X-Algolia-Application-Id': '0L0VF8KN1P',
    'X-Algolia-API-Key': 'd88f73217aa507303fcf78ad929f261c',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ query: 'minarc', hitsPerPage: 10 })
})
```

Algolia records store product content inside a nested `data.fields` object (Contentful structure). The relevant path for image data is:

```
hit.data.fields.mainImage.en.fields.damImageUrl.en
// e.g. "https://cdn-kemppi.contenthub.fi/cdn-cgi/image/format=auto/api/v1/cdn/2157016"
```

### Product images (Content Hub CDN)

Images are served from Kemppi's Cloudflare-proxied Content Hub CDN. The URL pattern supports on-the-fly image transformation:

```
https://cdn-kemppi.contenthub.fi/cdn-cgi/image/format=webp,fit=scale-down,width={w},quality={q}/api/v1/cdn/{numericId}
```

The numeric ID is the Content Hub asset ID (different from the alphanumeric Contentful asset ID also present in Algolia records — use the `damImageUrl` field, not the Contentful `sys.id`).

The prototype uses one representative image per family, with per-product overrides for cases where models look meaningfully different (e.g. X5 wire feeders vs power sources). Image resolution is handled by `getItemImg()`:

```js
function getItemImg(item, size = 200) {
  // 1. Check per-product override first
  // 2. Fall back to family image
  // 3. Fall back to family's own id (for Family-type items)
  const url = PRODUCT_IMAGES[item.id]
           || FAMILY_IMAGES[item.familyId]
           || FAMILY_IMAGES[item.id];
  if (!url) return null;
  return url.replace(/width=\d+/, `width=${size}`);
}
```

Current numeric IDs in use:

| Family | Content Hub ID |
|---|---|
| Minarc | 2157016 |
| Minarc Evo | 2156866 |
| Minarc T (AC/DC) | 4527719 |
| Minarc T DC | 9863343 |
| MinarcTig | 2160438 |
| MinarcMig Auto | 4519038 |
| Master M | 2185466 |
| Master M 205/323 | 2182268 |
| X5 FastMig (power source) | 2191605 |
| X5 FastMig (wire feeder) | 2192230 |

---

## Search State Machine

The overlay has three named states managed by `showState(state)`:

| State | Element shown | When |
|---|---|---|
| `empty` | `#state-empty` | Overlay opens or input is fully cleared |
| `autocomplete` | `#autocomplete-wrap` | User is typing before any search is committed |
| `preview` | `#state-preview` | A search has been committed (Enter / suggestion click) |

### `searchCommitted` flag

A boolean `searchCommitted` controls whether the dropdown or the live preview is shown while the user types.

- **`false` (default):** typing shows the autocomplete dropdown — the normal pre-search state.
- **`true`:** set when `commitSearch()` is called. While true, further typing triggers a debounced live-update of the preview panel instead of showing the dropdown. The dropdown is fully suppressed.

`searchCommitted` resets to `false` when:
- The × clear button is clicked
- The user empties the field via backspace (detected in `updateSearchUI` when `query === ''`)
- The overlay is opened fresh via `openSearch()`

### Inline ghost text

While `searchCommitted` is true, the top autocomplete suggestion is rendered as grey inline text directly inside the search bar — the portion the user has already typed is rendered with `color: transparent` so only the completion tail is visible.

```
User typed:  "minarc t"
Ghost shows: "minarc t 223 ACDC GM"  ← "minarc t" invisible, " 223 ACDC GM" grey
```

Pressing **Tab** accepts the completion: the ghost text is written into the input and a new live search fires. The ghost is cleared on blur, clear, or when no suggestion starts with the current query.

Implementation: `#search-ghost` is a `position: absolute` div layered behind the `<input>` inside `.search-input-wrap`. Both elements share the same `font-size` and `font-family` so the text aligns pixel-perfectly.

### Live search in committed state

`updateSearchUI(query)` — called debounced on every keystroke — behaves differently depending on `searchCommitted`:

```
searchCommitted = false  →  render autocomplete dropdown
searchCommitted = true   →  call search(query), re-render preview panel silently
```

This means once a user has committed a search, the preview results update in real time as they edit the query, without needing to press Enter again. Enter from the preview state navigates to the full results page.

---

## The CATALOG Object

At runtime, `FAMILIES`, `PRODUCTS`, and `CONTENT_ITEMS` are merged into a single flat `CATALOG` array that Fuse.js indexes. Each item gets a `type` field added during the merge:

```js
const CATALOG = [
  ...FAMILIES.map(f  => ({ ...f, type: 'family',  category: 'Product family', family: f.name,              sku: '' })),
  ...PRODUCTS.map(p  => ({ ...p, type: 'product',                              family: familyMap[p.familyId]?.name || '' })),
  ...CONTENT_ITEMS,   // already have type: 'article' | 'faq' | 'doc'
];
```

`PRODUCTS_FAMILIES` is a filtered subset used for the left column of the preview panel (products and families only, no content).

---

## Image Rendering

### Blend mode

All product images are rendered with `mix-blend-mode: multiply`. Because the image containers have a light background (`#f5f5f5` or white), this makes the white areas of product photos transparent — the product appears to float without a visible white box behind it.

Applied to:
- `.pc-img img` — preview panel product/family cards
- `.res-grid-img img` — results page product grid
- `.res-list-thumb img` — results page list rows (families, content, docs)

### Container sizing

| Element | Size | Shape |
|---|---|---|
| `.pc-img` | 112 × 112 px (mobile) | Square |
| `.res-grid-img` | 100% width, `aspect-ratio: 1/1` | Square |
| `.res-list-thumb` | 96 × 96 px | Square |

`.res-grid-img` uses a CSS container query: when the container is wider than 180px (desktop 6-column grid), the image inside is constrained to 72% of the container size via `max-width`/`max-height`, giving it visible padding on all sides.

### SVG placeholders

Items without a Content Hub image fall back to inline SVG icons rather than a blank container. Three symbols are defined in a hidden `<svg>` sprite at the top of `<body>`:

| Symbol | Used for |
|---|---|
| `#icon-product` | Products in the preview panel and grid (orange box/cart shape) |
| `#icon-family` | Product families in list rows (2×2 grid of squares) |
| `#icon-doc` | Content and user documentation in list rows (stacked pages) |

`renderResListRow()` selects the icon based on `item.type`:
```js
const iconId = item.type === 'family' ? 'icon-family' : 'icon-doc';
```

---

## Fuzzy Search (Fuse.js)

**Library:** Fuse.js 7.0.0, loaded from cdnjs CDN.

Three separate Fuse instances are created at startup, each indexed against a different slice of the data:

| Instance | Dataset | Used for |
|---|---|---|
| `fuseProductsFamilies` | `PRODUCTS_FAMILIES` | Preview panel left column |
| `fuseContent` | `CONTENT_ITEMS` | Preview panel right column |
| `fuseAll` | `CATALOG` | Full results page + autocomplete fallback |

### Search configuration

```js
const SEARCH_CONFIG = {
  keys: [
    { name: 'name',     weight: 3.0 },   // Product/family name — highest priority
    { name: 'sku',      weight: 2.0 },   // SKU code search
    { name: 'family',   weight: 1.5 },   // Family name on product records
    { name: 'weldType', weight: 1.2 },   // Welding process label
    { name: 'category', weight: 1.0 },   // Product category label
    { name: 'desc',     weight: 0.5 },   // Description text — lowest priority
  ],
  threshold:          0.35,   // 0 = exact match only, 1 = match anything
  distance:           200,    // How far from start of string a match can be
  includeScore:       true,
  minMatchCharLength: 2,
  includeMatches:     true,
};
```

**Tuning notes:**
- Lowering `threshold` (e.g. to 0.2) makes search stricter — fewer typo-tolerant matches.
- Raising `threshold` (e.g. to 0.5) returns more results but increases noise.
- `name` weight at 3.0 ensures product title matches always outrank description matches.
- `sku` at 2.0 means typing a SKU code like `MIT223DCGM` reliably surfaces the correct product.

### Debouncing

Search is debounced at **180ms** (`DEBOUNCE_MS`). The preview panel is shown only after the user has typed at least `RICH_PREVIEW_THRESHOLD` characters (default: 2).

---

## Autocomplete Suggestion Logic

The autocomplete dropdown uses a deliberate two-tier hierarchical ordering rather than pure fuzzy scoring, to ensure family-level suggestions always appear before individual model numbers.

### Tier 1 — Family matches (deterministic prefix/contains)

Families whose name starts with or contains the query are collected first. Within this tier, exact prefix matches rank above contains-matches, and shorter names rank above longer ones.

```
Query: "minarc"
→ Minarc          (prefix match, shortest)
→ Minarc Evo      (prefix match)
→ Minarc T        (prefix match)
→ Minarc T DC     (prefix match)
→ MinarcTig       (contains)
→ MinarcMig Auto  (contains)
```

### Tier 2 — Product matches

Products whose name or SKU starts with or contains the query are collected next. Within this tier, products belonging to a family matched in Tier 1 rank above others, then prefix matches rank above contains-matches.

```
Query: "minarc t 223"
→ (no family matches at this specificity — families exhausted in Tier 1)
→ Minarc T 223 ACDC GM      (product, prefix, family matched)
→ Minarc T 223 ACDC GM AU   (product, prefix, family matched)
→ Minarc T 223 DC GM        (product, prefix, family matched)
→ …
```

### Slot allocation

The dropdown shows at most `AUTOCOMPLETE_LIMIT` suggestions (default: 6). Up to 4 slots are reserved for family matches; remaining slots go to products. If fewer than 6 total results are found via prefix/contains matching, Fuse.js fuzzy search fills the remaining slots (providing typo tolerance for the tail of the list).

### Fuzzy fallback

The fuzzy fallback uses the `fuseAll` instance and deduplicates against items already in the list. This means a misspelling like "minarc tiq" will still surface `MinarcTig` even though it doesn't prefix-match.

---

## Display Limits

| Constant | Default | Controls |
|---|---|---|
| `AUTOCOMPLETE_LIMIT` | 6 | Max items in the dropdown |
| `PREVIEW_PRODUCT_LIMIT` | 3 | Products shown in the preview panel left column |
| `PREVIEW_SUGGESTION_LIMIT` | 6 | Content items shown in the preview panel right column |
| `RICH_PREVIEW_THRESHOLD` | 2 | Min characters before preview panel appears |
| `DEBOUNCE_MS` | 180 | Typing debounce in milliseconds |

---

## Full Results Page

The results page (`#results-page`) is shown when the user presses Enter from the preview state or clicks "See all results". It runs `searchAll(query)` which returns results grouped by type: `product`, `family`, `article`, `faq`, `doc`.

### Layout

| Content type | Layout |
|---|---|
| Products | 6-column grid (`res-grid`) on desktop, 2-column on mobile |
| Product families | 2-column list grid (`res-list-grid`) |
| Content (articles + FAQs) | 2-column list grid |
| User documentation | 2-column list grid |

### Filter tabs

A row of filter chips above the results lets users narrow by content type. The active filter is stored in a single string:

```js
let activeFilter = 'all'; // 'all' | 'product' | 'family' | 'article' | 'doc'
```

Clicking a tab calls `toggleFilter(filterId)`:
- If `filterId === 'all'` → reset to `'all'`
- If `filterId === activeFilter` → deselect, reset to `'all'`
- Otherwise → set `activeFilter = filterId`

Only one filter can be active at a time (single-select). `activeFilter` resets to `'all'` whenever a new search is opened or the results-page search input is used.

**"All" view:** each content type gets a section with up to 6 preview items and a "See all →" link that activates the corresponding filter.

**Filtered view:** the selected type's full result list is shown, with no preview limit.

### `renderResultsPage(query)`

Rebuilds both the tab strip and the results content on every filter change. Tabs are rendered with `onclick="toggleFilter('...')"` — the query is read from the input at call time rather than passed through the onclick attribute, avoiding HTML attribute quoting issues.

---

## Extending the Data

To add a new product family, add an entry to `FAMILIES`:

```js
{ id: 'new-family-id', brandGroup: 'master', name: 'New Family', weldType: 'MIG/MAG',
  url: '/en/family/new-family-id', desc: 'Short description.' }
```

To add products under it, add entries to `PRODUCTS` with the matching `familyId`:

```js
{ id: 'new-product-id', familyId: 'new-family-id', name: 'New Product 300',
  sku: 'NP300GM', category: 'Power source', desc: 'Short spec summary.' }
```

To add an image, fetch the Content Hub numeric ID via Algolia (see `damImageUrl` above) and add it to `FAMILY_IMAGES` or `PRODUCT_IMAGES`:

```js
FAMILY_IMAGES['new-family-id'] = _ch(1234567);
```
