# Sling URL Decomposition — How AEM Resolves URLs
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 Why URL Decomposition Is Critical

Every single request to AEM goes through Sling's URL resolution process. Understanding this process is fundamental to:
- Debugging why a component isn't rendering
- Writing correct servlet registrations
- Understanding selector-based logic
- Building correct URLs in HTL and Java
- Configuring Dispatcher filter rules correctly

---

## 📐 AEM URL Anatomy — Complete Breakdown

```
https://www.mysite.com/content/mysite/en/products/laptop.variant.compare.html/2024/models?color=black&sort=price#specs

Schema    Host             Path                              Selectors   Ext   Suffix        Query String              Fragment
─────── ─────────────── ──────────────────────────────────  ──────────  ────  ────────────  ────────────────────────  ───────
https:// www.mysite.com  /content/mysite/en/products/laptop .variant   .html /2024/models  ?color=black&sort=price   #specs
                                                             .compare
```

### Components Explained

| Part | Value | What It Does |
|------|-------|-------------|
| **Scheme** | `https://` | Protocol |
| **Host** | `www.mysite.com` | Server hostname |
| **Path** | `/content/mysite/en/products/laptop` | JCR node path |
| **Selectors** | `.variant.compare` | Dot-separated modifiers before extension |
| **Extension** | `.html` | Determines content type / servlet |
| **Suffix** | `/2024/models` | Additional path data after extension |
| **Query String** | `?color=black&sort=price` | Key-value parameters |
| **Fragment** | `#specs` | Client-side anchor (AEM never sees this) |

> **AEM only sees:** path + selectors + extension + suffix + query string
> (Fragment is handled entirely by the browser)

---

## 🔍 Sling's URL Resolution Algorithm — Step by Step

When a request arrives, Sling does this:

```
Request: GET /content/mysite/en/products/laptop.variant.html

STEP 1: PATH RESOLUTION
═══════════════════════
Sling tries to find a JCR node at:
  /content/mysite/en/products/laptop    → EXISTS? → yes → this is the RESOURCE

STEP 2: SERVLET/SCRIPT RESOLUTION
══════════════════════════════════
With resource resolved (/content/mysite/en/products/laptop):
  a. Get sling:resourceType of resource → "mysite/components/content/product"
  b. Look for a servlet or script registered for this resourceType + selector + extension:

  Search order (most specific first):
  1. /apps/mysite/components/content/product/variant.html     (resourceType + "variant" selector + "html")
  2. /apps/mysite/components/content/product/variant.GET.html (resourceType + "variant" + method + "html")
  3. /apps/mysite/components/content/product/html.html        (resourceType + "html" extension only)
  4. /apps/mysite/components/content/product/GET.html         (resourceType + method)
  5. /apps/mysite/components/content/product.html             (resourceType default)
  6. Walk sling:resourceSuperType chain...
  7. /libs/sling/servlet/default/... (absolute fallback)

STEP 3: SCRIPT EXECUTION
═════════════════════════
Execute the matched script (or servlet) with:
  - request.getResource() → /content/mysite/en/products/laptop
  - request.getSelectors() → ["variant"]
  - request.getExtension() → "html"
  - request.getSuffix()    → null
```

---

## 🧭 URL Resolution Deep Dive — With Fallbacks

```
Request: GET /content/mysite/en/products/laptop.model.json

SLING SEARCHES FOR (in this exact order):
1. Registered Servlet:   resourceType="mysite/components/product" selector="model" extension="json"
2. Script file:          /apps/mysite/components/product/model.json.html
3. Servlet:              resourceType="mysite/components/product" extension="json"
4. Script file:          /apps/mysite/components/product/json.html
5. Servlet:              resourceType="mysite/components/product" (any extension)
6. Walk sling:resourceSuperType → repeat steps for parent type
7. Default servlet:      GET.json, default.json, etc.
```

---

## 🎯 Selectors in Practice

Selectors are the most powerful part of AEM's URL structure — they let one resource URL serve multiple formats.

```
/content/mysite/en/products/laptop.html       → Full page rendering
/content/mysite/en/products/laptop.mobile.html → Mobile-specific rendering
/content/mysite/en/products/laptop.model.json  → Sling Model Exporter (JSON API)
/content/mysite/en/products/laptop.feed.xml    → RSS/XML feed
/content/mysite/en/products/laptop.thumbnail.png → Thumbnail image
/content/mysite/en/products/laptop.infinity.json → Full JCR subtree (DEFAULT — BLOCK AT DISPATCHER!)

Multiple selectors:
/content/mysite/en/products/laptop.tidy.5.json    → Formatted JSON, 5 levels deep
/content/mysite/en/products/laptop.variant.html   → Product variant view
```

