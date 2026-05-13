# Day 4 — Editable Templates
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What Are Editable Templates?

Editable Templates (introduced in AEM 6.2, standard from 6.3+) allow **template authors** — not developers — to create and manage page templates directly in the AEM UI without writing a single line of code.

### The Problem They Solve

Before Editable Templates, every template was hard-coded by developers. Business teams had to raise tickets to developers to:
- Create a new landing page template
- Change which components are allowed on a page
- Lock/unlock certain page sections

With Editable Templates, template authors can:
- Create new page templates from scratch in the AEM Template Editor
- Decide which components are allowed and where
- Lock structural elements (header/footer) that authors can't touch
- Set content policies (e.g., "only allow 2-column layout on Hero")

---

## 📁 Where Templates Live

```
/conf/                                    ← Configuration root
  └── mysite/                             ← Site-specific config
        └── settings/
              └── wcm/
                    ├── templates/        ← All editable templates
                    │     ├── homepage/
                    │     ├── article-page/
                    │     └── landing-page/
                    └── policies/         ← Content policies
                          └── mysite/
                                └── components/
                                      └── content/
                                            └── hero/
                                                  └── policy_1234/
```

### Important: `/conf` vs `/apps`

| Location | Type | Mutable? |
|----------|------|---------|
| `/conf/mysite/settings/wcm/templates/` | Editable Templates | ✅ Yes (modified by template authors) |
| `/apps/mysite/templates/` | Static Templates (legacy) | ❌ Only via package deployment |

---

## 🏗️ Three Layers of an Editable Template

This is the core concept — templates have three distinct layers:

### Layer 1: Structure

**Who manages it:** Template authors / Developers
**What it is:** Fixed components that appear on EVERY page using this template. Authors CANNOT move, edit, or delete these.

```
/conf/mysite/settings/wcm/templates/homepage/structure/
  └── jcr:content/                        [cq:PageContent]
        └── root/                         [wcm/foundation/components/responsivegrid]
              ├── header/                 ← LOCKED — Header appears on all pages
              │     sling:resourceType = "mysite/components/structure/header"
              │     editable = false     ← Authors CANNOT edit this
              └── footer/                 ← LOCKED
                    sling:resourceType = "mysite/components/structure/footer"
```

**In the Template Editor:** Components in the Structure layer have a **padlock icon**. Template authors can lock/unlock individual components.

### Layer 2: Initial Content

**Who manages it:** Template authors
**What it is:** Default content that is COPIED to new pages when they are created from this template. Unlike Structure, authors CAN edit, move, or delete initial content after page creation.

```
/conf/mysite/settings/wcm/templates/homepage/initial/
  └── jcr:content/
        └── root/
              └── hero/                  ← Default hero content
                    jcr:title = "Welcome to Our Site"
                    subtitle = "Discover amazing products"
```

**Analogy:** Structure = permanent wallpaper. Initial Content = furniture that comes with the apartment — you can rearrange or replace it.

### Layer 3: Policies (Formerly "Design")

**Who manages it:** Template authors
**What it is:** Configuration for components — which components are allowed, what style options they have, default settings. Stored in `/conf/mysite/settings/wcm/policies/`.

```
/conf/mysite/settings/wcm/policies/mysite/components/
  └── content/
        └── responsivegrid/
              └── policy_abc123/
                    allowedComponents/
                      ├── mysite/components/content/hero      → ✅ allowed
                      ├── mysite/components/content/text      → ✅ allowed
                      ├── mysite/components/content/video     → ❌ not listed = not allowed
```

---

## 📝 Template Definition — .content.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root
    xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    xmlns:cq="http://www.day.com/jcr/cq/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"

    jcr:primaryType="cq:Template"
    jcr:title="My Site - Homepage"
    jcr:description="Full-width homepage template"

    <!-- Controls where this template CAN be used -->
    allowedPaths="[/content/mysite(/.*)?,/content/mysite]"

    <!-- Which templates can be child pages of pages using this template -->
    allowedChildren="[/conf/mysite/settings/wcm/templates/article-page]"

    <!-- Which template can be the parent -->
    allowedParents="[/conf/mysite/settings/wcm/templates/.*]"

    ranking="{Long}1"    <!-- Lower number = appears first in template picker -->
    status="enabled"/>   <!-- enabled | disabled -->
```

---

## 🔧 Template Type (The Template's Template!)

Every template must be based on a **Template Type**. Template Types define:
- The page component used to render pages (e.g., `mysite/components/page/mysite-page`)
- Available edit capabilities
- Default structure

```
/conf/mysite/settings/wcm/template-types/
  └── mysite-page/
        ├── jcr:content/               ← Template Type config
        │     └── sling:resourceType = "mysite/components/page/mysite-page"
        └── structure/
              └── jcr:content/
                    └── root/          ← Default layout grid
```

**In practice:** Create Template Types once → developers. Create Templates daily → template authors.

---

## 💡 Creating a Template Programmatically (ContentPolicyManager)

```java
import com.day.cq.wcm.api.policies.ContentPolicy;
import com.day.cq.wcm.api.policies.ContentPolicyManager;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.api.resource.ResourceResolver;

// Reading a content policy in a Sling Model
@OSGiService
private ContentPolicyManager policyManager;

@ScriptVariable
private Resource resource;

