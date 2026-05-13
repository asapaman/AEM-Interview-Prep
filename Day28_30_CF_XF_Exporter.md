# Day 28, 29, 30 — Content Fragments, Experience Fragments & Sling Model Exporter
**Difficulty:** Medium–Hard | **4 YOE Focus**

---

## 📖 Content Fragments (CF)

Content Fragments are **structured, channel-agnostic content** stored in AEM DAM. They separate content from presentation — perfect for headless CMS delivery.

### CF vs Regular Page Content
| Aspect | Page Content | Content Fragment |
|--------|-------------|-----------------|
| Location | `/content/` | `/content/dam/` |
| Format | Component-based | Structured data model |
| Rendering | HTL templates | API / React / Angular |
| Use Case | Web pages | Multi-channel (web, app, IoT) |
| Editor | Page Editor | CF Editor |

### Content Fragment Model
Defined in: `Tools → Assets → Content Fragment Models`

Example Model: **Article**
- `title` (Single line text) → Required
- `body` (Multi-line text / Rich Text)
- `author` (Single line text)
- `publishDate` (Date and time)
- `tags` (Tags)
- `heroImage` (Content reference → DAM asset)

---

## 💻 Reading CF in Sling Model (Day 29)

```java
package com.mysite.core.models;

import com.adobe.cq.dam.cfm.ContentFragment;
import com.adobe.cq.dam.cfm.ContentElement;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.models.annotations.Model;
import org.apache.sling.models.annotations.injectorspecific.Self;
import org.apache.sling.models.annotations.DefaultInjectionStrategy;
import javax.inject.Inject;
import java.util.Calendar;

@Model(
    adaptables = Resource.class,
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class ArticleFragmentModel {

    @Self
    private Resource resource;

    private ContentFragment contentFragment;
    private String title;
    private String body;
    private String author;

    @javax.annotation.PostConstruct
    protected void init() {
        // Adapt resource to ContentFragment API
        contentFragment = resource.adaptTo(ContentFragment.class);

        if (contentFragment != null) {
            // Read elements by name (matches CF model field names)
            title = getElementValue("title");
            body = getElementValue("body");
            author = getElementValue("author");
        }
    }

    private String getElementValue(String elementName) {
        ContentElement element = contentFragment.getElement(elementName);
        if (element != null && element.getContent() != null) {
            return element.getContent().getValue(String.class);
        }
        return null;
    }

    // Access variations
    private String getVariationValue(String elementName, String variationName) {
        ContentElement element = contentFragment.getElement(elementName);
        if (element != null) {
            com.adobe.cq.dam.cfm.ElementTemplate variation = 
                element.getVariation(variationName);
            return variation != null ? variation.getContent().getValue(String.class) : null;
        }
        return null;
    }

    public String getTitle() { return title; }
    public String getBody() { return body; }
    public String getAuthor() { return author; }
}
```

### Using CF in HTL
```html
<!-- Reference the CF from a component dialog (path to DAM CF) -->
<sly data-sly-use.model="${'com.mysite.core.models.ArticleFragmentModel' @ resource=resource.getChild('fragmentResource')}"/>

<!-- Or when the component resource IS the CF -->
<sly data-sly-use.model="com.mysite.core.models.ArticleFragmentModel"/>
<article>
    <h1>${model.title}</h1>
    <p class="author">By ${model.author}</p>
    <div class="body">${model.body @ context='html'}</div>
</article>
```

---

## 🎨 Experience Fragments (XF)

Experience Fragments are **reusable page compositions** — full components/layouts that can be reused across pages and channels.

### CF vs XF
| Aspect | Content Fragment | Experience Fragment |
|--------|----------------|-------------------|
| Content type | Pure content (no layout) | Content + layout + design |
| Stored in | `/content/dam/` | `/content/experience-fragments/` |
| Rendering | Headless (API/app) | Full HTML rendering |
| Reuse | Data reuse across channels | Design reuse across pages |
| Use case | Article data, product info | Header, footer, promo banners |

### Common XF Use Cases
- Global navigation (include in multiple page templates)
- Promotional banners (reused on home + category pages)
- Chat widget (consistent across site)
- Cookie consent dialog

---

## 📤 Sling Model Exporter (Day 30)

