# Day 5 — Content Policies, Component Restrictions & Style System
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What Are Content Policies?

A **Content Policy** is a reusable set of configuration values for a component that is managed by template authors — not individual page authors — and not hardcoded by developers.

### Real-World Analogy

Imagine a hotel. Each room (component) has certain features:
- **Page author** = Guest: Can turn lights on/off but can't rewire the room
- **Template author** = Hotel manager: Sets which channels are available on the TV (policy)
- **Developer** = Electrician: Actually builds the wiring (code)

Policies = Hotel manager's rules that apply to ALL guests in that room type.

### What Policies Control

| Without Policy | With Policy |
|---------------|------------|
| Authors can pick ANY heading (H1–H6) on a title component | Policy restricts to H2, H3, H4 only |
| Authors can add ANY component | Policy limits to only Hero, Text, Image |
| Authors can enable/disable features freely | Policy pre-configures defaults |
| Image crops are unconstrained | Policy enforces 16:9 ratio |

---

## 📍 Where Policies Are Stored

```
/conf/
  └── mysite/
        └── settings/
              └── wcm/
                    └── policies/
                          └── mysite/
                                ├── components/
                                │     └── content/
                                │           ├── title/
                                │           │     └── policy_abc123/     ← Named policy
                                │           │           ├── defaultType = "h2"
                                │           │           └── allowedTypes = [h2, h3, h4]
                                │           └── image/
                                │                 └── policy_def456/
                                │                       └── allowedRenditionWidths = [480, 768, 1280]
                                └── wcm/
                                      └── foundation/
                                            └── components/
                                                  └── responsivegrid/
                                                        └── policy_grid789/
                                                              └── allowedComponents/
```

---

## 🔗 How Policies Connect to Components

The link between a component instance in a template and its policy is stored in the template's structure:

```
/conf/mysite/settings/wcm/templates/homepage/
  └── policies/
        └── jcr:content/
              └── root/
                    └── hero/
                          ← cq:policy = "mysite/components/content/hero/policy_abc123"
```

When the hero component renders, AEM reads `cq:policy`, finds the policy in `/conf/.../policies/`, and makes those properties available to the component's Sling Model.

---

## 📋 Reading Content Policy in a Sling Model

```java
package com.mysite.core.models;

import com.day.cq.wcm.api.Page;
import com.day.cq.wcm.api.policies.ContentPolicy;
import com.day.cq.wcm.api.policies.ContentPolicyManager;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.models.annotations.DefaultInjectionStrategy;
import org.apache.sling.models.annotations.Model;
import org.apache.sling.models.annotations.injectorspecific.*;

import javax.annotation.PostConstruct;
import java.util.Arrays;
import java.util.Collections;
import java.util.List;

@Model(
    adaptables = Resource.class,
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class TitleModel {

    // Author-configured value (per instance)
    @ValueMapValue
    private String jcrTitle;

    // Author-configured heading level
    @ValueMapValue(name = "type")
    private String headingLevel;

    @SlingObject
    private Resource resource;

    @OSGiService
    private ContentPolicyManager contentPolicyManager;

    // Computed from policy
    private String defaultHeadingLevel;
    private List<String> allowedHeadingLevels;

    @PostConstruct
    protected void init() {
        // Read content policy for this component
        ContentPolicy policy = contentPolicyManager.getPolicy(resource);

        if (policy != null) {
            // Policy properties (set by template author in policy editor)
            defaultHeadingLevel = policy.getProperties()
                                        .get("defaultType", "h2");
            String[] allowed = policy.getProperties()
                                     .get("allowedTypes", new String[]{"h2", "h3", "h4"});
            allowedHeadingLevels = Arrays.asList(allowed);
        } else {
            defaultHeadingLevel = "h2";
            allowedHeadingLevels = Arrays.asList("h2", "h3", "h4");
        }

        // Apply policy default if author didn't pick a heading level
        if (headingLevel == null || headingLevel.isEmpty()) {
            headingLevel = defaultHeadingLevel;
        }

        // Validate: if author's chosen level isn't in the allowed list, use default
        if (!allowedHeadingLevels.contains(headingLevel)) {
            headingLevel = defaultHeadingLevel;
        }
    }

    public String getJcrTitle()              { return jcrTitle; }
    public String getHeadingLevel()          { return headingLevel; }
    public List<String> getAllowedLevels()   { return Collections.unmodifiableList(allowedHeadingLevels); }
}
```

