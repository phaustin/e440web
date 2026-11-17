# Search: pressing Enter on a result drops `BASE_URL` prefix in static builds (clicking works)

<!--
Issue for jupyter-book/myst-theme.
Reproduced 2026-11-17 in headless Chrome with mystmd 1.13.0 (book-theme fetched at build time).
-->

**Versions:** mystmd 1.13.0 (also seen with jupyter-book 2.1.7 / mystmd 1.12.0),
`book-theme` template, static HTML build (`--html`). Code below is from
myst-theme `main` at 6cce1e2.

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

BASE_URL=/sub myst build --html

# Serve so the site lives under /sub
mkdir -p serve && cp -r _build/html serve/sub
cd serve && python -m http.server 8000
```

1. Open http://localhost:8000/sub/
2. Open search (⌘K / Ctrl+K) and type `Uniqueword`
3. **Click** the result → goes to `http://localhost:8000/sub/target` ✅
4. Search again, select the result with the arrow keys, press **Enter** →
   goes to `http://localhost:8000/target` (404) ❌

Observed result (headless Chrome, mystmd 1.13.0):

| Action      | Result link    | Landed at                             |
|-------------|----------------|---------------------------------------|
| Click       | `/sub/target`  | `http://localhost:8000/sub/target/` ✅ |
| ↓ + Enter   | `/sub/target`  | `http://localhost:8000/target` (404) ❌ |

## Cause

`SearchForm` (`packages/site/src/components/Navigation/Search.tsx`) handles
Enter by passing the raw result URL (e.g. `/target`) to the navigate provider:

```ts
const navigate = useNavigateProvider();
const baseurl = useBaseurl();   // assigned but never used
...
if (event.key === 'Enter') {
  const url = searchResults[selectedIndex]?.url;
  if (url) {
    navigate(url);
    closeSearch?.();
  }
}
```

`Document` in `packages/site/src/pages/Root.tsx` provides the link helpers:

- **Server-rendered site:** `{ Link, NavLink, navigate }`, where `navigate` is
  React Router's `useNavigate()`. It applies the router basename, so Enter works.
- **Static build (`staticBuild`):** only `Link`/`NavLink` with
  `reloadDocument`, and **no `navigate`**. `useNavigateProvider()` then falls
  back to `(to) => { window.location.href = to }`, which ignores the base URL.

Clicking works in both cases because it goes through `Link`, which resolves
against the router basename.

Adding the base URL inside `SearchForm` would be wrong: in a server-rendered
site the router `navigate` already applies the basename, so you'd get
`/sub/sub/target`. The fix belongs where the static-build providers are created.

## Proposed fix

In the static-build branch of `Root.tsx`, provide a `navigate` that does the
full-page load *with* the base URL (using the existing `withBaseurl` helper).
Also remove the unused `baseurl` in `Search.tsx` and complete the
`useCallback` dependency list.

```diff
--- a/packages/site/src/pages/Root.tsx
+++ b/packages/site/src/pages/Root.tsx
@@ -7,6 +7,7 @@ import {
   Theme,
   ThemeProvider,
   useThemeSwitcher,
+  withBaseurl,
 } from '@myst-theme/providers';
 import {
   Links,
@@ -61,6 +62,11 @@ export function Document({
     ? {
         Link: (props: any) => <Link {...{ ...props, reloadDocument: true }} />,
         NavLink: (props: any) => <NavLink {...{ ...props, reloadDocument: true }} />,
+        // Static builds do a full page load, which bypasses the router basename,
+        // so prefix the base URL ourselves (as Link/NavLink do via the router).
+        navigate: (to: string) => {
+          window.location.href = withBaseurl(to, baseurl);
+        },
       }
     : {
         Link: Link as any,
--- a/packages/site/src/components/Navigation/Search.tsx
+++ b/packages/site/src/components/Navigation/Search.tsx
@@ -469,7 +469,6 @@ function SearchForm({
   }, []);
   // Handle item selection
   const navigate = useNavigateProvider();
-  const baseurl = useBaseurl();
 
   // Handle item selection and navigation
   const handleSearchKeyPress = useCallback<KeyboardEventHandler<HTMLInputElement>>(
@@ -505,7 +504,7 @@ function SearchForm({
         }
       }
     },
-    [searchResults, selectedIndex],
+    [searchResults, selectedIndex, navigate, closeSearch],
   ); // Our form doesn't use the submit function
   const onSubmit = useCallback((event: FormEvent<HTMLFormElement>) => {
     event.preventDefault();
```

Suggested changeset (`.changeset/static-search-enter-baseurl.md`):

```md
---
'@myst-theme/site': patch
---

Fix keyboard (Enter) selection of a search result dropping the base URL in static builds.
```

**Verification:** I applied the same change to the built `root-*.js` bundle of
the repro above. In headless Chrome, ↓ + Enter then lands on
`http://localhost:8000/sub/target/`, and clicking is unchanged. I haven't run the
monorepo build or test suite against the patch.
