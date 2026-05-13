# AEM Core Components — Complete Guide
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What Are AEM Core Components?

**AEM Core Components** (also called WCM Core Components) are Adobe's library of standardized, production-ready, open-source AEM components. They cover the most common content needs — images, text, navigation, forms, search — built with best practices baked in.

- **GitHub:** https://github.com/adobe/aem-core-wcm-components
- **Current Version:** v2.x (AEM 6.5) / v3.x (AEM Cloud)
- **License:** Apache 2.0

### Why Use Core Components Over Custom Components?

| Aspect | Custom Component | Core Component |
|--------|-----------------|----------------|
| **Development time** | Days/weeks | Minutes (configure only) |
| **Accessibility** | Often missed | WCAG 2.1 AA compliant |
| **Style System** | Must implement yourself | Built-in |
| **Responsive images** | Manual implementation | Built-in (`srcset`, `sizes`) |
| **Internationalization** | Manual | Built-in |
| **Maintained by** | Your team | Adobe + Community |
| **AEM updates** | Manual updates | Auto-updated with AEM |

### The Right Approach for Custom Work

```
Need a component? Ask these questions in order:

1. Does a Core Component cover this use case?
   YES → Configure it + extend if needed (DON'T rebuild from scratch)
   NO  ↓

2. Is it a variation of a Core Component?
   YES → Extend using delegation pattern
   NO  ↓

3. Build a fully custom component
```

---

## 📦 Core Component Catalog

| Component | Resource Type | Use Case |
|-----------|--------------|---------|
| **Image** | `core/wcm/components/image/v3/image` | Responsive images, DAM assets |
| **Title** | `core/wcm/components/title/v3/title` | Heading with configurable level |
| **Text** | `core/wcm/components/text/v2/text` | Rich text content |
| **Button** | `core/wcm/components/button/v2/button` | CTA button with link |
| **Teaser** | `core/wcm/components/teaser/v2/teaser` | Card with image + title + description + CTA |
| **Navigation** | `core/wcm/components/navigation/v2/navigation` | Site navigation from page tree |
| **Breadcrumb** | `core/wcm/components/breadcrumb/v3/breadcrumb` | Hierarchical breadcrumb |
| **Search** | `core/wcm/components/search/v2/search` | Site search with autocomplete |
| **List** | `core/wcm/components/list/v4/list` | Dynamic page/asset lists |
| **Carousel** | `core/wcm/components/carousel/v1/carousel` | Image/content carousel |
| **Accordion** | `core/wcm/components/accordion/v1/accordion` | Expandable sections |
| **Tabs** | `core/wcm/components/tabs/v1/tabs` | Tab-based content |
| **Container** | `core/wcm/components/container/v1/container` | Layout container |
| **Experience Fragment** | `core/wcm/components/experiencefragment/v2/experiencefragment` | XF embedding |
| **Content Fragment** | `core/wcm/components/contentfragment/v1/contentfragment` | CF rendering |
| **Form components** | `core/wcm/components/form/*/v2/*` | Form fields, submit, hidden |
| **PDF Viewer** | `core/wcm/components/pdfviewer/v1/pdfviewer` | Inline PDF display |

---

## 🏗️ How Core Components Are Structured

Core Components follow a strict structure that you must understand to extend them:

```
/apps/core/wcm/components/teaser/v2/teaser/
  ├── .content.xml                       ← Component definition
  │     sling:resourceSuperType = "core/wcm/components/teaser/v1/teaser"
  ├── teaser.html                        ← HTL template
  ├── _cq_dialog/
  │     └── .content.xml                ← Author dialog
  ├── _cq_design_dialog/
  │     └── .content.xml                ← Design dialog (legacy)
  ├── _cq_editConfig/
  │     └── .content.xml                ← Edit bar config
  └── clientlibs/
        └── editor/                     ← Editor-specific JS/CSS
```

---

## 🔑 The Delegation Pattern — Core Concept

The **delegation pattern** is how you extend a Core Component without copying its code. Your custom model delegates behavior to the Core Component's model using `@Self @Via(type = ResourceSuperType.class)`.