```html
<!-- title.html — Uses heading level from model (which comes from policy + author choice) -->
<sly data-sly-use.model="com.mysite.core.models.TitleModel"/>

<!-- data-sly-element dynamically sets the HTML tag (h1, h2, h3...) -->
<div data-sly-element="${model.headingLevel}"
     class="title"
     data-sly-test="${model.jcrTitle}">
    ${model.jcrTitle}
</div>
```

---

## 🎨 AEM Style System — Making Components Flexible

The **Style System** (AEM 6.3+) allows template authors to define CSS class variations for a component, and page authors to apply those styles visually from the component toolbar — without any code changes.

### The Problem It Solves

Without Style System:
- Developer creates `hero--dark` component variant (separate component)
- Designer wants 5 color variants → 5 separate components
- Template gets cluttered with too many nearly-identical components

With Style System:
- Developer creates ONE hero component with CSS variations
- Template author adds style options: "Light", "Dark", "Blue", "Gradient"
- Page author picks from a visual palette on any hero component

### How Style System Works

**Step 1: Developer creates CSS classes**
```css
/* hero.css */
.hero { /* Base styles */ }
.hero--light { background: #fff; color: #000; }
.hero--dark  { background: #1a1a2e; color: #fff; }
.hero--blue  { background: #0070d2; color: #fff; }
.hero--gradient { background: linear-gradient(135deg, #0070d2, #1a1a2e); color: #fff; }
```

**Step 2: Template author configures styles in Template Editor → Component Policy**
```
Policy: Hero Banner → Styles Tab
  ├── Style Group: "Background"
  │     ├── "Light" → CSS class: hero--light
  │     ├── "Dark"  → CSS class: hero--dark
  │     └── "Blue"  → CSS class: hero--blue
  └── Style Group: "Text Size"
        ├── "Normal" → CSS class: (empty — default)
        └── "Large"  → CSS class: hero--large-text
```

**Step 3: Page author selects style from component toolbar**
- Author clicks hero component → clicks "Style" icon (paintbrush)
- Picks "Dark" + "Large" → AEM applies `hero--dark hero--large-text` to component wrapper

**Step 4: HTL gets styles automatically applied**
```html
<!-- The style classes are automatically added to the component wrapper
     by AEM's component rendering — NO changes needed in HTL! -->

<!-- AEM renders the component with the selected styles as CSS classes: -->
<div class="hero hero--dark hero--large-text">
    <!-- Your HTL content here -->
</div>
```

### Policy XML for Style System

```xml
<!-- /conf/mysite/settings/wcm/policies/mysite/components/content/hero/policy_style/ -->
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
          xmlns:jcr="http://www.jcp.org/jcr/1.0"
          xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
          jcr:primaryType="nt:unstructured"
          jcr:title="Hero Banner - Style Policy">
  <styles jcr:primaryType="nt:unstructured">
    <background jcr:primaryType="nt:unstructured"
                jcr:title="Background Theme">
      <items jcr:primaryType="nt:unstructured">
        <light  jcr:primaryType="nt:unstructured" jcr:title="Light" value="hero--light"/>
        <dark   jcr:primaryType="nt:unstructured" jcr:title="Dark"  value="hero--dark"/>
        <blue   jcr:primaryType="nt:unstructured" jcr:title="Blue"  value="hero--blue"/>
      </items>
    </background>
  </styles>
  <allowedComponents jcr:primaryType="nt:unstructured"/>
</jcr:root>
```

---

## 🔄 Policy vs Dialog — Who Configures What

| Setting | Configured In | By Whom |
|---------|--------------|---------|
| Page title text | Dialog | Page author |
| Whether to show author name | Policy | Template author |
| Allowed heading levels | Policy | Template author / Developer |
| Image crop ratio | Policy | Template author / Developer |
| Style variants (CSS classes) | Policy → Style System | Template author |
| Link URL | Dialog | Page author |
| Which components are allowed | Policy | Template author |

