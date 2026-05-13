# Day 20–21 — HTL / Sightly (Complete Guide)
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What is HTL?

**HTL (HTML Template Language)**, formerly called **Sightly**, is AEM's server-side templating language. It replaced JSP as the recommended way to render AEM components starting with AEM 6.0.

### Why HTL Over JSP?

| Concern | JSP | HTL |
|---------|-----|-----|
| **XSS Security** | Manual escaping required — easy to forget | **Auto-escaping by context** — secure by default |
| **Separation of concerns** | Java scriptlets mixed with HTML | Zero Java in templates — logic in Sling Models |
| **Testability** | Hard to test templates | Sling Models are pure Java — unit-testable |
| **Designer-friendly** | Java knowledge needed | Pure HTML-like syntax |
| **Expression language** | JSP EL | HTL Expression Language (simpler) |

### HTL File Location
```
/apps/mysite/components/content/hero/
  ├── hero.html           ← Main HTL file (named after component folder)
  ├── title.html          ← Can be included or used as script for selector
  └── _cq_dialog/
```

---

## 💡 HTL Syntax Fundamentals

HTL expressions and block statements are HTML-valid attributes — HTL files open correctly in any HTML editor.

```html
<!-- HTL Expression: outputs a value -->
${properties.title}

<!-- HTL Block Statement: a data-sly-* attribute on any HTML element -->
<p data-sly-test="${properties.title}">${properties.title}</p>

<!-- The <sly> tag: a virtual element that renders its content but NOT the tag itself -->
<!-- Use it when you need a block statement without an actual HTML element -->
<sly data-sly-use.model="com.mysite.core.models.HeroModel"/>
```

---

## 📋 All HTL Block Statements

### `data-sly-use` — Instantiate a Sling Model or Use-API class

```html
<!-- Instantiate a Sling Model -->
<sly data-sly-use.model="com.mysite.core.models.HeroModel"/>
<!-- Now use: ${model.title}, ${model.subtitle}, etc. -->

<!-- Instantiate a WCM Use-API (legacy) -->
<sly data-sly-use.page="com.day.cq.wcm.api.Page"/>

<!-- Instantiate another HTL file as a template class -->
<sly data-sly-use.templates="shared/templates.html"/>

<!-- Multiple use declarations -->
<sly data-sly-use.model="com.mysite.core.models.HeroModel"
     data-sly-use.clientlib="/libs/granite/sightly/templates/clientlib.html"/>

<!-- Pass parameters to the model (available via bindings in Use-class) -->
<sly data-sly-use.navModel="${'com.mysite.core.models.NavModel' @ maxDepth=3, rootPath='/content/mysite'}"/>
```

---

### `data-sly-test` — Conditional Rendering

```html
<!-- Renders element ONLY if expression is truthy -->
<h1 data-sly-test="${model.title}">${model.title}</h1>

<!-- Falsy values: false, 0, '', null, undefined, empty collections -->
<!-- Truthy values: non-empty strings, non-zero numbers, non-empty collections -->

<!-- Negation: use ! for NOT -->
<div data-sly-test="${!model.hasContent}" class="placeholder">
    No content configured. Please edit this component.
</div>

<!-- Save result to variable for reuse (avoids calling getter multiple times) -->
<div data-sly-test.hasContent="${model.hasContent}">
    <!-- ${hasContent} is available here as the boolean result -->
    <span>Content available: ${hasContent}</span>
</div>

<!-- Complex condition (prefer to put complex logic in Sling Model!) -->
<div data-sly-test="${model.isAdmin && model.featureEnabled}">
    Admin feature
</div>
```

---

### `data-sly-list` — Iterate Over Collections

```html
<!-- Basic iteration: item is the current element, itemList is the list -->
<ul>
    <li data-sly-list="${model.items}">${item}</li>
</ul>

<!-- Custom variable name (recommended for clarity) -->
<ul>
    <li data-sly-list.product="${model.products}">
        <h3>${product.name}</h3>
        <span>${product.price}</span>
        <a href="${product.link @ context='uri'}">View</a>
    </li>
</ul>

<!-- Loop status variable: itemName + 'List' -->
<ul>
    <li data-sly-list.card="${model.cards}">
        <!-- cardList.count = total, cardList.index = 0-based, cardList.first, cardList.last -->
        <span class="count">${cardList.index + 1} / ${cardList.count}</span>
        <h3 class="${cardList.first ? 'first-card' : ''}">${card.title}</h3>
    </li>
</ul>

<!-- Status variables for a list named "product" → productList -->
<!-- productList.index    = current index (0-based) -->
<!-- productList.count    = total items -->
<!-- productList.first    = true if first item -->
<!-- productList.last     = true if last item -->
<!-- productList.odd      = true on odd indexes (1, 3, 5...) -->
<!-- productList.even     = true on even indexes (0, 2, 4...) -->
<!-- productList.middle   = true if not first and not last -->
```

