# Day 2 — Component Development
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What is an AEM Component?

An **AEM Component** is the fundamental building block of a web page in AEM. Think of it like a **LEGO piece** — authors drag and drop components onto a page to build content.

Every component is:
1. A **JCR node** in `/apps/` that defines the component
2. An **HTL script** (or multiple scripts) that renders the HTML
3. An optional **dialog** for authors to configure it
4. An optional **Sling Model** (Java class) containing business logic

### Real-world Examples of Components
- **Hero Banner** — Full-width image with title and CTA button
- **Card Grid** — 3-column grid of cards with image, title, description
- **Navigation** — Site navigation with dropdowns
- **Form** — Contact/lead generation form
- **Accordion** — Expandable FAQ sections
- **Rich Text Editor** — WYSIWYG text editing

---

## 📁 Component Folder Structure

```
/apps/mysite/components/
  ├── content/                          ← Author-facing content components
  │     ├── hero/
  │     │     ├── .content.xml          ← Component definition (REQUIRED)
  │     │     ├── hero.html             ← Main HTL rendering script
  │     │     ├── _cq_dialog/           ← Author dialog (Touch UI)
  │     │     │     └── .content.xml
  │     │     ├── _cq_editConfig/       ← Edit bar behavior in Author
  │     │     │     └── .content.xml
  │     │     ├── _cq_htmlTag/          ← Wrapper element config
  │     │     └── clientlibs/           ← Component-specific CSS/JS
  │     │           ├── .content.xml
  │     │           ├── css/
  │     │           │     ├── css.txt
  │     │           │     └── hero.css
  │     │           └── js/
  │     │                 ├── js.txt
  │     │                 └── hero.js
  │     └── text/
  │           ├── .content.xml
  │           └── text.html
  ├── structure/                        ← Page structure components (locked in templates)
  │     ├── header/
  │     └── footer/
  └── page/                             ← Page rendering components
        └── mysite-page/
              ├── .content.xml
              └── mysite-page.html
```

---

## 📄 Component Definition (.content.xml)

This is the **most important file** in a component. It tells AEM:
- What type of node this is (`cq:Component`)
- What name to show in the Component Browser
- Which group it belongs to
- Whether it inherits from another component

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root
    xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    xmlns:cq="http://www.day.com/jcr/cq/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"

    jcr:primaryType="cq:Component"
    jcr:title="Hero Banner"
    jcr:description="Full-width hero banner with title, subtitle, and CTA"

    sling:resourceSuperType="core/wcm/components/image/v3/image"

    componentGroup="My Site - Content"/>
```

### Key Properties Explained

| Property | Type | Description |
|----------|------|------------|
| `jcr:primaryType` | String | Must be `cq:Component` |
| `jcr:title` | String | Display name shown in Component Browser |
| `jcr:description` | String | Tooltip/description for authors |
| `sling:resourceSuperType` | String | Parent component to inherit from |
| `componentGroup` | String | Group in Component Browser. Use `.hidden` to hide from authors. |

### Component Groups
```
componentGroup=".hidden"          → Hidden (page components, structure)
componentGroup="My Site - Content" → Visible in "My Site - Content" group
componentGroup="General"          → General group
```

---

## 🔑 sling:resourceType — The Magic Property

`sling:resourceType` is the property that **links content to a component**.

When an author drops a component on a page, AEM creates a JCR node under the page's `jcr:content` with:
```xml
<hero jcr:primaryType="nt:unstructured"
      sling:resourceType="mysite/components/content/hero"
      jcr:title="Welcome to AEM!"
      subtitle="Learn AEM development"/>