@PostConstruct
protected void init() {
    // Get the policy for the current resource
    ContentPolicy policy = policyManager.getPolicy(resource);
    if (policy != null) {
        // Read policy properties
        ValueMap policyProps = policy.getProperties();
        String allowedHeadingLevels = policyProps.get("headingLevels", "h1,h2,h3");
        boolean showDate = policyProps.get("showDate", false);
    }
}
```

---

## ✅ Template Allowed Components — Configuration

In the Template Editor:
1. Open template → Structure/Initial layer
2. Click a container (like the main parsys/grid)
3. Click "Policy" icon
4. Select or create a policy
5. Under "Allowed Components", check which component groups are allowed

This creates a policy in `/conf/mysite/settings/wcm/policies/` with:
```xml
<allowedComponents jcr:primaryType="nt:unstructured">
    <mysite_content
        jcr:primaryType="nt:unstructured"
        components="[
            mysite/components/content/hero,
            mysite/components/content/text,
            mysite/components/content/image,
            mysite/components/content/card-grid
        ]"/>
</allowedComponents>
```

---

## ☁️ AEM 6.5 vs Cloud — Template Differences

| Aspect | AEM 6.5 | AEM Cloud Service |
|--------|---------|-----------------|
| **Template storage** | `/conf/<site>/settings/wcm/templates/` | Same |
| **Create via UI** | ✅ Yes — Template Editor | ✅ Yes |
| **Deploy via package** | ✅ Yes | ✅ Yes |
| **Template types** | `/conf/` or `/apps/` | `/apps/` preferred |
| **Policies in package** | ✅ Deployable | ✅ Deployable |
| **Classic templates** | ✅ Still work | ⚠️ Discouraged |

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is an Editable Template and how does it differ from a Static Template?**

> **Answer:** An Editable Template is a modern AEM template (AEM 6.2+) managed by template authors in the AEM UI at `/conf/<site>/settings/wcm/templates/`. It has three layers: Structure (fixed, locked), Initial Content (default, editable by page authors), and Policies (component restrictions). Static templates are developer-created XML files in `/apps/<site>/templates/` with a fixed structure that cannot be changed without code deployment. Editable Templates give business teams control over template management without developer involvement.

**Q2. What is the difference between the Structure layer and the Initial Content layer?**

> **Answer:** The Structure layer contains components that are permanent and locked — they appear on every page using the template and page authors cannot edit, move, or delete them (header, footer, navigation). The Initial Content layer contains components that are COPIED to new pages when they are created — authors CAN edit, move, or delete this content after page creation (e.g., default hero text that each author can customize).

**Q3. What is a Content Policy and why is it preferred over cq:design_dialog?**

> **Answer:** A Content Policy stores configuration for a component that applies globally within a template's scope (e.g., allowed heading levels, image crop ratios). Unlike `cq:design_dialog` (which required entering Design mode, a separate UI), Content Policies are configured in the Template Editor's policy panel — a more intuitive, integrated authoring experience. Policies are also stored in `/conf/` making them part of the content lifecycle, not the code lifecycle. They're the standard from AEM 6.3+ and required for AEM Style System.

**Q4. What is a Template Type?**

> **Answer:** A Template Type is the "template for templates" — it defines which page component renders pages created from templates of this type. It also provides a starting structure. You typically create one Template Type per page component type (standard page, landing page, microsite page). Template authors then create multiple templates from the same Template Type, each with different Structure/Initial Content configurations.

**Q5. How do you restrict which templates can be used under a specific page in the content tree?**

> **Answer:** Use the `allowedPaths` property in the template's `.content.xml`. For example, `allowedPaths="[/content/mysite/blog(/.*)?,/content/mysite/blog]"` restricts this template to only be selectable when creating pages under `/content/mysite/blog/`. Also use `allowedParents` (which parent templates' pages can use this template) and `allowedChildren` (what child templates pages using this template can have).

**Q6. Where are Content Policies stored, and can they be deployed via package?**

> **Answer:** Yes — Content Policies are stored in `/conf/<site>/settings/wcm/policies/` which is part of JCR content. They can be exported via Package Manager and deployed to other environments. In cloud environments, policies authored by template authors must be extracted into a content package (ui.content) and deployed via Cloud Manager pipeline to ensure consistency across dev/stage/prod.

---

## ✅ Best Practices

1. **Always use Editable Templates** for new projects — never Static Templates
2. **Create Template Types** for each distinct page layout pattern
3. **Lock structural components** (header, footer) in the Structure layer so authors can't accidentally delete them
4. **Use Content Policies** to restrict allowed components — don't rely on authors knowing which to use
5. **Name templates clearly** — "Blog Article Page" is better than "template-v2"
6. **Export policies with content packages** — policies should be version-controlled
7. **Set `ranking`** on templates — control the order they appear in the template picker
8. **Use `status="disabled"`** to hide templates that are WIP or deprecated

---

## 🛠️ Hands-on Practice

1. In the AEM Template Editor (`/libs/wcm/core/content/sites/templates.html/conf/mysite`):
   - Create a Template Type with your page component
   - Create a template with a locked Header and Footer in Structure layer
   - Add initial content (default hero text) in Initial layer
   - Configure a policy for the main container allowing only Hero + Text + Image

2. Create a page using your template and verify:
   - Header/footer cannot be edited
   - Initial content IS editable
   - Only allowed components appear in the component browser