### Accessing Selectors in Java

```java
// In a Sling Model or Servlet
@SlingObject
private SlingHttpServletRequest request;

// Get all selectors as String
String selectorString = request.getRequestPathInfo().getSelectorString();
// "variant.compare" → the raw selector string

// Get selectors as array
String[] selectors = request.getRequestPathInfo().getSelectors();
// ["variant", "compare"] → individual selectors

// Check if a specific selector is present
boolean hasVariant = Arrays.asList(selectors).contains("variant");

// Get extension
String extension = request.getRequestPathInfo().getExtension();
// "html"

// Get suffix
String suffix = request.getRequestPathInfo().getSuffix();
// "/2024/models" or null if no suffix

// Get full resource path
String resourcePath = request.getResourcePath();
// "/content/mysite/en/products/laptop"
```

### Using Selectors in HTL (Conditional Rendering)

```html
<!-- Render differently based on selector -->
<sly data-sly-use.req="com.mysite.core.models.RequestModel"/>

<!-- Full page -->
<div class="product-full" data-sly-test="${req.selector == 'full'}">
    <sly data-sly-include="/apps/mysite/components/product/full.html"/>
</div>

<!-- Mobile card -->
<div class="product-card" data-sly-test="${req.selector == 'mobile'}">
    <sly data-sly-include="/apps/mysite/components/product/mobile.html"/>
</div>

<!-- Default (no selector) -->
<div class="product-default" data-sly-test="${!req.hasSelector}">
    <sly data-sly-include="/apps/mysite/components/product/default.html"/>
</div>
```

---

## 🗺️ ResourceResolver URL Mapping

AEM can **map** JCR paths to shorter, cleaner URLs. This is distinct from Apache RewriteRules.

```
/etc/map/http/www.mysite.com/     ← ResourceResolver mapping config
  jcr:primaryType = sling:Mapping
  sling:match = "www.mysite.com/"
  sling:internalRedirect = "/content/mysite/en/"
```

```java
// Using ResourceResolver.map() to get the clean URL
ResourceResolver resolver = ...;

String path = "/content/mysite/en/home";

// map() applies the /etc/map rules to get a clean URL
String cleanUrl = resolver.map(path);
// Result: "/home" or "https://www.mysite.com/home"

// In Sling Model — ALWAYS use map() for links
String ctaLink = resolver.map("/content/mysite/en/products") + ".html";
// Not: "/content/mysite/en/products.html"
// But: "/en/products.html" (after mapping)
```

---

## 🔗 Suffix — Advanced Usage

The suffix is the path segment after the extension. It's used to pass structured path data.

```
/content/mysite/en/search.results.html/electronics/laptops
                                       ↑─────────────────
                                       Suffix: /electronics/laptops

/content/mysite/en/compare.html/laptop-pro/laptop-lite
                                 ↑─────────────────────
                                 Suffix: /laptop-pro/laptop-lite (two product paths to compare)
```

```java
// Read suffix in a servlet
String suffix = request.getRequestPathInfo().getSuffix();
// "/electronics/laptops"

// Parse suffix parts
if (suffix != null) {
    String[] parts = suffix.substring(1).split("/");  // Remove leading /
    String category = parts[0];    // "electronics"
    String subCategory = parts.length > 1 ? parts[1] : null;  // "laptops"
}

// Build a URL with suffix in Java
String url = resolver.map("/content/mysite/en/search") + ".results.html"
             + "/electronics/laptops";
```

---

## 🌐 Extension-Based Content Negotiation

```
.html   → Renders via HTL/JSP → text/html
.json   → Renders via Sling GET Servlet → application/json
.xml    → Renders via Sling GET Servlet → application/xml
.pdf    → DAM asset direct download
.infinity.json → Full JCR subtree as JSON (⚠️ BLOCK AT DISPATCHER!)
.tidy.json     → Pretty-printed JSON
.model.json    → Sling Model Exporter JSON
.social.json   → Social media data
```

### The `.json` Security Problem

```
GET /content/mysite/en/home.infinity.json
→ Dumps the ENTIRE JCR subtree under /content/mysite/en/home as JSON
→ Includes ALL author names, metadata, internal paths, workflow data

MUST BE BLOCKED AT DISPATCHER:
/0100 { /type "deny" /url "*.json" /selectors "infinity,tidy,children,angular" }
/0101 { /type "deny" /url "*.json" /selectors "[0-9]*" }
/0102 { /type "deny" /extension "json" /path "/bin/*" }
```

---