### Why NOT Copy Core Component Code

```
❌ WRONG: Copy core component HTL/Java and modify
Problem: When Adobe releases a Core Component update (security fix, feature),
         you don't get it. Your copy is now diverged and unmaintained.

✅ RIGHT: Extend via delegation pattern
Benefit: When Adobe updates the core component, your extension
         automatically inherits the improvements.
```

### Complete Delegation Example — Extending Teaser

**Step 1: Create your component with `sling:resourceSuperType`**

```xml
<!-- /apps/mysite/components/content/mysite-teaser/.content.xml -->
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
          xmlns:cq="http://www.day.com/jcr/cq/1.0"
          xmlns:jcr="http://www.jcp.org/jcr/1.0"

          jcr:primaryType="cq:Component"
          jcr:title="My Site Teaser"
          jcr:description="Extended Teaser with custom fields"
          componentGroup="My Site - Content"

          sling:resourceSuperType="core/wcm/components/teaser/v2/teaser"/>
          <!--                     ↑ This is the KEY — inherit from Core Component -->
```

**Step 2: Extend the Dialog (add custom fields)**

```xml
<!-- /apps/mysite/components/content/mysite-teaser/_cq_dialog/.content.xml -->
<!-- Extend (not replace) the core teaser dialog -->
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
          xmlns:jcr="http://www.jcp.org/jcr/1.0"
          xmlns:nt="http://www.jcp.org/jcr/nt/1.0"

          jcr:primaryType="nt:unstructured"
          jcr:title="My Site Teaser"
          sling:resourceType="cq/gui/components/authoring/dialog">
  <content jcr:primaryType="nt:unstructured"
           sling:resourceType="granite/ui/components/coral/foundation/container">
    <items jcr:primaryType="nt:unstructured">
      <tabs jcr:primaryType="nt:unstructured"
            sling:resourceType="granite/ui/components/coral/foundation/tabs">
        <items jcr:primaryType="nt:unstructured">

          <!-- Inherit ALL core teaser tabs using sling:resourceSuperType -->
          <text sling:resourceType="granite/ui/components/coral/foundation/include"
                path="/apps/core/wcm/components/teaser/v2/teaser/_cq_dialog/content/items/tabs/items/text"/>

          <!-- Add a custom "Branding" tab -->
          <branding jcr:primaryType="nt:unstructured"
                    jcr:title="Branding"
                    sling:resourceType="granite/ui/components/coral/foundation/container">
            <items jcr:primaryType="nt:unstructured">
              <variant jcr:primaryType="nt:unstructured"
                       sling:resourceType="granite/ui/components/coral/foundation/form/select"
                       fieldLabel="Card Variant"
                       name="./variant">
                <items jcr:primaryType="nt:unstructured">
                  <default jcr:primaryType="nt:unstructured" text="Default" value="default"/>
                  <featured jcr:primaryType="nt:unstructured" text="Featured" value="featured"/>
                  <dark jcr:primaryType="nt:unstructured" text="Dark" value="dark"/>
                </items>
              </variant>
              <showBadge jcr:primaryType="nt:unstructured"
                         sling:resourceType="granite/ui/components/coral/foundation/form/checkbox"
                         fieldDescription="Show 'New' badge on this teaser"
                         name="./showBadge"
                         text="Show New Badge"
                         uncheckedValue="false"/>
            </items>
          </branding>

        </items>
      </tabs>
    </items>
  </content>
</jcr:root>
```

**Step 3: Write the Delegation-Based Sling Model**