---

### `data-sly-repeat` — Repeat Element Itself (not wrap in container)

```html
<!-- data-sly-list wraps children inside the element
     data-sly-repeat repeats the element itself -->

<!-- data-sly-list: ONE <ul>, MULTIPLE <li> -->
<ul>
    <li data-sly-list.item="${model.items}">${item}</li>
</ul>
<!-- Result: <ul><li>A</li><li>B</li></ul> -->

<!-- data-sly-repeat: MULTIPLE <tr> elements -->
<tr data-sly-repeat.row="${model.tableRows}">
    <td>${row.col1}</td>
    <td>${row.col2}</td>
</tr>
<!-- Result: <tr><td>...</td></tr><tr><td>...</td></tr> -->
```

---

### `data-sly-include` — Include Another HTL File

```html
<!-- Include another script from the SAME component -->
<sly data-sly-include="head-includes.html"/>

<!-- Include from another path -->
<sly data-sly-include="/apps/mysite/components/shared/breadcrumb.html"/>

<!-- Include with unwrap (strip the <sly> tag) -->
<sly data-sly-include="body.html" data-sly-unwrap/>
```

---

### `data-sly-resource` — Include a JCR Resource

```html
<!-- Include another AEM resource and render it using ITS own component -->
<sly data-sly-resource="/content/mysite/shared/header"/>

<!-- Include with specific resource type override -->
<sly data-sly-resource="${'hero' @ resourceType='mysite/components/content/hero'}"/>

<!-- Include child node of current resource -->
<sly data-sly-resource="heroImage"/>

<!-- Include with append/prepend path -->
<sly data-sly-resource="${@ path='myPath', resourceType='mysite/components/content/text'}"/>
```

---

### `data-sly-template` and `data-sly-call` — Reusable HTL Templates

```html
<!-- DEFINE a template (function definition) -->
<template data-sly-template.card="${@ title, imageUrl, link}">
    <div class="card">
        <img src="${imageUrl @ context='uri'}" alt="${title}"/>
        <h3><a href="${link @ context='uri'}">${title}</a></h3>
    </div>
</template>

<!-- CALL the template with parameters -->
<sly data-sly-call="${card @ title=product.name, imageUrl=product.imageUrl, link=product.link}"/>

<!-- Templates in separate files -->
<!-- shared-templates.html: define templates -->
<!-- hero.html: -->
<sly data-sly-use.templates="shared-templates.html"/>
<sly data-sly-call="${templates.card @ title='Hello', imageUrl='/dam/img.jpg', link='/home.html'}"/>
```

---

### `data-sly-element` — Dynamic HTML Tag Name

```html
<!-- Dynamically set the HTML element type -->
<div data-sly-element="${model.headingLevel}">${model.title}</div>
<!-- If model.headingLevel = "h2" → <h2>...</h2> -->
<!-- If model.headingLevel = "p"  → <p>...</p> -->

<!-- Use case: Title component where policy controls H1-H6 level -->
<sly data-sly-use.model="com.mysite.core.models.TitleModel"/>
<div class="title" data-sly-element="${model.element}">${model.title}</div>
```

---

### `data-sly-attribute` — Dynamic Attributes

```html
<!-- Set a single attribute dynamically -->
<a href="${model.link @ context='uri'}"
   target="${model.openInNewTab ? '_blank' : '_self' @ context='attribute'}">
   Click here
</a>

<!-- Set multiple attributes at once using a map -->
<div data-sly-attribute="${model.attributes}">Content</div>
<!-- model.attributes returns Map<String, String> {"class": "hero--dark", "data-id": "123"} -->

<!-- Remove an attribute by setting its value to null or empty -->
<a href="${model.link}"
   rel="${model.isExternal ? 'noopener noreferrer' : null @ context='attribute'}">
</a>
```

---

### `data-sly-unwrap` — Strip the Wrapper Element

```html
<!-- Renders children but strips the element itself -->
<div data-sly-unwrap>
    <p>This paragraph renders without the div wrapper</p>
</div>
<!-- Output: <p>This paragraph renders without the div wrapper</p> -->

<!-- Common use: conditional wrapper that should be skipped -->
<a href="${model.link}" data-sly-unwrap="${!model.hasLink}">
    <span>${model.label}</span>
</a>
<!-- If hasLink=true: <a href="..."><span>...</span></a> -->
<!-- If hasLink=false: <span>...</span> (no <a> wrapper) -->
```

