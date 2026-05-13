# Day 5 — Policies & Component Restrictions
**Difficulty:** Hard | **4 YOE Focus**

---

## 📖 Topic Explanation

Content Policies in AEM are configuration objects that define component behavior and restrictions. They are the modern replacement for design dialogs.

### Policy Storage
```
/conf/<site>/settings/wcm/policies/
  └── <site>/components/
        ├── core/wcm/components/title/
        │     └── default/              ← Policy node
        │           ├── jcr:primaryType = "nt:unstructured"
        │           └── allowedHeadings = ["H1","H2","H3"]
        └── core/wcm/components/parsys/
              └── default/
                    └── components = ["mysite/components/text", ...]
```

### Policy Assignment (in Template Structure)
```xml
<root jcr:primaryType="nt:unstructured"
      cq:policy="/conf/mysite/settings/wcm/policies/mysite/components/parsys/default"/>
```

---

## 🔑 Key Concepts

### Page Policy vs Content Policy

| | Page Policy | Content Policy |
|--|------------|----------------|
| **Applied to** | Page (via template) | Specific component instance |
| **Controls** | Client libraries, metadata, CSS classes | Component config, allowed child components |
| **Location** | Template structure → page `jcr:content` | Layout containers |

### How Component Restrictions Work (Layout Container)

1. Open Template Editor → Select Layout Container
2. Click **Policy** icon → Create/select policy
3. Under **Allowed Components**, enable specific component groups or individual components
4. Save → Authors can ONLY use those components in that container

---

## 🔧 Component Restriction via Policy (JCR)

```xml
<!-- Policy for a Layout Container -->
<jcr:root xmlns:jcr="..." xmlns:sling="...">
  <components jcr:primaryType="nt:unstructured">
    <!-- Allow Text component -->
    <text jcr:primaryType="nt:unstructured"
          enabled="{Boolean}true"
          path="mysite/components/content/text"/>
    <!-- Allow Image component -->
    <image jcr:primaryType="nt:unstructured"
           enabled="{Boolean}true"
           path="core/wcm/components/image/v3/image"/>
  </components>
</jcr:root>
```

---

## 🎨 Style System

The Style System (part of policies) allows authors to apply predefined CSS classes to components:

### Policy with Style System
```xml
<styles jcr:primaryType="nt:unstructured">
  <buttonStyles jcr:primaryType="nt:unstructured"
                jcr:title="Button Style"
                id="button-style">
    <items jcr:primaryType="nt:unstructured">
      <primary jcr:primaryType="nt:unstructured"
               jcr:title="Primary"
               value="btn-primary"/>
      <secondary jcr:primaryType="nt:unstructured"
                 jcr:title="Secondary"
                 value="btn-secondary"/>
    </items>
  </buttonStyles>
</styles>
```

In HTL, access the applied style:
```html
<div class="${component.appliedCssClasses}">...</div>
```

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between page policy and content policy?**
> - **Page Policy**: Applied to the page component via template. Controls page-level settings like client library categories to include, CSS classes on `<body>`, and metadata.
> - **Content Policy**: Applied to individual component instances within a template. Controls component behavior (e.g., allowed heading levels in Title, allowed components in Layout Container).

**Q2. How do you restrict which components can be placed in a specific area?**
> Via **Content Policy** on the Layout Container. In the Template Editor, select the Layout Container, open its policy, and under "Allowed Components" enable only the desired components. This is enforced by AEM's authoring UI — the component browser only shows allowed components.

**Q3. How does the Style System work?**
> The Style System allows you to define CSS class groups in a component's policy. Authors can then select from predefined styles in the component toolbar. AEM applies the selected CSS classes to the component's wrapper element. Developers just need to provide the CSS for those classes in clientlibs.

**Q4. What is a Layout Container vs a Parsys?**
> - **Parsys** (Paragraph System): Legacy AEM 6.x component. Static, not policy-driven. Component restrictions via design dialogs only.
> - **Layout Container**: Modern replacement. Policy-driven component restrictions. Supports Responsive Grid layout with breakpoints. Used in all Editable Templates.

**Q5. How do you programmatically read a component's content policy?**
```java
@OSGiService
private PolicyManager policyManager;

// In a Sling Model or Servlet
ContentPolicy policy = policyManager.getPolicy(currentResource);
if (policy != null) {
    ValueMap policyProps = policy.getProperties();
    String[] allowedHeadings = policyProps.get("allowedHeadings", String[].class);
}
```

**Q6. Can two different templates share the same policy?**
> Yes! Policies are stored separately from templates in `/conf/.../policies/`. Multiple templates can reference the same policy, enabling consistent behavior across the site.

---

## ✅ Best Practices

- Create **granular policies** per container type (hero, main content, sidebar)
- Never allow ALL components in a container — restrict to what makes design sense
- Use **Style System** for visual variants instead of creating multiple component types
- Store policies under `/conf/<site>/` not `/etc/` (deprecated in AEM 6.5+)
- Document policy decisions — it's easy to forget WHY certain components are restricted

---

## 🛠️ Hands-on Task

1. Create a policy for a Layout Container that allows only: Text, Image, Button, Video
2. Create a Style System for a Button component with 3 variants: Primary, Secondary, Ghost
3. Apply different policies to Header zone vs Body zone in a template