```java
package com.mysite.core.models;

import com.adobe.cq.wcm.core.components.models.Teaser;
import com.day.cq.wcm.api.Page;
import org.apache.sling.api.SlingHttpServletRequest;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.models.annotations.*;
import org.apache.sling.models.annotations.injectorspecific.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.annotation.PostConstruct;
import java.util.List;

/**
 * My Site Teaser — Extends Core Component Teaser via Delegation.
 *
 * Key pattern: implements Teaser (core interface) + delegates to core model
 * via @Self @Via(type = ResourceSuperType.class)
 */
@Model(
    adaptables   = SlingHttpServletRequest.class,
    adapters     = {MySiteTeaserModel.class, Teaser.class},
    resourceType = "mysite/components/content/mysite-teaser",
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class MySiteTeaserModel implements Teaser {

    private static final Logger LOG = LoggerFactory.getLogger(MySiteTeaserModel.class);

    // ── Delegate to Core Component's model ──────────────────────────────────
    // ResourceSuperType.class tells Sling to follow sling:resourceSuperType
    // and adapt to that component's model. Gives us ALL core teaser behavior.
    @Self
    @Via(type = ResourceSuperType.class)
    private Teaser coreTeaserModel;

    // ── Custom fields (from our extended dialog) ──────────────────────────
    @ValueMapValue
    private String variant;

    @ValueMapValue
    private boolean showBadge;

    // ── Standard Sling injections ─────────────────────────────────────────
    @ScriptVariable
    private Page currentPage;

    @PostConstruct
    protected void init() {
        if (coreTeaserModel == null) {
            LOG.warn("Core Teaser model could not be resolved for resource: {}",
                "check sling:resourceSuperType is set correctly");
        }

        // Apply defaults for custom fields
        if (variant == null) {
            variant = "default";
        }
    }

    // ── DELEGATED METHODS — Pass through to core model ────────────────────
    // All these methods are required by the Teaser interface.
    // We delegate them to coreTeaserModel so we don't lose core functionality.

    @Override
    public String getTitle() {
        return coreTeaserModel != null ? coreTeaserModel.getTitle() : null;
    }

    @Override
    public String getDescription() {
        return coreTeaserModel != null ? coreTeaserModel.getDescription() : null;
    }

    @Override
    public String getLinkURL() {
        return coreTeaserModel != null ? coreTeaserModel.getLinkURL() : null;
    }

    @Override
    public String getTitleType() {
        return coreTeaserModel != null ? coreTeaserModel.getTitleType() : "h3";
    }

    @Override
    public boolean isActionsEnabled() {
        return coreTeaserModel != null && coreTeaserModel.isActionsEnabled();
    }

    @Override
    public List<ListItem> getActions() {
        return coreTeaserModel != null ? coreTeaserModel.getActions() : null;
    }

    @Override
    public com.adobe.cq.wcm.core.components.models.Image getImage() {
        return coreTeaserModel != null ? coreTeaserModel.getImage() : null;
    }

    @Override
    public boolean isImageLinkHidden() {
        return coreTeaserModel != null && coreTeaserModel.isImageLinkHidden();
    }

    @Override
    public boolean isTitleLinkHidden() {
        return coreTeaserModel != null && coreTeaserModel.isTitleLinkHidden();
    }

    @Override
    public boolean isDescriptionHidden() {
        return coreTeaserModel != null && coreTeaserModel.isDescriptionHidden();
    }

    @Override
    public String getExportedType() {
        return "mysite/components/content/mysite-teaser";
    }

    // ── CUSTOM METHODS — New functionality ────────────────────────────────

    public String getVariant() {
        return variant;
    }

    public boolean isShowBadge() {
        return showBadge;
    }

    public boolean isConfigured() {
        return coreTeaserModel != null
            && (getTitle() != null || getImage() != null);
    }
}
```

**Step 4: Write the HTL Template**

