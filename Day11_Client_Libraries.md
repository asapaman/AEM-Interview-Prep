# Day 6–7 — Client Libraries (Clientlibs)
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What Are Client Libraries?

**Client Libraries (Clientlibs)** are AEM's system for managing, organizing, and delivering front-end assets (CSS and JavaScript) to browsers. They handle:
- **Bundling:** Combine multiple CSS/JS files into fewer HTTP requests
- **Minification:** Compress code for faster delivery
- **Category-based loading:** Load specific sets of CSS/JS where needed
- **Dependency management:** Load libraries in the correct order
- **Proxy serving:** Serve assets through a CDN-safe `/etc.clientlibs/` path

### Why Not Just Use `<link>` and `<script>` Tags?

In a plain HTML page, you manually manage script loading order, caching, and file paths. In AEM:
- Components are assembled dynamically from many bundles
- Each component might need its own CSS/JS
- The same clientlib might be needed by multiple components (but should only load once)
- Assets need unique URLs for cache busting after updates

AEM Clientlibs solve all these problems.

---

## 📁 Clientlib Structure

```
/apps/mysite/clientlibs/
  ├── clientlib-base/                   ← Global CSS/JS (loaded on all pages)
  │     ├── .content.xml                ← Clientlib definition node
  │     ├── css/
  │     │     ├── css.txt               ← Order and list of CSS files to include
  │     │     ├── base.css
  │     │     ├── typography.css
  │     │     └── variables.css
  │     └── js/
  │           ├── js.txt                ← Order and list of JS files to include
  │           ├── utils.js
  │           └── analytics.js
  │
  ├── clientlib-site/                   ← Site-specific CSS/JS
  │     ├── .content.xml
  │     ├── css/
  │     │     ├── css.txt
  │     │     └── site.css
  │     └── js/
  │           ├── js.txt
  │           └── site.js
  │
  └── clientlib-author/                 ← Author-only CSS (edit mode helpers)
        ├── .content.xml
        └── css/
              ├── css.txt
              └── author.css
```

---

## 📄 Clientlib Node Definition (.content.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root
    xmlns:cq="http://www.day.com/jcr/cq/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"

    jcr:primaryType="cq:ClientLibraryFolder"

    <!-- REQUIRED: The category name — used when including this clientlib -->
    categories="[mysite.base]"

    <!-- IMPORTANT: Allows serving from /etc.clientlibs/ proxy path -->
    allowProxy="{Boolean}true"

    <!-- Load order: lower number loads first (default 0) -->
    longCacheKey="${buildNumber}"

    <!-- If this lib depends on others, list them here -->
    <!-- AEM ensures they load BEFORE this one -->
    dependencies="[core.wcm.components.accordion.v1]"

    <!-- Embed another lib's content directly into this one -->
    <!-- Result: one fewer HTTP request -->
    embed="[mysite.vendor.jquery]"/>
```

---

## 🔑 Key Properties Explained

### `categories`

The **identifier** for your clientlib. Use it when including the lib in your page HTL.

```xml
categories="[mysite.base]"            <!-- Single category -->
categories="[mysite.base,mysite.all]" <!-- Multiple categories — included when EITHER is requested -->
```

**Naming convention:**
```
projectname.scope          → mysite.base
projectname.scope.variant  → mysite.base.print
```

### `allowProxy` — Critical for Security!

```xml
allowProxy="{Boolean}true"
```

Without `allowProxy`, clientlibs are served from their actual path:
```
/apps/mysite/clientlibs/clientlib-base/css/site.css
```

The Dispatcher typically **blocks all `/apps/` paths** for security. With `allowProxy=true`, AEM also serves the clientlib from a proxy path:
```
/etc.clientlibs/mysite/clientlibs/clientlib-base/css/site.css
```

The `/etc.clientlibs/` path is **Dispatcher-friendly** (typically allowed). **Always set `allowProxy=true`** on clientlibs that contain front-end assets.

### `dependencies` vs `embed`

```xml
<!-- DEPENDENCIES: Load this lib BEFORE mine, but keep as SEPARATE files -->
dependencies="[mysite.vendor.react]"
<!-- Result: 2 HTTP requests (react.js + mysite.site.js) -->
<!-- Use when: vendor lib is shared and cached separately -->