## 🧩 Script Resolution — Complete HTL Lookup Order

```
Component: sling:resourceType = "mysite/components/content/hero"
Request: GET /content/page/jcr:content/hero.mobile.html

Sling looks for scripts in this ORDER:
1. /apps/mysite/components/content/hero/mobile.html          (selector script)
2. /apps/mysite/components/content/hero/mobile.GET.html      (selector + method)
3. /apps/mysite/components/content/hero/GET.html             (method only)
4. /apps/mysite/components/content/hero/hero.html            (component name = default)
5. /apps/mysite/components/content/hero.html                 (fallback)

If none found, follow sling:resourceSuperType:
6-10. Repeat steps 1-5 for each ancestor in sling:resourceSuperType chain
11. /libs/sling/servlet/default/...                          (absolute fallback)
```

---

## 🔄 URL Mapping in AEM 6.5 vs Cloud

| Feature | AEM 6.5 | AEM Cloud |
|---------|---------|-----------|
| `/etc/map` | Supported | Supported |
| Vanity URLs | Page Properties → Vanity URL | Same |
| URL shortening | Apache mod_rewrite or `/etc/map` | Dispatcher rewrite + CDN rules |
| Externalizer | `DayLinkExternalizer` OSGi config | Externalizer + environment vars |

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is a Sling selector and how does AEM use it for script resolution?**

> **Answer:** A selector is a dot-separated segment in a URL between the resource path and the extension (e.g., `.mobile.` in `/products/laptop.mobile.html`). Sling uses it to find a more specific script or servlet. For a request with selector "mobile" to a component with resourceType "mysite/components/product", Sling looks for `/apps/mysite/components/product/mobile.html` FIRST, before falling back to the default `product.html`. This allows one component to serve multiple renditions (web, mobile, JSON, PDF) from the same JCR node.

**Q2. What is the difference between a selector and a suffix?**

> **Answer:** Selectors come BEFORE the extension: `/path.selector.html` — they influence script/servlet resolution and can be checked via `request.getSelectors()`. Suffixes come AFTER the extension: `/path.html/suffix/data` — they're used to pass path-structured data (like a second resource path for comparison pages, or category hierarchy for search). Suffixes don't influence script resolution; they're read by the servlet/script as additional data.

**Q3. What is the Sling script resolution order for a request with a selector?**

> **Answer:** Given `mysite/components/hero` and selector "mobile", Sling tries in order: `mobile.html`, `mobile.GET.html`, `GET.html`, `hero.html`, `hero.html` (component name fallback), then walks up `sling:resourceSuperType` repeating the same search, and finally falls back to Sling defaults. The FIRST matching script wins.

**Q4. Why is `.infinity.json` dangerous and how do you block it?**

> **Answer:** `.infinity.json` is a built-in Sling GET servlet selector that dumps the ENTIRE JCR subtree as JSON — including internal properties, author names, metadata, template paths, workflow history. This is a major data exposure risk. Block it at Dispatcher with: `/type "deny" /url "*.json" /selectors "infinity,tidy,children,angular"`. Also block numeric selectors used for deep JSON traversal.

**Q5. What does `ResourceResolver.map()` do and why should you always use it for links?**

> **Answer:** `ResourceResolver.map()` applies the `/etc/map` URL mapping rules to convert a JCR path to its public-facing URL. For example, `/content/mysite/en/home` might map to `/en/home` or even `https://www.mysite.com/home` depending on the mapping config. Always use `resolver.map(path)` before appending `.html` for internal links — hardcoding the JCR path means your links break when URL mapping changes, or show `/content/...` paths to users.

---

## ✅ Best Practices

1. **Always use `resolver.map()`** for internal links — never hardcode `/content/` paths in URLs
2. **Block `.infinity.json` and similar** at Dispatcher — data security
3. **Use selectors for variant renditions** — not separate components or query params
4. **Keep suffixes structured** — use clear `/category/subcategory` conventions
5. **Test servlet resolution with curl** — add `--verbose` to see which servlet handles the request
6. **Use specific resource-type servlet registration** — it uses Sling's resolution, not arbitrary paths

---

## 🛠️ Hands-on Practice

1. Create a component with two script files: `default.html` and `print.html` — access both via selectors
2. Write a servlet registered on a selector (e.g., `.data.json`) — return component data as JSON
3. Read and parse a suffix in a servlet — e.g., `/search.results.html/electronics/laptops`
4. Test `.infinity.json` on your local AEM instance — then add a Dispatcher filter to block it
5. Set up `/etc/map` to map `/content/mysite/en` → `/en` — test that `resolver.map()` returns clean URLs
