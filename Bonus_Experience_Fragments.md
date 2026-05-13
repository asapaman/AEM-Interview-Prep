# Experience Fragments — Deep Dive
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What Are Experience Fragments?

An **Experience Fragment (XF)** is a group of AEM components (with layout and styling) that together form a complete, reusable visual block. Think of it as a **mini-page** — it has a fully-designed UI that can be:

- **Embedded** in multiple AEM pages (rendered as HTML inline)
- **Exported** to Adobe Target for A/B testing/personalization
- **Published** as a standalone HTML page
- **Exported** as JSON for email systems, third-party CMS, etc.

### The Core Problem XF Solves

```
WITHOUT Experience Fragments:
  Page 1: Copy-paste promotional banner components
  Page 2: Copy-paste same promotional banner components
  Page 3: Copy-paste same promotional banner components

  Marketing says "Update the promo banner text" →
  Editor has to find and update 15+ pages manually 😱

WITH Experience Fragments:
  Create ONE "Summer Sale Banner" XF
  Reference it from Page 1, Page 2, Page 3, ...
  Marketing says "Update the promo banner text" →
  Editor updates ONE XF → all 15+ pages show updated content 🎉
```

---

## 📁 Experience Fragment Structure in JCR

```
/content/experience-fragments/
  └── mysite/
        └── promotions/
              └── summer-sale-banner/          (cq:Page)
                    ├── jcr:content/           (cq:PageContent)
                    │     ├── cq:template = "/libs/cq/experience-fragments/templates/xfpage"
                    │     ├── jcr:title = "Summer Sale Banner"
                    │     └── root/ (wcm/foundation/components/responsivegrid)
                    │           ├── banner/ (mysite/components/content/hero)
                    │           │     ├── jcr:title = "Summer Sale — Up to 50% Off!"
                    │           │     └── ctaLink = "/content/mysite/en/sale"
                    │           └── text/  (mysite/components/content/text)
                    │                 └── text = "<p>Valid until July 31st...</p>"
                    │
                    └── jcr:content/             ← Alternate XF variation
                          (same as above but with different content)
```

---

## 🔀 XF Variations

Just like Content Fragments, Experience Fragments support **variations** — different visual presentations of the same content block for different contexts.

```
/content/experience-fragments/mysite/promotions/summer-sale-banner/
  ├── master/                    ← Default — full web banner
  ├── mobile/                    ← Compact mobile version
  ├── email/                     ← HTML email version (no advanced CSS)
  └── adobe-target/              ← Version formatted for Target export
```

### Built-in XF Types

| XF Type | Purpose | Export Format |
|---------|---------|--------------|
| `Web` (default) | Embedded in AEM pages | HTML |
| `Web - Stand Alone` | Self-contained page | HTML (with full head/body) |
| `Adobe Target - JSON` | A/B testing in Adobe Target | JSON |
| `Adobe Target - HTML` | Target personalization | HTML |
| `Social Media Post` | Social media sharing | Text + image |

---

## 🔗 Using an XF in an AEM Page

### Method 1: Experience Fragment Component (Author)

1. Author adds the **"Experience Fragment"** component to a page
2. Opens its dialog → clicks path picker → selects the XF path
3. AEM renders the XF's HTML inline in the page

```
Component Path: /apps/core/wcm/components/experiencefragment/v2/experiencefragment
```

### Method 2: HTL Include (Developer)

```html
<!-- Include an XF programmatically in HTL -->
<sly data-sly-resource="${'/content/experience-fragments/mysite/promotions/summer-sale-banner/master/jcr:content/root'
    @ resourceType='cq/experience-fragments/editor/components/experiencefragment'}"/>

<!-- OR more cleanly via a Sling Model that resolves the XF path -->
<sly data-sly-use.model="com.mysite.core.models.XfIncludeModel"/>
<sly data-sly-resource="${model.xfPath}"/>
```

### Method 3: Sling Include in Java

```java
// In a servlet or component, programmatically include an XF
RequestDispatcher dispatcher = slingRequest.getRequestDispatcher(
    "/content/experience-fragments/mysite/header/master/jcr:content/root"
);
if (dispatcher != null) {
    dispatcher.include(slingRequest, slingResponse);
}
```