<!-- EMBED: Merge the other lib's content INTO this file -->
embed="[mysite.vendor.lodash]"
<!-- Result: 1 HTTP request (lodash + mysite.site.js combined) -->
<!-- Use when: vendor lib is small and only used by this clientlib -->
```

---

## 📋 css.txt and js.txt — File Load Order

These files tell AEM which CSS/JS files to include and in what ORDER.

```
# css.txt for clientlib-base
# Lines starting with # are comments
# Each line is a path relative to the clientlib folder

#base=css
variables.css
typography.css
base.css
grid.css

# Can also reference files in subdirectories
components/hero.css
components/nav.css
components/footer.css
```

```
# js.txt for clientlib-base

#base=js
utils.js
analytics.js
site-init.js
```

**Important:** AEM concatenates these files in the ORDER listed in css.txt/js.txt. Load order matters for CSS specificity and JS initialization.

---

## 📥 Including Clientlibs in HTL (Page Component)

```html
<!-- /apps/mysite/components/page/mysite-page/mysite-page.html -->

<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8"/>
    <meta name="viewport" content="width=device-width, initial-scale=1.0"/>

    <!-- Include CSS from multiple categories -->
    <sly data-sly-use.clientlib="/libs/granite/sightly/templates/clientlib.html"/>

    <!-- Method 1: Include both CSS and JS for a category -->
    <sly data-sly-call="${clientlib.all @ categories='mysite.base'}"/>

    <!-- Method 2: Include ONLY CSS (put in <head>) -->
    <sly data-sly-call="${clientlib.css @ categories='mysite.site'}"/>

    <!-- Method 3: Include multiple categories at once -->
    <sly data-sly-call="${clientlib.css @ categories=['mysite.base','mysite.fonts']}"/>
</head>

<body>
    <!-- Page content -->
    <sly data-sly-include="body.html"/>

    <!-- Method 4: Include ONLY JS at bottom of body (best practice) -->
    <sly data-sly-call="${clientlib.js @ categories='mysite.site'}"/>

    <!-- Include author-only clientlib (edit mode styles) -->
    <sly data-sly-call="${clientlib.all @ categories='mysite.author'}"
         data-sly-test="${wcmmode.edit}"/>
</body>
</html>
```

### Clientlib HTL Template Methods

| Method | Result |
|--------|--------|
| `clientlib.all` | Includes both `<link>` (CSS) and `<script>` (JS) tags |
| `clientlib.css` | Includes only `<link rel="stylesheet">` tags |
| `clientlib.js` | Includes only `<script src="...">` tags |

---

## 🔗 Component-Level Clientlibs

```
/apps/mysite/components/content/hero/
  ├── .content.xml              ← Component definition
  ├── hero.html                 ← HTL template
  └── clientlibs/              ← Component-specific assets
        ├── .content.xml
        │     categories="[mysite.component.hero]"
        │     allowProxy="{Boolean}true"
        ├── css/
        │     ├── css.txt
        │     └── hero.css
        └── js/
              ├── js.txt
              └── hero.js
```

**Strategy 1 (Current Project):** Include component clientlibs individually:
```html
<!-- In the page component, include each component's clientlib -->
<sly data-sly-call="${clientlib.all @ categories='mysite.component.hero'}"/>
```

**Strategy 2 (Recommended at Scale):** Aggregate all component clientlibs into one:
```xml
<!-- clientlib-site .content.xml -->
embed="[mysite.component.hero, mysite.component.nav, mysite.component.card]"
```

**Strategy 3 (Modern):** Use the `cq:htmllibmanager` to auto-collect all categories matching a pattern.

---

## 🏎️ Performance: CSS in `<head>`, JS at Bottom of `<body>`

```html
<!-- ✅ CORRECT PATTERN -->
<html>
<head>
    <!-- CSS in head: render-blocking is acceptable for CSS (needed to paint) -->
    <sly data-sly-call="${clientlib.css @ categories='mysite.all'}"/>
