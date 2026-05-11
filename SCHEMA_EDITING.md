# Editing structured-data JSON

This repo holds JSON-LD for **askamelia.com**. The files are consumed by the live site (for example, client-side injectors that fetch JSON from GitHub Pages), search engines, and validators—so **shape and URLs are part of the contract**, not just prose.

Editors should treat these files like **data APIs**: valid JSON, stable `@id`s, and URLs that match production routes.

---

## Before you save (every file)

1. **Validate JSON** before committing (IDE, [jsonlint](https://jsonlint.com/), or `python3 -m json.tool yourfile.json`). If the parser errors, nothing downstream can load the file.
2. **Prefer copying an existing sibling entry** over inventing a new shape. Each file has its own pattern; stay consistent within that file.
3. **Use Schema.org vocabulary** (`@type`, `@id`, `url`, etc.). Prefer **`@type`** (with an `@`). Do not use plain `"type"` on JSON-LD nodes—some bundles in this repo still have legacy `"type": "thing"` on the wrapper; new work should use **`"@type": "Thing"`** (or the specific type) for compatibility with JSON-LD processors.
4. **Keep `@id` URLs unique and stable** across the site. Reuse the same `@id` string everywhere that entity is referenced (e.g. `https://www.askamelia.com/#organization`).
5. **Match production URLs** (scheme, host, path). Pathnames are often used for routing and injector matching; `www` vs non-`www` and trailing slashes matter unless the consumer normalizes them.
6. **`isPartOf` / `breadcrumb` / cross-references** should point at the **correct** page or entity for that content—not the homepage—unless you truly mean the whole site.

---

## File overview

| File | Root shape | What it represents |
|------|------------|---------------------|
| `blog_page_schema.json` | **Array** of bundles | Full JSON-LD **per blog article** (large `@graph` per post). |
| `blog_schema.json` | **Array** of nodes | **Blog index / listing** plus related entities (`WebSite`, `CollectionPage`, `ItemList`, `BlogPosting`, …)—flat nodes, not wrapped in per-post `@graph` bundles. |
| `provider_page_v1_schema.json` | **Array** of nodes | **Provider** pages (v1): one top-level object per provider (e.g. `Physician`). |
| `provider_page_v2_schema.json` | **Array** of bundles | **Provider** pages (v2): `@context` + `@graph` per provider; richer graphs. |
| `location_page_v2_schema.json` | **Array** of bundles | **Location** pages: `@context` + `@graph` per location area. |
| `medspa_local_schema.json` | **Array** (currently one outer item) | **Local medspa** treatment URLs; nested `@graph` structure inside the file. |
| `amelia_location_procedure_schemas_array.json` | **Array** of bundles | **Local procedure** landing pages (e.g. `/local/...`): `@context` + `@graph` per page. |

Always open the file you are editing and confirm whether the root is a **single object**, an **array of objects**, or an **array of graph bundles**—do not assume every file matches `blog_page_schema.json`.

---

## Site injector contract (blog posts)

The blog injector loads **`blog_page_schema.json`**, parses JSON, then:

1. Treats the root as an **array of bundles** (or a single bundle wrapped for compatibility—see engineering).
2. For each bundle, reads **`@graph`**.
3. Finds a node with **`"@type": "WebPage"`** and compares that node’s **`url`** pathname to the current page pathname.
4. Injects the **matched bundle** (the whole `{ "@context", "@type", "@graph" }` object) as one `<script type="application/ld+json">`.

**Rules for `blog_page_schema.json`:**

- Add **one new object** to the **root array** per new article.
- Keep the same **bundle shell** as existing posts: `"@context": "https://schema.org"`, `"@type": "Thing"`, `"@graph": [ ... ]` (do not add extra `url` / `@id` on that shell—put URLs on the right nodes inside `@graph`).
- Ensure **exactly one** primary **`WebPage`** in `@graph` whose **`url`** is the canonical blog URL for that article.
- Keep **`@id`** fragments consistent (`#webpage`, `#thing`, `#article`, `#faq`, …) and wire **`isPartOf`** to **this article’s** `#webpage`, not the homepage, where applicable.
- For **FAQPage** / **`mainEntity`**: no duplicate **`name`** with conflicting answers; each **`acceptedAnswer.text`** must answer its **`name`**.

---

## Per-file editing notes

### `blog_page_schema.json`

- **Root:** `[ bundle, bundle, ... ]`.
- **Match key for injectors:** `WebPage.url` inside each bundle’s `@graph`.
- **FAQ / Article:** align `isPartOf` with the post’s `WebPage` `@id`.

### `blog_schema.json`

- **Root:** `[ node, node, ... ]` — each element is a **top-level** Schema.org node (no shared `@graph` wrapper).
- Contains **`CollectionPage`** / **`ItemList`** for the blog index and **`BlogPosting`** (or similar) entries with **`url`** / **`@id`** for individual posts.
- When adding or reordering list items, update **`numberOfItems`** and **`itemListElement`** positions if your pattern uses them.
- Keep **`@id`** references (`#blog`, `#post`, organization) aligned with `blog_page_schema.json` and the live site.

### `provider_page_v1_schema.json`

- **Root:** `[ physician, physician, ... ]` — flat **`Physician`** (or equivalent) documents with **`@context`** on each object.
- Each object should have a stable **`@id`** and **`url`** pointing at `/providers/...`.
- **`performProcedure`**, **`address`**, **`sameAs`**, etc. should stay internally consistent.

### `provider_page_v2_schema.json`

- **Root:** `[ bundle, bundle, ... ]` — each bundle has **`@context`**, **`@graph`**, and may include outer **`@id`** / **`url`** for the provider (follow **existing entries** in this file for consistency).
- Prefer **`"@type": "Thing"`** (capital **T**) on the bundle shell for JSON-LD tooling; avoid lowercase `"thing"` for new entries.
- Inner graph typically includes **`Person`/`Physician`**, **`WebPage`**, **`FAQPage`**, service areas, etc.—**mirror a complete existing provider** when adding a new one.

### `location_page_v2_schema.json`

- **Root:** `[ bundle, bundle, ... ]` — each item is **`@context` + `@graph`** for a location experience (often **`LocalBusiness` / `MedicalBusiness`**, **`WebPage`**, FAQs, organization references).
- **`@id`** patterns usually include **`/locations/{slug}#...`**. Match slugs and URLs to the real location pages.

### `medspa_local_schema.json`

- **Root:** an **array**; the structure uses **nested** `@graph` blocks inside a top-level graph. **Do not flatten** without coordinating with engineering—injectors or parsers may depend on this layout.
- Treatment pages use paths like **`/local-medspa/...`**. **`WebPage.url`** (and related **`MedicalBusiness`** / procedure **`@id`s**) must match the live medspa URLs.
- When adding a new treatment page, **copy the closest existing treatment block** and update all URLs, names, and nested references.

### `amelia_location_procedure_schemas_array.json`

- **Root:** `[ bundle, bundle, ... ]` — many **local procedure** landers (`/local/...`).
- Each bundle uses **`@context`**, **`@graph`**, and typically **`WebPage`** / **`MedicalWebPage`**, **`BreadcrumbList`**, procedure and location references.
- **Legacy wrapper field:** some entries use **`"type": "thing"`** instead of **`"@type": "Thing"`**. New entries should use **`"@type"`**; consider fixing legacy rows when touching them.
- **`breadcrumb`** and **`BreadcrumbList`** item names/URLs must match the real hierarchy; watch for copy-paste errors (wrong city or location slug).

---

## JSON-LD hygiene (all files)

- **Valid JSON only** — no comments, no trailing commas, escape control characters inside strings (unescaped newlines or raw tabs inside `"..."` break `JSON.parse`).
- **One concern per edit when possible** — large unrelated reformats make reviews hard and increase merge conflict risk.
- **Strings in `articleBody` or long descriptions** — preserve escaping for quotes; prefer `\n` for newlines inside strings where the file already uses that pattern.
- After edits, **re-validate** and, for blog pages, **smoke-test** the injector path in the browser console (success log vs “no matching schema”).

---

## Quick checklist (new blog post in `blog_page_schema.json`)

- [ ] New object appended to the **root array**
- [ ] Same bundle keys as siblings: `@context`, `@type`, `@graph` (no extra root `url`)
- [ ] **`WebPage`** in `@graph` with correct canonical **`url`**
- [ ] **`@id`s** unique per post; cross-refs resolve inside the bundle (or to shared org URLs)
- [ ] **`isPartOf`** on Article/FAQPage points at **this post’s** `WebPage`, not `https://www.askamelia.com/#webpage`, unless intentional
- [ ] FAQ questions/answers unique and aligned
- [ ] `python3 -m json.tool blog_page_schema.json` succeeds
- [ ] Deploy / push so GitHub Pages serves the updated file before expecting the site to show new data

---

## Who to ask

If a change might **alter the root JSON type** (object vs array), **rename files**, or **restructure `@graph` nesting**, check with whoever maintains the **fetch + inject** scripts on askamelia.com so the contract stays aligned.
