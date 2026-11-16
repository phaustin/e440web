# Search: pressing Enter on a result drops `BASE_URL` prefix (clicking works)

<!--
Draft issue for jupyter-book/myst-theme (or jupyter-book/jupyter-book).
Written from reading the built theme bundle; run the repro once before filing.
-->

**Versions:** jupyter-book 2.1.7 (mystmd 1.12.0), static HTML build

## Description

When a site is built with a `BASE_URL`, choosing a search result with the
keyboard (arrow keys, then Enter) goes to the result's URL *without* the base
URL. Clicking the same result works.

## Minimal reproduction

```bash
mkdir repro && cd repro
printf 'version: 1\nproject:\n  title: Repro\nsite:\n  template: book-theme\n' > myst.yml
printf '# Home\n\nSee the other page.\n' > index.md
printf '# Target Page\n\nUniqueword here.\n' > target.md

BASE_URL=/sub jupyter-book build --html

# Serve so the site lives under /sub
mkdir -p serve && cp -r _build/html serve/sub
cd serve && python -m http.server 8000
```

1. Open http://localhost:8000/sub/
2. Open search and type `Uniqueword`
3. **Click** the result → goes to `http://localhost:8000/sub/target` ✅
4. Search again, use the arrow keys to select the result, press **Enter** →
   goes to `http://localhost:8000/target` (404) ❌

## Cause (from the built bundle)

In `SearchForm`, the Enter handler passes the raw result URL to the navigate
provider:

```js
const navigate = useNavigateProvider();
useBaseurl();                    // return value is discarded
...
if (e.key === 'Enter') {
  const url = searchResults[selectedIndex]?.url;   // e.g. "/target", no base URL
  url && (navigate(url), closeSearch?.());
}
```

In a static build, the theme provider supplies `Link`/`NavLink` (with
`reloadDocument`) but no `navigate`. So `useNavigateProvider()` falls back to
`url => { window.location.href = url }`, which ignores the router basename.
Clicking works because it goes through `Link`, which does apply the basename.

The bare `useBaseurl();` call suggests the intended code was:

```js
const baseurl = useBaseurl();
...
navigate(withBaseurl(url, baseurl));
```
