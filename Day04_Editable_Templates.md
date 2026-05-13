# Day 4 — Editable Templates
**Difficulty:** Hard | **4 YOE Focus**

---

## 📖 Topic Explanation

Editable Templates (introduced in AEM 6.2) allow template authors to create and manage templates directly in AEM — without developer involvement. They replaced Static Templates.

### Where They Live
- Template definitions: `/conf/<site>/settings/wcm/templates/`
- Template policies: `/conf/<site>/settings/wcm/policies/`

### Template Structure in JCR
```
/conf/mysite/settings/wcm/templates/my-page-template/
  ├── jcr:content                    ← Template metadata
  │     ├── jcr:title = "My Page"
  │     ├── sling:resourceType = "wcm/core/components/editable-template"
  │     └── cq:deviceGroups = [...]
  ├── structure/                     ← Locked/structural content
  │     └── jcr:content/
  │           └── root/              ← Layout container
  │                 └── header/      ← Fixed (locked) components
  ├── initial/                       ← Default content for new pages
  │     └── jcr:content/
  └── policies/                      ← Policy assignments
        └── jcr:content/
```

---

## 🔑 Key Concepts

### Structure vs Initial Content vs Policies

| Section | Purpose | Editable by Author? |
|---------|---------|-------------------|
| **Structure** | Fixed layout, locked components (header/footer) | ❌ No |
| **Initial** | Default content copied to new pages | ✅ Only at page creation |
| **Policies** | Which components are allowed, style settings | Template authors only |

### Template Status
- **Draft** → Being created, can't be used for pages
- **Enabled** → Active, can be used to create pages
- **Disabled** → Archived, existing pages use it but no new pages

---

## 🔧 Creating a Template — Key Steps

1. Navigate to **Tools → General → Templates → `<site>`**
2. Click **Create** → Select template type (Page template)
3. Define **Structure**: Add locked components (header, footer, nav)
4. Define **Initial content**: Pre-populate editable areas
5. Set **Policies**: Allow specific components in layout containers
6. **Enable** the template
7. Configure **Allowed Templates** on the site root page (`cq:allowedTemplates`)

### Template Path Configuration
```xml
<!-- In page component's .content.xml or via policy -->
<cq:allowedTemplates>
  <path>/conf/mysite/settings/wcm/templates/.*</path>
</cq:allowedTemplates>
```

---

## 📐 Template Policies

Policies define:
- Which **components** can be added to a container
- **Style system** classes available for a component
- Component-specific **design settings** (replacing legacy design_dialog)

Policy assignment in template → `/conf/.../policies/jcr:content/root`:
```xml
<root jcr:primaryType="nt:unstructured"
      cq:policy="/conf/mysite/settings/wcm/policies/mysite/components/core/wcm/components/parsys/default"/>
```

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between Static Templates and Editable Templates?**
> | Aspect | Static Templates | Editable Templates |
> |--------|-----------------|-------------------|
> | Location | `/apps/.../templates/` | `/conf/.../settings/wcm/templates/` |
> | Management | Developer-only | Template authors |
> | Policies | design_dialog | Content Policies |
> | Flexibility | Low | High |
> | Best Practice | Legacy | Current standard |

**Q2. Explain Structure vs Initial Content in Editable Templates.**
> - **Structure**: Components locked for all pages (header, footer). Authors CANNOT edit or delete them. Content is inherited from the template.
> - **Initial Content**: Default editable content. When a new page is created from the template, this content is **copied** to the page. Authors can freely edit it afterward.

**Q3. How do template policies work?**
> Policies are stored in `/conf/.../policies/` and referenced from template structure nodes. They define: allowed components in a layout container, allowed style system classes, and component-specific configuration. Multiple templates can share the same policy.

**Q4. How do you restrict which templates are available when creating a page?**
> Set `cq:allowedTemplates` property on the parent page node with a list of regex patterns for allowed template paths. E.g., `/conf/mysite/settings/wcm/templates/landing-.*` allows only landing page templates.

**Q5. What is a Responsive Grid / Layout Container?**
> The Layout Container (`wcm/foundation/components/responsivegrid`) is the core component that enables drag-and-drop of components in the Editor. Its policy controls allowed components. The Responsive Grid enables breakpoint-based column layouts for responsive pages.

**Q6. How do you troubleshoot a template that isn't showing up when creating a page?**
> Check:
> 1. Template status → must be **Enabled**
> 2. `cq:allowedTemplates` on parent page matches template path
> 3. User has permissions to use the template (`crx:replicate`, `jcr:read` on template)
> 4. Template type exists and is valid

---

## ✅ Best Practices

- Always use **Editable Templates** — never use Static Templates for new projects
- Lock **structural components** (header/nav/footer) in the structure layer
- Create **separate templates** for different page types (landing, article, home)
- Use **policies** to restrict components per container — don't allow ALL components everywhere
- Use `/conf/<site>/` for all template/policy storage — never `/etc/` (deprecated)
- Share policies across templates for consistency

---

## 🛠️ Hands-on Task

Create an editable template:
1. Page template for "Article Page"
2. Lock header and footer in Structure
3. Allow only Text, Image, and Video components in main content area via Policy
4. Set initial content with a placeholder title
