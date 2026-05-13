# Day 20 & 21 — HTL / Sightly
**Difficulty:** Medium–Hard | **4 YOE Focus**

---

## 📖 Topic Explanation

HTL (HTML Template Language), formerly called Sightly, is AEM's server-side templating language. It's **logic-less HTML-first** — business logic belongs in Sling Models, HTL only handles rendering.

### Key Principles
- HTML-valid syntax: `data-sly-*` attributes
- Context-aware XSS protection by default
- No scriptlets — all logic in Sling Models

---

## 📋 All data-sly-* Statements

### 1. `data-sly-use` — Instantiate a Sling Model
```html
<!-- Adapt from Java class -->
<sly data-sly-use.model="com.mysite.core.models.HeroModel"/>

<!-- Adapt from HTL template file -->
<sly data-sly-use.template="templates/common.html"/>

<!-- Adapt with options passed to model -->
<sly data-sly-use.model="${'com.mysite.models.MyModel' @ itemId=itemId}"/>
```

### 2. `data-sly-test` — Conditional Rendering
```html
<!-- Show element only if condition is true -->
<div data-sly-test="${model.title}">
    <h1>${model.title}</h1>
</div>

<!-- Negate with ! -->
<p data-sly-test="${!model.isEmpty}">Content goes here</p>

<!-- Store result in variable for reuse -->
<div data-sly-test.hasTitle="${model.title}">
    <h1 data-sly-test="${hasTitle}">${model.title}</h1>
    <span data-sly-test="${!hasTitle}">No title</span>
</div>
```

### 3. `data-sly-list` — Iterate Collections
```html
<!-- Basic iteration -->
<ul>
    <li data-sly-list="${model.items}">${item.label}</li>
</ul>

<!-- Named iterator + index -->
<ul>
    <li data-sly-list.navItem="${model.navItems}">
        <span>${navItemList.index}: ${navItem.title}</span>
        <!-- navItemList provides: index, count, first, last, odd, even -->
    </li>
</ul>
```

### 4. `data-sly-repeat` — Like list but on the element itself
```html
<!-- Repeat the element for each item (no wrapper needed) -->
<div class="card" data-sly-repeat="${model.cards}">
    <h3>${item.title}</h3>
    <p>${item.description}</p>
</div>
```

### 5. `data-sly-resource` — Include a Child Resource
```html
<!-- Include a child resource with its own component rendering -->
<div data-sly-resource="${'header' @ resourceType='mysite/components/header'}"/>

<!-- Include existing child node -->
<div data-sly-resource="${'footer'}"/>

<!-- With wrapper element replaced -->
<sly data-sly-resource="${'navigation'}"/>
```

### 6. `data-sly-include` — Include Another HTL Script
```html
<!-- Include another HTL file (shares same context) -->
<sly data-sly-include="footer.html"/>
<sly data-sly-include="/apps/mysite/components/common/tracking.html"/>
```

### 7. `data-sly-call` — Call a Template Block
```html
<!-- Define a template -->
<template data-sly-template.card="${@ title, description, link}">
    <div class="card">
        <h3>${title}</h3>
        <p>${description}</p>
        <a href="${link}">Read More</a>
    </div>
</template>

<!-- Call the template -->
<sly data-sly-list="${model.items}">
    <sly data-sly-call="${card @ title=item.title, description=item.desc, link=item.url}"/>
</sly>
```

### 8. `data-sly-attribute` — Dynamic Attributes
```html
<!-- Single attribute -->
<a data-sly-attribute.href="${model.link}">Click</a>

<!-- Conditional attribute -->
<a data-sly-attribute.target="${model.isExternal ? '_blank' : null}">Link</a>

<!-- Multiple attributes from map -->
<div data-sly-attribute="${model.attributeMap}">...</div>
```

### 9. `data-sly-element` — Dynamic Tag Name
```html
<!-- Change tag name dynamically -->
<h1 data-sly-element="${model.headingLevel}">${model.title}</h1>
<!-- Renders as <h1>, <h2>, <h3> etc. based on model -->
```

### 10. `data-sly-unwrap` — Remove Wrapper Element
```html
<!-- The div itself is removed, only inner content renders -->
<div data-sly-unwrap="${model.noWrapper}">
    <p>Content</p>
</div>
```