```

When rendering, Sling sees `sling:resourceType="mysite/components/content/hero"` and looks for:
```
/apps/mysite/components/content/hero/hero.html  ← Rendering script
```

**It's the bridge between content (JCR) and presentation (HTL).**

---

## 🧬 sling:resourceSuperType — Inheritance

Like `extends` in Java, `sling:resourceSuperType` makes your component **inherit** scripts from a parent component.

```
Your Component                Parent Component
/apps/mysite/components/      core/wcm/components/
  content/custom-text/          text/v2/text/
    .content.xml                  .content.xml
    custom-text.html              text.html
                                  edit.html
                                  help.html
```

If your component only has `custom-text.html`, for all other scripts (edit.html, help.html), AEM walks up the `sling:resourceSuperType` chain to find them in the Core Component.

### Why Use Core Components as Super Type?
Adobe's **Core Components** (open-source, `github.com/adobe/aem-core-wcm-components`) are production-ready, accessible, and best-practice AEM components. Rather than building from scratch:
1. Set `sling:resourceSuperType` to point to a Core Component
2. Only override the scripts you need to change
3. Inherit all the good stuff (accessible markup, responsive images, etc.)

---

## 📝 Author Dialog (cq:dialog)

The dialog is what authors see when they click the **wrench/configure icon** on a component in the Page Editor.

### Basic Dialog Structure
```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root
    xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    xmlns:cq="http://www.day.com/jcr/cq/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    xmlns:nt="http://www.jcp.org/jcr/nt/1.0"

    jcr:primaryType="nt:unstructured"
    jcr:title="Hero Banner"
    sling:resourceType="cq/gui/components/authoring/dialog">

  <content
      jcr:primaryType="nt:unstructured"
      sling:resourceType="granite/ui/components/coral/foundation/tabs">

    <items jcr:primaryType="nt:unstructured">

      <!-- TAB 1: Content -->
      <content
          jcr:primaryType="nt:unstructured"
          jcr:title="Content"
          sling:resourceType="granite/ui/components/coral/foundation/container">

        <items jcr:primaryType="nt:unstructured">

          <!-- Title field -->
          <title
              jcr:primaryType="nt:unstructured"
              sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
              fieldLabel="Title"
              fieldDescription="Main heading of the hero banner"
              name="./title"
              required="{Boolean}true"/>

          <!-- Subtitle field -->
          <subtitle
              jcr:primaryType="nt:unstructured"
              sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
              fieldLabel="Subtitle"
              name="./subtitle"/>

          <!-- Rich text editor -->
          <description
              jcr:primaryType="nt:unstructured"
              sling:resourceType="cq/gui/components/authoring/dialog/richtext"
              fieldLabel="Description"
              name="./description"
              useFixedInlineToolbar="{Boolean}true"/>

          <!-- Image path picker -->
          <backgroundImage
              jcr:primaryType="nt:unstructured"
              sling:resourceType="granite/ui/components/coral/foundation/form/pathfield"
              fieldLabel="Background Image"
              name="./backgroundImage"
              rootPath="/content/dam/mysite"/>

          <!-- Checkbox -->
          <openInNewTab
              jcr:primaryType="nt:unstructured"
              sling:resourceType="granite/ui/components/coral/foundation/form/checkbox"
              text="Open link in new tab"
              name="./openInNewTab"
              value="{Boolean}true"/>

          <!-- Select/Dropdown -->
          <theme
              jcr:primaryType="nt:unstructured"
              sling:resourceType="granite/ui/components/coral/foundation/form/select"
              fieldLabel="Theme"
              name="./theme">
            <items jcr:primaryType="nt:unstructured">
              <light
                  jcr:primaryType="nt:unstructured"
                  text="Light"
                  value="light"
                  selected="{Boolean}true"/>
              <dark
                  jcr:primaryType="nt:unstructured"
                  text="Dark"
                  value="dark"/>
            </items>
          </theme>

        </items>
      </content>

      <!-- TAB 2: Advanced -->
      <advanced
          jcr:primaryType="nt:unstructured"
          jcr:title="Advanced"
          sling:resourceType="granite/ui/components/coral/foundation/container">
        <items jcr:primaryType="nt:unstructured">
          <id
              jcr:primaryType="nt:unstructured"
              sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
              fieldLabel="HTML ID"
              name="./id"/>
        </items>
      </advanced>

    </items>
  </content>