---

## 🎯 Exporting XF to Adobe Target

Experience Fragments can be exported to **Adobe Target** as offers for A/B testing and personalization.

### Setup

```
AEM → Tools → Cloud Services → Legacy Cloud Services
→ Adobe Target → Create Configuration
→ Client Code: YOUR_ANALYTICS_CLIENT_CODE
→ Authentication Method: Bearer Token

Then:
AEM → Tools → Cloud Services → Adobe Target → (your config)
→ Associated Site: /content/mysite
```

### Export Process

```
Sites Console → Select XF page
→ Properties → Adobe Target tab → Export to Target → Create Offer

AEM sends XF HTML to Target as an "Offer"
In Adobe Target → Activities → Create A/B Test
→ Add offer (your XF) as a variation
→ Target determines which variant to show each user
```

### Programmatic Export

```java
import com.adobe.cq.target.TargetIntegrationService;

@Reference
private TargetIntegrationService targetService;

public void exportToTarget(String xfPath, ResourceResolver resolver) {
    Resource xfResource = resolver.getResource(xfPath);
    if (xfResource != null) {
        targetService.exportExperienceFragment(xfResource);
        LOG.info("XF exported to Target: {}", xfPath);
    }
}
```

---

## 💻 Reading XF Data in Java

```java
package com.mysite.core.models;

import com.adobe.cq.wcm.core.components.models.ExperienceFragment;
import com.day.cq.wcm.api.Page;
import com.day.cq.wcm.api.PageManager;
import org.apache.sling.api.SlingHttpServletRequest;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.api.resource.ResourceResolver;
import org.apache.sling.models.annotations.*;
import org.apache.sling.models.annotations.injectorspecific.*;

import javax.annotation.PostConstruct;

@Model(
    adaptables = SlingHttpServletRequest.class,
    adapters = ExperienceFragment.class,
    resourceType = "mysite/components/content/xf-include",
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class XfIncludeModel implements ExperienceFragment {

    // XF path configured by author in component dialog
    @ValueMapValue
    private String fragmentVariationPath;  // Full path to XF variation

    @SlingObject
    private ResourceResolver resourceResolver;

    @ScriptVariable
    private Page currentPage;

    private String resolvedXfPath;
    private boolean xfExists;

    @PostConstruct
    protected void init() {
        if (fragmentVariationPath == null) return;

        Resource xfResource = resourceResolver.getResource(fragmentVariationPath);
        if (xfResource != null) {
            xfExists = true;
            resolvedXfPath = fragmentVariationPath + "/jcr:content/root";
        } else {
            xfExists = false;
            LOG.warn("XF not found at path: {}", fragmentVariationPath);
        }
    }

    public String getResolvedXfPath()  { return resolvedXfPath; }
    public boolean isXfExists()        { return xfExists; }
    public String getFragmentPath()    { return fragmentVariationPath; }

    @Override
    public String getExportedType() {
        return "mysite/components/content/xf-include";
    }
}
```

```html
<!-- xf-include.html -->
<sly data-sly-use.model="com.mysite.core.models.XfIncludeModel"/>

<div class="xf-include" data-sly-test="${model.xfExists}">
    <sly data-sly-resource="${model.resolvedXfPath}"/>
</div>

<!-- Author placeholder when XF not configured -->
<div class="xf-include xf-include--empty"
     data-sly-test="${!model.xfExists}">
    <p>No Experience Fragment configured. Please select one in the component dialog.</p>
</div>
```

---

## 🏗️ XF Building Blocks — Shared Across XF Variations

**Building Blocks** are reusable pieces within an XF. Think of them as "components within a component" that can be shared across multiple XF variations.

```
Summer Sale Banner XF:
├── master variation
│     ├── [Shared: Header Logo Building Block]    ← Used in all variations
│     ├── [Promo Text: "Summer Sale 50% Off!"]
│     └── [CTA Button: "Shop Now"]
└── mobile variation
      ├── [Shared: Header Logo Building Block]    ← SAME building block reused
      ├── [Promo Text: "50% Off!"]               (shorter for mobile)
      └── [CTA Button: "Shop"]                   (shorter)
```

---

## 🔄 XF vs Component — When to Use Which