---

### `data-sly-text` — Set Text Content

```html
<!-- Set text content (escapes HTML by default) -->
<h1 data-sly-text="${model.title}"></h1>
<!-- Equivalent to: <h1>${model.title}</h1> -->

<!-- But data-sly-text takes precedence over child content -->
<h1 data-sly-text="${model.title}">Placeholder text (ignored if model.title is set)</h1>
```

---

## 🔒 HTL Context Escaping — Critical Security Concept

HTL **automatically escapes output** based on where it's used. The escape mode is determined by context.

```html
<!-- CONTEXT AUTO-DETECTION -->
<div>${model.title}</div>
<!-- Context: HTML → auto HTML-escapes: <script> becomes &lt;script&gt; -->

<a href="${model.link}">Link</a>
<!-- Context: URI → auto URI-escapes: spaces become %20 -->

<p class="${model.cssClass}">...</p>
<!-- Context: HTML attribute → auto HTML-attribute-escapes -->
```

### Explicit Context Override

```html
<!-- HTML context (default) — HTML entity encoding -->
<div>${model.text}</div>

<!-- URI context — URL encoding -->
<a href="${model.url @ context='uri'}">Link</a>

<!-- HTML attribute — HTML attribute safe encoding -->
<div class="${model.cssClass @ context='attribute'}">

<!-- RAW HTML — NO ESCAPING (use only for trusted rich text!) -->
<div>${model.richText @ context='html'}</div>
<!-- ⚠️ DANGEROUS: only use for content from AEM's RTE — never for user input -->

<!-- Script context (inside <script> tags) — JS string escaping -->
<script>var title = "${model.title @ context='scriptString'}";</script>

<!-- Style context (inside <style> tags) -->
<style>.${model.className @ context='styleToken'} { color: red; }</style>

<!-- Unsafe (disables ALL escaping — NEVER use) -->
<div>${model.html @ context='unsafe'}</div>
<!-- ❌ XSS RISK — never use this -->
```

### Context Summary Table

| Context | Used For | Example |
|---------|---------|---------|
| `html` (default) | HTML body content | `<p>${text}</p>` |
| `attribute` | HTML attributes | `class="${cls @ context='attribute'}"` |
| `uri` | href, src, action | `href="${link @ context='uri'}"` |
| `scriptString` | JS string values | `var x = "${val @ context='scriptString'}"` |
| `styleToken` | CSS class/property | `.${cls @ context='styleToken'}` |
| `text` | Explicit text node | Same as default |
| `html` (explicit) | Trusted HTML (RTE) | `${richText @ context='html'}` |
| `unsafe` | ❌ Never use | Disables all escaping |

---

## 💻 Real-World HTL Example — Complete Card Component