</jcr:root>
```

### Common Dialog Field Types

| Granite UI Resource Type | Field Type |
|--------------------------|-----------|
| `foundation/form/textfield` | Single-line text input |
| `foundation/form/textarea` | Multi-line text area |
| `foundation/form/pathfield` | Path picker (browsable JCR tree) |
| `foundation/form/checkbox` | Boolean checkbox |
| `foundation/form/select` | Dropdown/select |
| `foundation/form/numberfield` | Number input |
| `foundation/form/multifield` | Repeatable group of fields |
| `foundation/form/colorfield` | Color picker |
| `foundation/form/datepicker` | Date/time picker |
| `cq/gui/components/authoring/dialog/richtext` | Rich text editor |
| `foundation/form/radiogroup` | Radio button group |
| `foundation/form/hidden` | Hidden field |
| `foundation/form/tagsautocomplete` | Tag picker |

### 🔑 The `name` Property — Where Data Gets Saved
The `name` property on each field determines where the value is saved in JCR.

- `name="./title"` → Saved as property `title` on the **current resource node**
- `name="./jcr:title"` → Saved as `jcr:title` (a standard JCR property)
- `name="items/./label"` → Saved in a child node called `items`

**Always use `./` prefix** to store relative to the current component node.

---

## 🖥️ HTL Rendering Script (hero.html)

```html
<!-- /apps/mysite/components/content/hero/hero.html -->

<!--
  data-sly-use: Instantiates the Sling Model.
  'model' is the variable name we use to access its methods.
-->
<sly data-sly-use.model="com.mysite.core.models.HeroModel"/>

<!--
  The component renders ONLY if the model is not empty.
  This prevents empty boxes in author mode when no content is added.
-->
<div class="hero hero--${model.theme @ context='attribute'}"
     data-sly-test="${model.hasContent}"
     id="${model.id @ context='attribute'}">

    <!-- Background image -->
    <div class="hero__background"
         style="background-image: url('${model.backgroundImageUrl @ context='uri'}')">
    </div>

    <!-- Content overlay -->
    <div class="hero__content">

        <!-- Title: renders only if not empty -->
        <h1 class="hero__title"
            data-sly-test="${model.title}">
            ${model.title}
        </h1>

        <!-- Subtitle -->
        <p class="hero__subtitle"
           data-sly-test="${model.subtitle}">
            ${model.subtitle}
        </p>

        <!-- Rich text description: use context='html' for HTML content -->
        <div class="hero__description"
             data-sly-test="${model.description}">
            ${model.description @ context='html'}
        </div>

        <!-- CTA Button: only show if link and label exist -->
        <div class="hero__cta"
             data-sly-test="${model.ctaLink && model.ctaLabel}">
            <a href="${model.ctaLink @ context='uri'}"
               class="btn btn--primary"
               target="${model.openInNewTab ? '_blank' : '_self' @ context='attribute'}"
               rel="${model.openInNewTab ? 'noopener noreferrer' : '' @ context='attribute'}">
                ${model.ctaLabel}
            </a>
        </div>

    </div>
</div>

<!-- Author placeholder: shows a hint when no content is configured -->
<div class="cmp-hero--empty"
     data-sly-test="${!model.hasContent}">
    <p>Please configure the Hero Banner component.</p>
</div>
```

---

## ☕ Sling Model (HeroModel.java)

```java
package com.mysite.core.models;

import com.day.cq.wcm.api.Page;
import org.apache.commons.lang3.StringUtils;
import org.apache.sling.api.SlingHttpServletRequest;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.api.resource.ResourceResolver;
import org.apache.sling.models.annotations.DefaultInjectionStrategy;
import org.apache.sling.models.annotations.Model;
import org.apache.sling.models.annotations.injectorspecific.*;