| Scenario | Use |
|----------|-----|
| Header/Footer used on every page | XF (reusable layout block) |
| Promotional banner for seasonal campaign | XF (update once, reflects everywhere) |
| Simple text-image card | Component (not complex enough for XF) |
| Cookie consent popup | XF (shared across all pages) |
| A/B test banner in Target | XF (export to Target) |
| Complex form | Component or dedicated page |
| Navigation menu | XF or component (depends on complexity) |

---

## ☁️ AEM 6.5 vs Cloud — XF Differences

| Feature | AEM 6.5 | AEM Cloud Service |
|---------|---------|-----------------|
| **XF location** | `/content/experience-fragments/` | Same |
| **Target export** | Via Cloud Service config | Same (with IMS-based auth) |
| **XF API** | Limited | Improved REST API |
| **Universal Editor** | Not available | Available for XF authoring |
| **Preview** | Standard | XF-specific preview URL |

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is an Experience Fragment and how does it differ from a Content Fragment?**

> **Answer:**
> - **Content Fragment:** Pure structured data — no HTML, no layout. Consumed via GraphQL/REST API. Perfect for headless delivery across any channel. Example: product data (name, price, description).
> - **Experience Fragment:** A fully designed visual block — HTML components with layout and styling. Embedded in AEM pages or exported to Adobe Target. Example: a promotional banner with image, headline, CTA button.
> CF = data without presentation. XF = presentation with content.

**Q2. What is an XF variation and when would you use different variations?**

> **Answer:** Variations are different presentations of the SAME XF for different contexts or channels. For example, a "Summer Sale Banner" might have a `master` (full web version), `mobile` (compact), `email` (HTML email safe — no complex CSS), and `adobe-target` (formatted for Target A/B test). All variations share the same author intent but have channel-appropriate presentation.

**Q3. How do you export an XF to Adobe Target?**

> **Answer:** First, configure Adobe Target Cloud Service in AEM (Tools → Cloud Services → Adobe Target). Then: in Sites console, select the XF → Properties → Adobe Target tab → "Export to Target" → creates an offer in Target. In Adobe Target UI, use this offer as a variation in A/B or experience targeting activities. The XF's HTML is sent to Target as an HTML offer.

**Q4. What is the JCR path for Experience Fragments?**

> **Answer:** `/content/experience-fragments/<site>/` — they're stored under `/content`, not under `/apps`. They're content (mutable), not code (immutable). The URL of a typical XF is: `/content/experience-fragments/mysite/promotions/summer-sale-banner/master.html`. When including in a page, you typically reference the variation's `jcr:content/root` path.

**Q5. How does including an XF in multiple pages work for performance?**

> **Answer:** The XF is rendered inline at each referencing page. When an author updates the XF, all pages referencing it show the new content — but the pages themselves are NOT automatically republished. The pages' Dispatcher cache IS invalidated only if the XF is included via the XF component (which triggers replication). To ensure all pages show the updated XF, either: 1) the XF page has its own replication that triggers downstream cache invalidation, or 2) you use TTL-based caching at Dispatcher so pages naturally refresh.

---

## ✅ Best Practices

1. **Use XFs for marketing/promotional content** that needs consistent updates across pages
2. **Name XFs clearly** — "Summer Sale 2024 - Hero Banner - EN" beats "Banner v3"
3. **Use variations** for channel-specific adaptations (mobile, email, Target)
4. **Organize under `/content/experience-fragments/<site>/` folders** by content type
5. **Don't over-XF-ify** — simple components used in one place don't need to be XFs
6. **Test Target export early** — Target integration requires specific configuration
7. **Use Building Blocks** within an XF to share sub-components across variations
8. **Set up proper workflow** for XF approvals — same as page approval workflows

---

## 🛠️ Hands-on Practice

1. Create a "Homepage Hero" XF in `/content/experience-fragments/mysite/homepage/hero/`
2. Add a Hero component and Text component to it
3. Include it on 3 different AEM pages using the Experience Fragment component
4. Update the XF text — verify all 3 pages show the updated content
5. Create a `mobile` variation with simplified content
6. (Advanced) Set up Adobe Target Cloud Service and export the XF as a Target offer