</head>
<body>
    <!-- Content renders without waiting for JS -->
    <main>...</main>

    <!-- JS at end of body: doesn't block page render -->
    <sly data-sly-call="${clientlib.js @ categories='mysite.all'}"/>
</body>
</html>

<!-- ❌ WRONG PATTERN -->
<head>
    <!-- Loading JS in <head> blocks page rendering! -->
    <sly data-sly-call="${clientlib.all @ categories='mysite.all'}"/>
</head>
```

---

## 🌍 Clientlib URL Structure

```
AEM serves clientlibs at two paths:

1. Direct path (NOT Dispatcher-safe — use only internally):
/apps/mysite/clientlibs/clientlib-base.css

2. Proxy path (allowProxy=true — Dispatcher allows this):
/etc.clientlibs/mysite/clientlibs/clientlib-base.css

3. AEM Cloud — Dispatcher always serves via:
/etc.clientlibs/...
```

### Cache-Busting URL

AEM appends a hash/timestamp to clientlib URLs so browsers get fresh files after deployments:
```
/etc.clientlibs/mysite/clientlibs/clientlib-base.css?v=1704067200
                                                          ↑ changes on content update
```

---

## ☁️ AEM 6.5 vs Cloud — Clientlib Differences

| Aspect | AEM 6.5 | AEM Cloud Service |
|--------|---------|-----------------|
| **Minification** | `HtmlLibraryManager` OSGi config | Same, but via Cloud Manager pipeline |
| **Debug mode** | `?debug=layout` URL param | Same |
| **allowProxy** | Required for Dispatcher | Required (stricter enforcement) |
| **Linting** | Optional | CSS/JS linting in Cloud Manager |
| **CSS Preprocessors** | Manual build step | Supported via `aemsync` and build pipeline |

---

## 🐛 Debugging Clientlibs

```
Problem: CSS/JS not loading on page

Step 1: Check clientlib exists
→ CRXDE: /apps/mysite/clientlibs/clientlib-site
→ Verify .content.xml exists with correct jcr:primaryType=cq:ClientLibraryFolder

Step 2: Verify categories match
→ .content.xml categories="[mysite.site]"
→ HTL: categories='mysite.site'
→ Must be EXACTLY the same string

Step 3: Check allowProxy
→ If accessing /etc.clientlibs/... → allowProxy="{Boolean}true" must be set

Step 4: Use AEM debug tool
→ http://localhost:4502/libs/granite/ui/content/dumplibs.html
→ Search for your category → verify files are listed

Step 5: Check for JS errors
→ Browser console → look for 404s on clientlib files
→ Check /var/log/error.log for HtmlLibraryManager errors

Step 6: Force rebuild
→ http://localhost:4502/libs/granite/ui/content/dumplibs.rebuild.html
→ Click "Rebuild" → clears clientlib cache and recompiles
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is `allowProxy=true` and why is it important?**

> **Answer:** `allowProxy=true` on a `cq:ClientLibraryFolder` enables AEM to serve the clientlib's assets from the `/etc.clientlibs/` proxy path in addition to the direct `/apps/` path. The Dispatcher security configuration typically blocks all requests to `/apps/` (since it contains code, not public content). The `/etc.clientlibs/` path is explicitly allowed in the Dispatcher, making it the safe, public URL for front-end assets. Always set `allowProxy=true` on clientlibs that serve CSS/JS to browsers.

**Q2. What is the difference between `embed` and `dependencies` in a clientlib?**

