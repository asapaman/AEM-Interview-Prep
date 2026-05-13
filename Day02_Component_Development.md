# Day 2 — Component Development
**Difficulty:** Medium | **4 YOE Focus**

---

## 📖 Topic Explanation

AEM Components are the building blocks of pages. Each component maps to a JCR node with a `sling:resourceType`, a dialog for author input, and HTL scripts for rendering.

### Component Structure
```
/apps/mysite/components/
  └── content/
        └── mycomponent/
              ├── .content.xml          ← Component definition (jcr:primaryType=cq:Component)
              ├── mycomponent.html      ← HTL rendering script
              ├── _cq_dialog/           ← Touch UI dialog
              │     └── .content.xml
              └── clientlibs/           ← Component-specific JS/CSS
```

### .content.xml (Component Definition)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
          xmlns:cq="http://www.day.com/jcr/cq/1.0"
          xmlns:jcr="http://www.jcp.org/jcr/1.0"
          jcr:primaryType="cq:Component"
          jcr:title="My Text Component"
          sling:resourceSuperType="core/wcm/components/text/v2/text"
          componentGroup="My Site - Content"/>
```

---

## 🔑 Key Concepts

### sling:resourceType
- Maps a JCR content node to a component script folder
- `sling:resourceType="mysite/components/content/hero"` → AEM looks for scripts under `/apps/mysite/components/content/hero/`

### sling:resourceSuperType (Inheritance)
- Allows a component to **inherit** scripts from a parent component
- AEM walks the supertype chain to find the rendering script
- Used heavily with **Core Components** as base

### Core Components vs Custom Components
| Aspect | Core Components | Custom Components |
|--------|----------------|-------------------|
| Maintenance | Adobe-maintained | Team-maintained |
| Editable Templates | Built-in policy support | Must be implemented |
| HTL | Clean, best practice | Developer controlled |
| Best practice | Extend/overlay | Build from scratch |

---

## 📝 Dialog (.content.xml)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
          xmlns:cq="http://www.day.com/jcr/cq/1.0"
          xmlns:jcr="http://www.jcp.org/jcr/1.0"
          xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
          jcr:primaryType="nt:unstructured"
          jcr:title="My Component"
          sling:resourceType="cq/gui/components/authoring/dialog">
  <content jcr:primaryType="nt:unstructured"
           sling:resourceType="granite/ui/components/coral/foundation/fixedcolumns">
    <items jcr:primaryType="nt:unstructured">
      <column jcr:primaryType="nt:unstructured"
              sling:resourceType="granite/ui/components/coral/foundation/container">
        <items jcr:primaryType="nt:unstructured">
          <title jcr:primaryType="nt:unstructured"
                 sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
                 fieldLabel="Title"
                 name="./jcr:title"
                 required="{Boolean}true"/>
          <description jcr:primaryType="nt:unstructured"
                       sling:resourceType="granite/ui/components/coral/foundation/form/textarea"
                       fieldLabel="Description"
                       name="./description"/>
        </items>
      </column>
    </items>
  </content>
</jcr:root>
```

### HTL Rendering Script
```html
<!-- mycomponent.html -->
<sly data-sly-use.model="com.mysite.core.models.MyComponentModel"/>
<div class="my-component">
    <h2>${model.title @ context='html'}</h2>
    <p>${model.description @ context='html'}</p>
</div>
```

---

## ❓ Interview Questions & Answers

**Q1. What is sling:resourceType and why is it important?**
> It's the property on a JCR content node that tells Sling which component script to use for rendering. It's the fundamental link between content and presentation in AEM.

**Q2. What is the difference between resourceType and resourceSuperType?**
> `resourceType` defines WHICH component renders the content. `resourceSuperType` defines the PARENT component to inherit scripts from. If a script isn't found in the current component, Sling walks up the supertype chain.

**Q3. What is cq:dialog vs cq:design_dialog?**
> - `cq:dialog`: Author dialog for editing **instance-level** properties (per-component content)
> - `cq:design_dialog`: Design dialog for editing **design-level** properties (shared across all instances of a component in a template/page design — legacy Classic UI concept)
> In modern AEM, `cq:design_dialog` is replaced by **content policies** in editable templates.

**Q4. How do you extend a Core Component?**
> 1. Create your component with `sling:resourceSuperType` pointing to the core component
> 2. Only override the HTL scripts you need to change
> 3. Add your own dialog fields (merge with core dialog using `sling:hideResource` or granite:hide)
> 4. Use `@ChildResource` in Sling Model to access nested data

**Q5. What is the component group property used for?**
> `componentGroup` determines which group the component appears in when authors open the component browser. Use `".hidden"` to hide a component from authors (e.g., page components, structural components).

**Q6. How does AEM find the HTL script to render a component?**
> Sling script resolution order:
> 1. Look under `/apps/{resourceType}/` for `{resourceType}.html`, `GET.html`, etc.
> 2. Walk `sling:resourceSuperType` chain
> 3. Fall back to `nt:unstructured` default rendering

---

## ✅ Best Practices

- Always extend **Core Components** instead of building from scratch
- Use `sling:resourceSuperType` for inheritance — avoid copy-pasting HTL
- Name dialog fields with `./` prefix to store relative to current node (e.g., `name="./title"`)
- Use `componentGroup=".hidden"` for page/structural components not meant for author drag-drop
- Keep HTL logic minimal — move business logic to Sling Models
- Use `cq:editConfig` to configure in-place editing, drag-and-drop targets

---

## 🛠️ Hands-on Task

Create a **Text Component** with:
- Dialog with: title (textfield), description (textarea), link (pathfield)
- Sling Model backing it
- HTL rendering the data