```html
<!-- /apps/mysite/components/content/card/card.html -->

<sly data-sly-use.model="com.mysite.core.models.CardModel"
     data-sly-use.clientlib="/libs/granite/sightly/templates/clientlib.html"/>

<!-- Load component CSS -->
<sly data-sly-call="${clientlib.css @ categories='mysite.component.card'}"/>

<!-- Only render if model has content -->
<article class="card card--${model.variant @ context='attribute'}"
         data-sly-test="${model.hasContent}"
         id="${model.id @ context='attribute'}">

    <!-- Card image: only show if image path is present -->
    <div class="card__image" data-sly-test="${model.imageUrl}">
        <img src="${model.imageUrl @ context='uri'}"
             alt="${model.imageAlt}"
             loading="lazy"/>
    </div>

    <!-- Card body -->
    <div class="card__body">

        <!-- Tag/category label -->
        <span class="card__tag" data-sly-test="${model.tag}">
            ${model.tag}
        </span>

        <!-- Title (H3 by default, configurable via policy) -->
        <div class="card__title" data-sly-element="${model.headingElement}">
            ${model.title}
        </div>

        <!-- Description: rich text from RTE -->
        <div class="card__description"
             data-sly-test="${model.description}">
            ${model.description @ context='html'}
        </div>

        <!-- CTA Links -->
        <div class="card__actions"
             data-sly-test="${model.primaryCtaLink && model.primaryCtaLabel}">

            <a href="${model.primaryCtaLink @ context='uri'}"
               class="btn btn--primary"
               target="${model.primaryCtaNewTab ? '_blank' : '_self' @ context='attribute'}"
               rel="${model.primaryCtaNewTab ? 'noopener noreferrer' : '' @ context='attribute'}">
                ${model.primaryCtaLabel}
            </a>

            <!-- Secondary CTA: only if both link and label present -->
            <a href="${model.secondaryCtaLink @ context='uri'}"
               class="btn btn--secondary"
               data-sly-test="${model.secondaryCtaLink && model.secondaryCtaLabel}">
                ${model.secondaryCtaLabel}
            </a>
        </div>

    </div>

</article>

<!-- Author placeholder when no content configured -->
<div class="card card--empty"
     data-sly-test="${!model.hasContent}">
    <p>Configure this Card component using the edit dialog.</p>
</div>

<!-- Component JS -->
<sly data-sly-call="${clientlib.js @ categories='mysite.component.card'}"/>
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is HTL and why was it introduced?**

> **Answer:** HTL (HTML Template Language) is AEM's secure server-side templating language that replaced JSP. It was introduced because JSP allowed dangerous practices: Java scriptlets in templates (poor separation of concerns), easy-to-forget manual XSS escaping, and tight coupling between logic and presentation. HTL has no Java in templates (logic belongs in Sling Models), automatic context-aware XSS escaping (secure by default), and an HTML-valid syntax that works in standard HTML editors.

**Q2. What is context-aware escaping in HTL?**

> **Answer:** HTL automatically determines the escaping mode based on where an expression is used in HTML. In the body (`<p>${text}</p>`), it applies HTML entity encoding (`<` → `&lt;`). In attributes (`href="${url}"`), it applies URI encoding. In JavaScript strings (`"${val @ context='scriptString'}"`), it applies JS escaping. This prevents XSS by default — you can't accidentally output malicious scripts. You must explicitly opt into `context='html'` for trusted rich text and `context='unsafe'` (never) to disable escaping.

**Q3. What is `data-sly-use` and when would you use it?**

> **Answer:** `data-sly-use` instantiates a Sling Model, WCM Use-API class, or another HTL file and binds it to a local variable. It's the bridge between the Java logic layer and the template. You use it whenever you need data from a Sling Model: `data-sly-use.model="com.mysite.core.models.HeroModel"` then access `${model.title}`.

**Q4. What is the `<sly>` element and why use it?**

> **Answer:** `<sly>` is an HTL virtual element that renders its contents but produces NO output element itself. Use it when you need a block statement (like `data-sly-use`, `data-sly-test`, `data-sly-include`) but don't want an actual HTML wrapper. For example, `<sly data-sly-use.model="..."/>` instantiates the model without adding any HTML to the page output.

**Q5. What is `data-sly-unwrap` used for?**

> **Answer:** `data-sly-unwrap` strips the element it's placed on, rendering only its children. It's useful for conditional wrapper elements — e.g., wrap content in `<a>` only if there's a link, otherwise just render the content. `<a href="${link}" data-sly-unwrap="${!hasLink}">content</a>` → if `hasLink=false`, renders just `content` without the `<a>` tag.

**Q6. What is the difference between `data-sly-list` and `data-sly-repeat`?**

> **Answer:** `data-sly-list` iterates over a collection and renders the CONTENTS of the element for each item (the element itself renders once as the container). `data-sly-repeat` repeats the ELEMENT ITSELF for each item. For example, in a `<ul>`, use `data-sly-list` on `<li>` (one `<ul>`, many `<li>`). For table rows, use `data-sly-repeat` on `<tr>` to repeat the row element itself.

---

## ✅ Best Practices

1. **Never use `context='unsafe'`** — it disables all XSS protection
2. **Use `context='html'` sparingly** — only for content from AEM's RTE
3. **Always use `context='uri'`** for hrefs and src attributes
4. **Keep HTL dumb** — no complex logic in templates; all business logic in Sling Models
5. **Use `<sly>` for block statements** without HTML output
6. **Use `data-sly-test`** to hide empty components instead of rendering empty wrappers
7. **Name `data-sly-list` variables descriptively** (`data-sly-list.product` not just `data-sly-list`)
8. **Use `data-sly-template` for reusable markup patterns** within a component

---

## 🛠️ Hands-on Practice

**Day 20:**
1. Create an HTL template using all block statements: `use`, `test`, `list`, `include`
2. Build a simple card that shows different content based on `model.variant`
3. Experiment with `data-sly-element` to render H2 or H3 based on a model property

**Day 21:**
1. Implement a `data-sly-template` for a reusable badge component
2. Create a shared HTL file with `data-sly-template` definitions and call it from multiple components
3. Deliberately inject HTML-unsafe content and verify HTL auto-escapes it
4. Test `context='html'` with RTE output — verify it renders correctly