> **Answer:**
> - `dependencies` = "Load these OTHER libs BEFORE me, but keep them as separate HTTP requests." Use when the dependency is large and shared — browsers can cache it separately.
> - `embed` = "Copy the content of these libs INTO my own output." Results in fewer HTTP requests because everything is in one file. Use for small, private utilities.
> Example: jQuery should be a dependency (large, cached by browser across sites). A small custom utility used only by this component should be embedded.

**Q3. What is `css.txt`/`js.txt` and why does order matter?**

> **Answer:** These are text files that list which CSS/JS files to include and in what order. AEM concatenates the listed files in that exact sequence. Order matters because:
> - CSS: Later rules override earlier ones. Put variables and base styles BEFORE component styles.
> - JS: If `site-init.js` depends on `utils.js`, utils.js must be listed first. Otherwise you get "undefined is not a function" errors.

**Q4. How do you include a clientlib only on Author (edit mode)?**

> **Answer:**
```html
<sly data-sly-use.clientlib="/libs/granite/sightly/templates/clientlib.html"/>
<sly data-sly-use.wcmmode="com.day.cq.wcm.api.WCMMode"/>
<!-- Only include author-specific CSS when in Edit or Design mode -->
<sly data-sly-call="${clientlib.css @ categories='mysite.author'}"
     data-sly-test="${wcmmode == 'EDIT' || wcmmode == 'DESIGN'}"/>
```
> Or more simply using the `wcmmode` object available in HTL scripts with `WCMMode`.

**Q5. How do you troubleshoot a clientlib that isn't loading?**

> **Answer:** My checklist:
> 1. Verify the clientlib node exists in CRXDE with `jcr:primaryType=cq:ClientLibraryFolder`
> 2. Check the `categories` value matches exactly what's used in HTL (case-sensitive)
> 3. Verify `allowProxy=true` if expecting `/etc.clientlibs/` path
> 4. Check `css.txt`/`js.txt` — listed files must actually exist in the clientlib folder
> 5. Use `/libs/granite/ui/content/dumplibs.html` to verify category registration
> 6. Rebuild clientlibs at `/libs/granite/ui/content/dumplibs.rebuild.html`
> 7. Check browser console for 404 errors on specific CSS/JS files

**Q6. Should CSS go in `<head>` or `<body>`? What about JS?**

> **Answer:** CSS should be in `<head>` — browsers need CSS to paint the page, so render-blocking here is expected and acceptable (avoids Flash of Unstyled Content). JS should be at the bottom of `<body>` unless it's critical for initial render. Loading JS in `<head>` without `async`/`defer` blocks HTML parsing and delays page render. In HTL: use `clientlib.css` in `<head>` and `clientlib.js` just before `</body>`.

---

## ✅ Best Practices

1. **Always set `allowProxy=true`** — clientlibs without it break on Dispatcher
2. **Put CSS in `<head>`, JS before `</body>`** — improves page render performance
3. **Use categories instead of direct paths** — decoupled, reusable
4. **Separate author clientlibs** (`mysite.author`) — use WCM mode check, don't load on Publish
5. **One base clientlib, component-specific clientlibs** — don't put everything in one file
6. **Explicit file order in css.txt/js.txt** — don't rely on filesystem order
7. **Use `dependencies` for shared vendor libs** (jQuery, React) — lets browser cache independently
8. **Use `embed` for small private utils** — reduces HTTP requests
9. **Rebuild clientlibs** after adding/removing files from css.txt/js.txt during development

---

## 🛠️ Hands-on Practice

**Day 6:**
1. Create a `clientlib-base` with category `mysite.base` and `allowProxy=true`
2. Add `css.txt` listing `variables.css`, `base.css`, `typography.css` in that order
3. Include it in your page component's `<head>` using `data-sly-call`

**Day 7:**
1. Create a component-specific clientlib for a Hero component: `mysite.component.hero`
2. Embed it in `clientlib-site` using the `embed` property
3. Create an author-only clientlib `mysite.author` with edit mode helper CSS
4. Include it only when `wcmmode.edit` is true
5. Test by visiting `/libs/granite/ui/content/dumplibs.html` and searching for your categories