---

## 🔐 Component Group Restrictions

```java
// Allowed components in a container are controlled by policy
// But developers can also restrict via componentGroup:

// In component .content.xml:
// componentGroup=".hidden" → NEVER appears in component browser (for structural components)
// componentGroup="My Site - Content" → Appears in "My Site - Content" group

// In template policy, allowed components reference groups:
// "My Site - Content" group enabled → all components in that group become available
// Individual components can also be allowed: "mysite/components/content/hero"
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is a Content Policy in AEM?**

> **Answer:** A Content Policy is a reusable configuration associated with a component at the template level. It stores settings that apply to all instances of a component within a template's scope — such as allowed heading levels, image aspect ratios, or enabled features. Unlike dialog properties (per-instance, set by page authors), policies are set by template authors and apply uniformly. They're stored in `/conf/<site>/settings/wcm/policies/` and linked to components via the template structure.

**Q2. How does the AEM Style System work end-to-end?**

> **Answer:**
> 1. Developer defines CSS classes for visual variants (`.hero--dark`, `.hero--blue`)
> 2. Developer includes those classes in the component's clientlib CSS
> 3. Template author goes to Template Editor → Component Policy → Styles tab → adds style groups with class names
> 4. Page author in the Page Editor selects the component, clicks the "Style" (paintbrush) icon, and picks a style
> 5. AEM automatically adds the selected CSS class to the component's wrapper `<div>` — no HTL changes needed
> The developer writes one component, business teams manage all visual variants.

**Q3. What is the difference between Content Policy and `cq:design_dialog`?**

> **Answer:** Both configure component behavior at the template/design level rather than per-instance. However:
> - `cq:design_dialog` requires switching to "Design Mode" (a legacy AEM authoring mode) — less intuitive and deprecated
> - Content Policies are configured in the Template Editor's intuitive UI, integrated with modern authoring
> - Content Policies are stored in `/conf/` (content, portable, deployable), while design dialogs write to `/etc/designs/` (legacy path)
> - Content Policies support the Style System; `cq:design_dialog` does not
> For new development, always use Content Policies.

**Q4. How do you read a Content Policy in Java/Sling Model?**

> **Answer:** Inject `ContentPolicyManager` as an OSGi service in your Sling Model, then call `contentPolicyManager.getPolicy(resource)` to get the `ContentPolicy` for the current resource. From the policy, call `getProperties()` to get a `ValueMap` of all policy-configured values. Always null-check the policy since components outside template context may not have one.

**Q5. How do allowed components work in a parsys/grid?**

> **Answer:** Each content container (responsive grid/parsys) in a template has an associated Content Policy that specifies which component groups or individual components are allowed. When an author activates the component browser, AEM reads the policy for the current container and only shows allowed components. The policy lists either entire component groups (e.g., "My Site - Content") or individual resourceTypes. This prevents authors from using inappropriate components in the wrong zones.

---

## ✅ Best Practices

1. **Use Content Policies** for all template-level configuration — avoid `cq:design_dialog`
2. **Combine Style System with one base component** — don't create separate components for visual variants
3. **Keep style classes semantic** — `hero--dark` better than `hero--v2`
4. **Define allowed components in policies** — prevents template pollution
5. **Export policies in `ui.content` package** — they're content, not code
6. **Use `contentPolicyManager.getPolicy(resource)`** — not `resource.getValueMap()` — for policy values

---

## 🛠️ Hands-on Practice

1. **Configure a Title policy:** In Template Editor, restrict allowed heading levels to H2, H3 only. Set default to H2.
2. **Read policy in Java:** Write a Sling Model that reads `allowedTypes` from the Title component's policy.
3. **Style System exercise:**
   - Create a Card component with `.card--featured`, `.card--compact`, `.card--highlight` CSS classes
   - Add these as style options in the component's Content Policy
   - Test that page authors can switch styles from the component toolbar
4. **Allowed Components:** Create a two-column layout template where the left column only allows Text + Image, and the right only allows Form + CTA components