---

## 🔒 Context-Aware Escaping

HTL automatically applies XSS escaping based on context:

```html
<!-- Default: HTML entity escaping -->
${model.title}                           → HTML context

<!-- Explicit contexts -->
${model.link @ context='uri'}            → URL encoding
${model.jsVar @ context='scriptString'} → JS string escaping
${model.style @ context='styleToken'}   → CSS escaping
${model.html @ context='html'}          → Raw HTML (unsafe - use carefully)
${model.attr @ context='attribute'}     → HTML attribute escaping
${model.text @ context='text'}          → Text context (same as default)

<!-- Disable escaping (ONLY for trusted, sanitized HTML) -->
${model.richText @ context='unsafe'}
```

---

## 🔗 Include vs Resource vs Call

| Statement | Use Case | Shares context? |
|-----------|---------|----------------|
| `data-sly-include` | Include another HTL script | ✅ Yes |
| `data-sly-resource` | Include a separate JCR resource with its own component | ❌ No |
| `data-sly-call` | Call a template block defined in same/included file | ✅ Yes |

---

## ❓ Interview Questions & Answers

**Q1. How do you call a Sling Model in HTL?**
> `<sly data-sly-use.model="com.mysite.core.models.MyModel"/>`
> Then access: `${model.propertyName}`. HTL adapts the current resource or request to the model based on the model's `adaptables`.

**Q2. What is the difference between `data-sly-include` and `data-sly-resource`?**
> - `data-sly-include`: Includes another HTL script in the SAME context/request. The included script has access to the same variables and resource.
> - `data-sly-resource`: Renders a SEPARATE JCR resource with ITS OWN component rendering. Creates a sub-request. The child resource can be a different component type.

**Q3. How does HTL handle XSS protection?**
> HTL automatically applies context-aware escaping. By default, all expressions `${...}` are HTML-escaped. Use `@ context='uri'` for URLs, `@ context='scriptString'` in JS, etc. This prevents most XSS vulnerabilities without developer effort. `context='unsafe'` disables escaping — use only for sanitized HTML.

**Q4. What is `data-sly-list` iterator variable?**
> When using `data-sly-list.item="${collection}"`, HTL provides a `itemList` object with:
> - `itemList.index`: 0-based index
> - `itemList.count`: 1-based count
> - `itemList.first`: true for first item
> - `itemList.last`: true for last item
> - `itemList.odd` / `itemList.even`: for alternating styles

**Q5. How do you conditionally add an HTML attribute?**
```html
<!-- Set to null to omit the attribute entirely -->
<a data-sly-attribute.target="${model.openInNewTab ? '_blank' : null}"
   href="${model.link @ context='uri'}">
    Click Here
</a>
```

**Q6. What is `<sly>` tag?**
> `<sly>` is HTL's "invisible" element — it's used when you need a data-sly-* statement but don't want any HTML element in the output. The `<sly>` tag and its `data-sly-*` attributes are removed from output; only the inner content (or effect) remains.

**Q7. How do you create reusable HTL snippets?**
> Use `data-sly-template` to define a template block, then `data-sly-call` to invoke it with parameters:
```html
<template data-sly-template.button="${@ label, link, style}">
  <a href="${link @ context='uri'}" class="btn btn-${style @ context='attribute'}">${label}</a>
</template>
<sly data-sly-call="${button @ label='Click Me', link='/page.html', style='primary'}"/>
```

---

## ✅ Best Practices

- Never put business logic in HTL — all logic in Sling Models
- Always use the correct `context` for escaping (especially `context='uri'` for links)
- Use `<sly>` instead of wrapper `<div>` for structural statements
- Define reusable snippets with `data-sly-template` + `data-sly-call`
- Use `data-sly-test` for conditional rendering — never inline ternaries for complex logic
- Avoid `context='unsafe'` — if needed, ensure content is properly sanitized

---

## 🛠️ Hands-on Tasks

**Day 20**: Build an HTL template that renders a product list using `data-sly-list` with first/last styling
**Day 21**: Create a reusable card template using `data-sly-template` + `data-sly-call`, include it from multiple components
