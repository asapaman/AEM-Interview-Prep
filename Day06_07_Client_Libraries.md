# Day 6 & 7 — Client Libraries (Clientlibs)
**Difficulty:** Medium | **4 YOE Focus**

---

## 📖 Topic Explanation

Client Libraries (Clientlibs) are AEM's mechanism for managing and serving CSS and JavaScript files. They handle:
- Concatenation of multiple files into one request
- Minification (in production)
- Dependency management between libraries
- Category-based inclusion in pages

### Clientlib JCR Structure
```
/apps/mysite/clientlibs/
  └── site/
        ├── .content.xml         ← cq:ClientLibraryFolder definition
        ├── css/
        │     ├── css.txt        ← List of CSS files (order matters!)
        │     ├── main.css
        │     └── variables.css
        └── js/
              ├── js.txt         ← List of JS files (order matters!)
              ├── app.js
              └── utils.js
```

### .content.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:cq="http://www.day.com/jcr/cq/1.0"
          xmlns:jcr="http://www.jcp.org/jcr/1.0"
          jcr:primaryType="cq:ClientLibraryFolder"
          categories="[mysite.site]"
          dependencies="[jquery, granite.utils]"
          embed="[mysite.fonts]"
          allowProxy="{Boolean}true"/>
```

### css.txt / js.txt
```
#base=css
variables.css
main.css
components/button.css
```
> The `#base=css` line sets the base folder. Files are concatenated in listed order.

---

## 🔑 Key Properties

| Property | Description |
|----------|------------|
| `categories` | Name(s) used to include this library in pages. Can be an array. |
| `dependencies` | Other categories that must be loaded BEFORE this library (loaded separately) |
| `embed` | Other categories whose files are MERGED INTO this library's output |
| `allowProxy` | Required to serve via `/etc.clientlibs/` (Dispatcher-friendly path) |

---

## 🔄 Dependencies vs Embed

### Dependencies
```xml
dependencies="[jquery, mysite.base]"
```
- The dependent libraries are loaded as **separate requests** but guaranteed to be loaded **before** this library
- Deduplication: if multiple clientlibs depend on `jquery`, it's only loaded once

### Embed
```xml
embed="[mysite.fonts, mysite.icons]"
```
- The embedded libraries' CSS/JS is **merged INTO** this library's output
- Results in **fewer HTTP requests** (all in one file)
- Cannot be loaded separately once embedded

---

## 📦 Including Clientlibs in HTL

### Method 1: HTL data-sly-call (Recommended)
```html
<!-- In page component HTL -->
<sly data-sly-use.clientlib="/libs/granite/sightly/templates/clientlib.html">
  <sly data-sly-call="${clientlib.css @ categories='mysite.site'}"/>
</sly>
<!-- ... page content ... -->
<sly data-sly-use.clientlib="/libs/granite/sightly/templates/clientlib.html">
  <sly data-sly-call="${clientlib.js @ categories='mysite.site'}"/>
</sly>
```

### Method 2: Page Policy (via Core Components Page)
Configure clientlib category in **Page Policy** → Core Components page automatically loads it.

### Method 3: ui:includeClientLib (JSP - Legacy)
```jsp
<ui:includeClientLib categories="mysite.site" css="true" js="false"/>
```

---

## 🔒 allowProxy & Dispatcher

Without `allowProxy`:
- Clientlibs served from `/apps/mysite/clientlibs/...`
- Dispatcher BLOCKS `/apps/` paths (security)

With `allowProxy=true`:
- Clientlibs served from `/etc.clientlibs/mysite/clientlibs/...`
- Dispatcher ALLOWS `/etc.clientlibs/` → safe to cache

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between `embed` and `dependencies` in clientlibs?**
> - `dependencies`: Libraries loaded as separate files, guaranteed before this one. Library remains independent.
> - `embed`: Library's content is **physically merged** into this library's CSS/JS output. Reduces HTTP requests but the embedded library can no longer be loaded independently.

**Q2. Why do we need `allowProxy=true`?**
> Dispatcher blocks all requests to `/apps/` for security. With `allowProxy=true`, AEM creates a proxy path at `/etc.clientlibs/` which Dispatcher allows. This is mandatory for any clientlib that needs to be served to end users.

**Q3. How do you debug a clientlib that isn't loading?**
> 1. Check AEM Felix Console → **LibraryManager** servlet: `/libs/granite/ui/content/dumplibs.html`
> 2. Verify the category name matches what's in the page's include call
> 3. Check `css.txt`/`js.txt` for typos in file names
> 4. Check browser Network tab — is the clientlib URL 404ing?
> 5. Check Dispatcher cache — may need to flush

**Q4. What is `#base=css` in css.txt?**
> `#base=` sets the relative root directory for file paths listed below it. `#base=css` means all subsequent file paths are relative to the `css/` subfolder of the clientlib folder.

**Q5. How do you include different clientlibs for Author vs Publish?**
> Use WCM Mode check in HTL:
```html
<sly data-sly-use.wcmmode="com.day.cq.wcm.api.WCMMode">
  <sly data-sly-test="${!wcmmode.disabled}">
    <!-- Author-only clientlib (edit, preview modes) -->
    <sly data-sly-call="${clientlib.js @ categories='mysite.author'}"/>
  </sly>
</sly>
```

**Q6. What happens if you have circular dependencies in clientlibs?**
> AEM will throw a `CyclicDependencyException` and the clientlib won't render. Break the cycle by extracting common code into a shared library.

---

## ✅ Best Practices

- Always set `allowProxy=true` — required for Dispatcher-served environments
- Use `embed` to combine small, always-co-loaded libraries (reduce requests)
- Use `dependencies` for large, shared libraries like jQuery (allow reuse/deduplication)
- Keep component-specific styles in **component clientlibs**, site-wide styles in **site clientlib**
- Load CSS in `<head>`, JS at end of `<body>` (use `async` or `defer` for JS)
- Never put business logic in clientlibs — clientlibs are for presentation only
- Use minification in prod: Configure HtmlLibraryManager OSGi config with `minify=true`

---

## 🛠️ Hands-on Task

1. Create a **site** clientlib (`mysite.site`) with CSS and JS
2. Create a **component** clientlib (`mysite.components.hero`) embedded into site clientlib
3. Include them in a page component via HTL
4. Verify they serve via `/etc.clientlibs/` path