```html
<!-- /apps/mysite/components/content/mysite-teaser/mysite-teaser.html -->
<sly data-sly-use.model="com.mysite.core.models.MySiteTeaserModel"/>

<!-- Render using core component's HTL (includes all core rendering logic) -->
<!-- We only override specific parts -->
<div class="mysite-teaser mysite-teaser--${model.variant @ context='attribute'}"
     data-sly-test="${model.configured}">

    <!-- Badge (custom feature) -->
    <span class="mysite-teaser__badge"
          data-sly-test="${model.showBadge}">
        New
    </span>

    <!-- Delegate image rendering to core component -->
    <sly data-sly-resource="${'image' @ resourceType='core/wcm/components/image/v3/image'}"/>

    <!-- Custom title rendering -->
    <div class="mysite-teaser__content">
        <div class="mysite-teaser__title"
             data-sly-element="${model.titleType}"
             data-sly-test="${model.title && !model.titleLinkHidden}">
            <a href="${model.linkURL @ context='uri'}">${model.title}</a>
        </div>

        <p class="mysite-teaser__description"
           data-sly-test="${model.description && !model.descriptionHidden}">
            ${model.description}
        </p>

        <!-- CTA Actions -->
        <div class="mysite-teaser__actions"
             data-sly-test="${model.actionsEnabled && model.actions}">
            <a class="btn btn--primary"
               data-sly-list.action="${model.actions}"
               href="${action.url @ context='uri'}">
                ${action.title}
            </a>
        </div>
    </div>
</div>

<!-- Author placeholder -->
<div class="mysite-teaser mysite-teaser--empty"
     data-sly-test="${!model.configured}">
    Configure this Teaser component.
</div>
```

---

## 🔄 Overlay Pattern (Use Sparingly)

**Overlay** copies a Core Component file to `/apps` at the same path structure, overriding it system-wide. Use only when you need to change behavior for ALL instances globally.

```
Core Component location:
/libs/core/wcm/components/image/v3/image/image.html

Overlay location (copy here to override globally):
/apps/core/wcm/components/image/v3/image/image.html
```

```
Overlay vs Extension:
Overlay  → Modifies ALL instances of the component everywhere
Extension → Modifies only YOUR custom component (using sling:resourceSuperType)

⚠️ Prefer extension over overlay — overlays are hard to maintain!
```

---

## 🖼️ Core Image Component — Working with Responsive Images

The Core Image component handles complex responsive image scenarios:

```java
import com.adobe.cq.wcm.core.components.models.Image;

@Model(adaptables = SlingHttpServletRequest.class, ...)
public class MyImageModel implements Image {

    @Self
    @Via(type = ResourceSuperType.class)
    private Image coreImageModel;

    // Core Image provides:
    // getSrc()          → main image URL
    // getSrcUriTemplate() → URL template for srcset
    // getWidths()       → responsive widths for srcset
    // getAlt()          → alt text (from DAM metadata or dialog)
    // getTitle()        → image caption
    // getFileReference() → DAM path
    // isLazyEnabled()   → whether lazy loading is on
    // getImageLink()    → link wrapping the image

    @Override
    public String getSrc() {
        return coreImageModel != null ? coreImageModel.getSrc() : null;
    }
    // ... delegate all other methods
}
```

```html
<!-- Core Image renders a full responsive picture element: -->
<picture>
    <source srcset="/content/dam/image.coreimg.80.320.jpeg 320w,
                    /content/dam/image.coreimg.80.480.jpeg 480w,
                    /content/dam/image.coreimg.82.768.jpeg 768w"
            type="image/jpeg"/>
    <img src="/content/dam/image.coreimg.82.1280.jpeg"
         alt="Product image"
         loading="lazy"
         width="1280"
         height="720"/>
</picture>
```

---

## 📋 Core Component Interfaces — Key Java APIs

```java
// Key interfaces in com.adobe.cq.wcm.core.components.models:
com.adobe.cq.wcm.core.components.models.Image
com.adobe.cq.wcm.core.components.models.Teaser
com.adobe.cq.wcm.core.components.models.Title
com.adobe.cq.wcm.core.components.models.Text
com.adobe.cq.wcm.core.components.models.Navigation
com.adobe.cq.wcm.core.components.models.NavigationItem
com.adobe.cq.wcm.core.components.models.List
com.adobe.cq.wcm.core.components.models.ListItem
com.adobe.cq.wcm.core.components.models.Search
com.adobe.cq.wcm.core.components.models.Button
com.adobe.cq.wcm.core.components.models.Breadcrumb
com.adobe.cq.wcm.core.components.models.ComponentExporter

// Maven dependency:
<dependency>
    <groupId>com.adobe.cq</groupId>
    <artifactId>core.wcm.components.core</artifactId>
    <version>2.x.x</version>
    <scope>provided</scope>
</dependency>
```

---

## ☁️ AEM 6.5 vs Cloud — Core Component Differences