import javax.annotation.PostConstruct;

/**
 * Sling Model for the Hero Banner component.
 *
 * adaptables = SlingHttpServletRequest.class → Can adapt from request
 * defaultInjectionStrategy = OPTIONAL → Failed injections = null (won't throw exception)
 */
@Model(
    adaptables = SlingHttpServletRequest.class,
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class HeroModel {

    // ── Injected from component's JCR properties (author-configured values) ──

    @ValueMapValue
    private String title;

    @ValueMapValue
    private String subtitle;

    @ValueMapValue
    private String description;

    @ValueMapValue
    private String backgroundImage;   // Path like /content/dam/mysite/hero.jpg

    @ValueMapValue
    private String ctaLabel;

    @ValueMapValue
    private String ctaLink;

    @ValueMapValue
    private Boolean openInNewTab;

    @ValueMapValue(name = "theme")
    private String theme;

    @ValueMapValue
    private String id;

    // ── Sling-provided objects ──

    @SlingObject
    private ResourceResolver resourceResolver;

    @ScriptVariable
    private Page currentPage;

    // ── Computed fields (set in @PostConstruct) ──

    private String backgroundImageUrl;
    private boolean hasContent;

    /**
     * @PostConstruct runs AFTER all injections are complete.
     * Use for:
     *   - Null checks and default values
     *   - Computed/derived properties
     *   - API calls or JCR operations
     */
    @PostConstruct
    protected void init() {
        // Resolve background image path to a rendition URL
        if (StringUtils.isNotBlank(backgroundImage)) {
            // Map internal JCR path to URL (applies URL mappings)
            backgroundImageUrl = resourceResolver.map(backgroundImage);
        }

        // Determine if the component has enough content to render
        hasContent = StringUtils.isNotBlank(title) || StringUtils.isNotBlank(backgroundImage);

        // Apply defaults
        if (StringUtils.isBlank(theme)) {
            theme = "light";
        }

        // Resolve internal CTA link path
        if (StringUtils.isNotBlank(ctaLink) && ctaLink.startsWith("/content/")) {
            ctaLink = resourceResolver.map(ctaLink) + ".html";
        }
    }

    // ── Public getters (called by HTL) ──

    public String getTitle() { return title; }
    public String getSubtitle() { return subtitle; }
    public String getDescription() { return description; }
    public String getBackgroundImageUrl() { return backgroundImageUrl; }
    public String getCtaLabel() { return ctaLabel; }
    public String getCtaLink() { return ctaLink; }
    public boolean isOpenInNewTab() { return Boolean.TRUE.equals(openInNewTab); }
    public String getTheme() { return theme; }
    public String getId() { return id; }
    public boolean isHasContent() { return hasContent; }
}
```

### 🔑 How HTL Calls Getters
When HTL writes `${model.title}`, it calls `model.getTitle()`.
When HTL writes `${model.hasContent}`, it calls `model.isHasContent()` (for booleans).
When HTL writes `${model.openInNewTab}`, it calls `model.isOpenInNewTab()`.

---

## 🔍 cq:dialog vs cq:design_dialog

| | `cq:dialog` | `cq:design_dialog` |
|--|------------|-------------------|
| **Purpose** | Per-instance configuration | Site-wide design settings |
| **Who edits** | Content authors | Template authors / Developers |
| **Scope** | Only this component instance | All instances sharing the same design |
| **Modern AEM** | Used always | Replaced by Content Policies |
| **Example** | Author enters a specific title | Developer restricts available heading levels |

> ✅ In modern AEM (6.5+ Editable Templates), `cq:design_dialog` is largely **replaced by Content Policies**. You will rarely need to create `cq:design_dialog` in new projects.

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is an AEM component and how does it work?**

> **Answer:** An AEM component is a reusable UI element that combines structure (JCR node definition), behavior (Sling Model/Java logic), and presentation (HTL template). When an author drops a component on a page, AEM creates a JCR content node storing the author's input. When a page is requested, Sling reads the `sling:resourceType` of this node, finds the corresponding HTL script in `/apps/`, instantiates the associated Sling Model to process the data, and renders HTML.

**Q2. What is sling:resourceType and why is it the most important property?**

> **Answer:** `sling:resourceType` is the JCR property that maps content to its rendering component. It's the cornerstone of Sling's resource resolution. Every piece of AEM content (page, component, asset) has a `sling:resourceType` that tells Sling which scripts to use for rendering. Without it, Sling wouldn't know how to render the content.

**Q3. What is sling:resourceSuperType and when do you use it?**

> **Answer:** It's component inheritance. When your component has `sling:resourceSuperType="core/wcm/components/text/v2/text"`, AEM looks for rendering scripts in your component first, then walks up to the supertype component for scripts you haven't overridden. Use it to extend Core Components — you only override what you need to change and inherit everything else. This is the recommended pattern for component development.

**Q4. What is the difference between a cq:Page and its jcr:content node?**

> **Answer:** `cq:Page` is the container node for an AEM page (like a folder). The actual page content, properties, and metadata live in its mandatory child node `jcr:content` (type `cq:PageContent`). Properties like `jcr:title`, `sling:resourceType` (which page component to use), `cq:template`, and all component data are stored in `jcr:content` and its descendants. When AEM renders a page, it actually renders the `jcr:content` node.

**Q5. In a dialog field, what does `name="./title"` mean?**

> **Answer:** The `name` property defines where in JCR the field value is saved when an author submits the dialog. `./title` means: save as the `title` property on the current resource node (the `.` refers to current node, `/title` is the property name). Without the `./` prefix, the property name might not save correctly.

**Q6. What is the componentGroup property used for?**

> **Answer:** `componentGroup` determines which group the component appears under in the Component Browser (the sidebar where authors drag components from). For example, `componentGroup="My Site - Content"` groups your component under "My Site - Content". Use `componentGroup=".hidden"` to hide structural components (like page components, header/footer) from the author's drag-and-drop browser.

**Q7. How do you create a component that authors cannot move or delete?**

> **Answer:** In Editable Templates, lock the component in the Structure layer. In the Template Editor, add the component, then click its "lock" icon. This creates the component in the template's structure section, making it appear on all pages using the template, but authors cannot edit, move, or delete it.

---

## ✅ Best Practices

1. **Always extend Core Components** via `sling:resourceSuperType` — don't build from scratch
2. **Keep HTL dumb** — no business logic in HTL. All logic in Sling Models.
3. **Use `DefaultInjectionStrategy.OPTIONAL`** — prevents null injection from crashing the page
4. **Use `componentGroup=".hidden"`** for page/layout components not meant for author drag-drop
5. **Always use `./` prefix** in dialog `name` fields to store relative to current node
6. **Add `jcr:description`** to components — helps authors understand what each component does
7. **Use `data-sly-test`** to conditionally render sections — avoid empty divs on Publish
8. **Use the correct HTL context** for escaping: `context='uri'` for URLs, `context='html'` for rich text

---

## 🛠️ Hands-on Practice

1. **Create a Text component:**
   - Dialog: title (textfield), body text (rich text), link (pathfield), "Open in new tab" (checkbox)
   - Sling Model: reads all values, resolves internal paths
   - HTL: renders title as `<h2>`, body as HTML, link as `<a>` tag

2. **Extend Core Title component:**
   - `sling:resourceSuperType="core/wcm/components/title/v3/title"`
   - Add a `subtitle` field to the dialog
   - Override only the `title.html` script to add your subtitle

3. **Explore CRXDE:** After adding a component to a page, find its content node in CRXDE and verify the dialog values are saved correctly