Sling Model Exporter serializes a Sling Model to JSON (or other formats) automatically via URL.

### Setup
```java
@Model(
    adaptables = SlingHttpServletRequest.class,
    adapters = {ProductModel.class, com.adobe.cq.export.json.ComponentExporter.class},
    resourceType = "mysite/components/product",
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
@Exporter(name = ExporterConstants.SLING_MODEL_EXPORTER_NAME,
          extensions = ExporterConstants.SLING_MODEL_EXTENSION)
public class ProductModel implements ComponentExporter {

    @ValueMapValue
    private String title;

    @ValueMapValue
    private String price;

    @ValueMapValue
    private String imageUrl;

    public String getTitle() { return title; }
    public String getPrice() { return price; }
    public String getImageUrl() { return imageUrl; }

    @Override
    public String getExportedType() {
        return "mysite/components/product";
    }
}
```

### Access via URL
```
GET /content/mysite/products/laptop.model.json
```

Response:
```json
{
  ":type": "mysite/components/product",
  "title": "MacBook Pro",
  "price": "$1999",
  "imageUrl": "/content/dam/products/laptop.jpg"
}
```

---

## 🔗 GraphQL for Content Fragments (AEMaaCS)

AEMaaCS includes built-in GraphQL API for CF delivery:

```graphql
# Query all Article CFs
query ArticleList {
  articleList {
    items {
      title
      author
      publishDate
      body {
        html
        plaintext
      }
    }
  }
}
```

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between Content Fragment and Experience Fragment?**
> - **Content Fragment**: Pure structured content (text, numbers, references). No visual design. Used for headless delivery via APIs/GraphQL. Think: article data, product catalog.
> - **Experience Fragment**: Full authoring experience (components + layout + design). Renders HTML. Used for design reuse across pages. Think: promo banners, footers.

**Q2. How do you read a Content Fragment in a Sling Model?**
> Adapt the resource to `ContentFragment.class`:
```java
ContentFragment cf = resource.adaptTo(ContentFragment.class);
ContentElement element = cf.getElement("title");
String value = element.getContent().getValue(String.class);
```

**Q3. What are CF Variations?**
> Variations are alternate versions of a CF's content. E.g., an Article CF might have:
> - `master` (full length)
> - `summary` (short version for cards)
> - `social` (social media optimized)
> Access via: `element.getVariation("summary")`

**Q4. What is Sling Model Exporter and what is it used for?**
> Sling Model Exporter automatically serializes a Sling Model to JSON (or XML) when accessed via `.model.json` URL. It's used for:
> 1. **SPA (Single Page Application)**: React/Angular apps fetch component data via JSON API
> 2. **Headless delivery**: Serve AEM content to non-AEM frontends
> 3. **AEM SPA Editor**: The SPA Editor uses Model.json to hydrate the SPA

**Q5. What is `ComponentExporter` interface?**
> Implementing `ComponentExporter` is required for components used in AEM SPA Editor. It provides `getExportedType()` which returns the component's resource type — used by the SPA Editor to map JSON to React/Angular components.

**Q6. What is the GraphQL API in AEMaaCS for CFs?**
> AEMaaCS provides a built-in persisted GraphQL API endpoint for querying Content Fragments. Auto-generates a GraphQL schema based on CF models. Access via:
> - GraphiQL IDE: `/content/graphiql.html`
> - Endpoint: `/content/cq:graphql/global/endpoint.json`

---

## ✅ Best Practices

- Use **Content Fragments** for data that needs to appear in multiple channels
- Use **Experience Fragments** for UI elements reused across pages (header, banners)
- Always **adapt to interface**, not implementation: `resource.adaptTo(ContentFragment.class)`
- Use CF **Variations** for channel-specific content (mobile vs desktop vs email)
- For Sling Model Exporter, implement `ComponentExporter` if used in SPA Editor
- Use **GraphQL persisted queries** (not ad-hoc) in production for security and performance
- Cache CF data in the Sling Model — don't re-fetch in every getter

---

## 🛠️ Hands-on Tasks

**Day 28**: Create a CF model (Article) and an XF (Promo Banner). Compare the two.
**Day 29**: Write a Sling Model that reads an Article CF's title, body, and author variation.
**Day 30**: Add `@Exporter` to a Sling Model and access it via `.model.json` URL.