| Aspect | AEM 6.5 | AEM Cloud |
|--------|---------|-----------|
| **Core Components version** | v2.x | v3.x (latest) |
| **Included by default** | Need to install package | Pre-installed |
| **Image Servlet** | `AdaptiveImageServlet` | `AssetDelivery` API |
| **Dynamic Media** | Optional add-on | Integrated |
| **SPA support** | v2 SPA page component | v3 + Remote SPA |
| **Upgrade frequency** | You manage versions | Continuous updates |

---

## ❓ Interview Questions & Detailed Answers

**Q1. What are AEM Core Components and why should you use them?**

> **Answer:** Core Components are Adobe's open-source library of production-ready, accessible, Style System-enabled, responsive AEM components. They should be the first choice for common content needs (text, image, navigation, etc.) because they save development time, are WCAG 2.1 AA accessible, support the Style System out of the box, and are maintained by Adobe with continuous improvements and security fixes. Custom components should be built only for business-specific functionality not covered by Core Components.

**Q2. What is `sling:resourceSuperType` and how does it enable the delegation pattern?**

> **Answer:** `sling:resourceSuperType` is a property on a component node that establishes inheritance — "my component is a type of X component." Sling uses this for script resolution: if your component doesn't have `mycomponent.html`, Sling looks in the `sling:resourceSuperType` component. The delegation pattern uses this + `@Self @Via(type = ResourceSuperType.class)` in a Sling Model to get an instance of the parent component's model. This means your model gets ALL the core component's logic for free, and you only add or override what you need.

**Q3. What is the difference between overlay and extension for Core Components?**

> **Answer:** **Extension** (via `sling:resourceSuperType`): Creates a NEW component that inherits from Core. Only pages using YOUR component are affected. Preferred approach — targeted, safe, maintainable. **Overlay** (copy file to `/apps/core/...`): Replaces Core Component behavior GLOBALLY — ALL instances of that core component everywhere are affected, including ones not using your custom resourceType. Use overlay only for system-wide changes and be very careful — it blocks future Core Component updates.

**Q4. What does `@Via(type = ResourceSuperType.class)` do?**

> **Answer:** It tells Sling's model injection to follow the `sling:resourceSuperType` chain to find a model to inject. Without it, `@Self` would inject the current model itself. With `@Via(type = ResourceSuperType.class)`, Sling traverses up the resourceSuperType hierarchy until it finds a Sling Model that adapts to the current resource — effectively getting the Core Component's model implementation. This is how the delegation pattern works without manually instantiating the parent model.

**Q5. If a new version of a Core Component (v3) is released, how do you upgrade your extension?**

> **Answer:** Update `sling:resourceSuperType` in your component's `.content.xml` from `core/wcm/components/teaser/v2/teaser` to `core/wcm/components/teaser/v3/teaser`. Update your Java model to implement the new version's interface (`com.adobe.cq.wcm.core.components.models.Teaser`). Add any new interface methods that v3 introduces to your delegation model. Since you're delegating most logic, the upgrade is minimal — you only need to handle new methods added in v3.

---

## ✅ Best Practices

1. **Always check Core Components first** before building custom — 90% of needs are covered
2. **Use delegation, never copy** — copying means missing future fixes
3. **Keep `sling:resourceSuperType`** pointing to a specific version (`/v2/`) — prevents unexpected breaking changes
4. **Implement the full interface** — all interface methods must be implemented when delegating
5. **Don't override what you don't need** — if Core's `getTitle()` is fine, don't override it
6. **Add custom fields in separate dialog tabs** — don't disrupt the core dialog structure
7. **Use the Style System** for visual variants — don't create separate components for colors/styles

---

## 🛠️ Hands-on Practice

1. Create a `mysite-image` component extending `core/wcm/components/image/v3/image`
2. Add a custom dialog field "Caption Position" (above/below image)
3. Write a delegation model that adds `getCaptionPosition()` while delegating all Image interface methods to core
4. Write HTL that renders the image with the caption in the chosen position
5. Verify overlay vs extension: temporarily overlay the core image's HTL file — observe the system-wide effect vs your component-scoped extension
