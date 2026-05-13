# Day 28–30 — Content Fragments, Experience Fragments & Sling Model Exporter
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 Content Fragments — AEM's Headless Building Block

### What Are Content Fragments?

A **Content Fragment (CF)** is structured, reusable content stored in AEM's DAM that is **completely separate from its presentation**. Unlike pages (which are content + layout combined), Content Fragments are pure content — they can be consumed by any channel: websites, mobile apps, SPAs, IoT devices.

### Real-World Analogy

Think of Content Fragments as a **database record** with a defined schema, but stored in AEM:
- **Schema** = Content Fragment Model (defines fields)
- **Record** = Content Fragment instance (the actual content)
- **Query** = GROQ/GraphQL API or Sling Model Exporter

### Content Fragments vs Experience Fragments

| | Content Fragment | Experience Fragment |
|--|---|---|
| **Purpose** | Structured content, channel-agnostic | Pre-assembled HTML fragments |
| **Presentation** | NO — pure content only | YES — includes layout |
| **Output** | JSON/XML via API | HTML (or JSON representation) |
| **Reuse** | Across channels (web, app, email) | Within AEM pages or external embedding |
| **Created in** | AEM Assets (DAM) | AEM Sites |
| **Example** | Product description, author bio | Homepage hero, promotional banner |

---

## 📋 Content Fragment Model

A **Content Fragment Model** defines the schema (structure) for a type of content.

```
/conf/mysite/settings/dam/cfm/models/
  └── product/
        ├── jcr:content/
        │     ├── jcr:title = "Product"
        │     └── model/
        │           └── field definitions
```

### Field Types in CF Models

| Field Type | Use Case |
|-----------|---------|
| Single-line text | Title, SKU, short label |
| Multi-line text | Description, body copy |
| Number | Price, quantity |
| Boolean | Is featured, is in stock |
| Date/Time | Publish date, expiry |
| Enumeration | Status (active/inactive), color |
| Tags | AEM tags |
| Content Reference | Reference to another asset/page |
| Fragment Reference | Reference to another Content Fragment |
| JSON Object | Custom structured data |

### Creating a CF Model Programmatically

```java
// Typically done via AEM UI: Assets → Content Fragment Models
// But can also be deployed as JCR structure in ui.content

// After creating the model, authors create fragment instances at:
// Assets > Create > Content Fragment > select model
```

---

## 📄 Content Fragment Instance — Structure in JCR

```
/content/dam/mysite/products/laptop-pro/
  └── laptop-pro (dam:Asset)
        └── jcr:content (dam:AssetContent)
              └── data/
                    └── master/ (content fragment variation)
                          ├── productName = "Laptop Pro 15"
                          ├── sku = "LP-2024-001"
                          ├── price = "1299.99"
                          ├── description = "High-performance laptop..."
                          ├── inStock = "true"
                          └── category = "Electronics"
```

---

## 🔗 Content Fragment API — Accessing CFs in Java

```java
package com.mysite.core.models;

import com.adobe.cq.dam.cfm.ContentFragment;
import com.adobe.cq.dam.cfm.ContentElement;
import com.adobe.cq.dam.cfm.FragmentData;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.api.resource.ResourceResolver;
import org.apache.sling.models.annotations.DefaultInjectionStrategy;
import org.apache.sling.models.annotations.Model;
import org.apache.sling.models.annotations.injectorspecific.*;

import javax.annotation.PostConstruct;
import java.util.*;

@Model(
    adaptables = Resource.class,
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class ProductContentFragmentModel {

    // Path to the Content Fragment, configured in component dialog
    @ValueMapValue
    private String fragmentPath;

    @SlingObject
    private ResourceResolver resourceResolver;

    // Computed fields
    private String productName;
    private String sku;
    private Double price;
    private String description;
    private boolean inStock;
    private List<String> features;

    @PostConstruct
    protected void init() {
        if (fragmentPath == null || fragmentPath.isEmpty()) return;

        // 1. Resolve the fragment resource
        Resource fragmentResource = resourceResolver.getResource(fragmentPath);
        if (fragmentResource == null) return;

        // 2. Adapt to ContentFragment
        ContentFragment fragment = fragmentResource.adaptTo(ContentFragment.class);
        if (fragment == null) return;

        // 3. Read fragment elements (fields defined in the CF Model)
        productName  = getStringElement(fragment, "productName");
        sku          = getStringElement(fragment, "sku");
        description  = getStringElement(fragment, "description");

        // Type-safe element reading
        ContentElement priceElement = fragment.getElement("price");
        if (priceElement != null) {
            FragmentData priceData = priceElement.getValue();
            price = priceData.getValue(Double.class);
        }

        ContentElement inStockElement = fragment.getElement("inStock");
        if (inStockElement != null) {
            FragmentData inStockData = inStockElement.getValue();
            inStock = Boolean.TRUE.equals(inStockData.getValue(Boolean.class));
        }

        // Read multi-value element
        ContentElement featuresElement = fragment.getElement("features");
        if (featuresElement != null) {
            FragmentData featuresData = featuresElement.getValue();
            String[] featuresArr = featuresData.getValue(String[].class);
            features = featuresArr != null ? Arrays.asList(featuresArr) : Collections.emptyList();
        }
    }

    private String getStringElement(ContentFragment fragment, String elementName) {
        ContentElement element = fragment.getElement(elementName);
        if (element == null) return null;
        FragmentData data = element.getValue();
        return data.getValue(String.class);
    }

    // Getters
    public String getProductName()   { return productName; }
    public String getSku()           { return sku; }
    public Double getPrice()         { return price; }
    public String getDescription()   { return description; }
    public boolean isInStock()       { return inStock; }
    public List<String> getFeatures(){ return features; }
    public boolean isConfigured()    { return productName != null; }
}
```

---

## 🌐 Content Fragments + GraphQL (AEM Headless)

AEM Cloud Service (and 6.5 with add-on) supports **GraphQL** for querying Content Fragments.

### Enable Persisted Queries

```graphql
# Create a persisted query named: mysite/getAllProducts
query getAllProducts {
    productList {
        items {
            _path
            productName
            sku
            price
            description
            inStock
        }
    }
}
```

### Calling the API

```javascript
// Client-side JavaScript (React/Angular/Vue)
const response = await fetch(
    '/graphql/execute.json/mysite/getAllProducts',
    {
        headers: {
            'Authorization': 'Bearer ' + accessToken  // For authenticated requests
        }
    }
);
const { data } = await response.json();
const products = data.productList.items;
```

---

## 🧩 Experience Fragments (XF)

### What Are Experience Fragments?

An **Experience Fragment** is a group of AEM components (with layout) that forms a complete, reusable UI block. Think of it as a **"mini-page"** that can be:
- Embedded on multiple AEM pages
- Exported to Adobe Target for A/B testing
- Published as a standalone page
- Sent to email clients

### XF vs CF

```
Content Fragment:
  "Apple MacBook Pro - 16 inch M3 chip, 16GB RAM, $2499"
  (Pure data, no presentation)

Experience Fragment:
  ┌──────────────────────────────┐
  │  [MacBook Image]             │
  │  Apple MacBook Pro           │
  │  M3 chip • 16GB • $2499      │
  │  [Buy Now] [Learn More]      │
  └──────────────────────────────┘
  (Complete visual component, with layout)
```

### XF Structure in JCR

```
/content/experience-fragments/mysite/
  └── promotions/
        └── summer-sale-banner/    (cq:Page)
              └── jcr:content/    (cq:PageContent)
                    ├── cq:template = "/libs/cq/experience-fragments/templates/xfpage"
                    └── root/     (wcm/foundation/components/responsivegrid)
                          ├── hero → sling:resourceType="mysite/components/content/hero"
                          └── text → sling:resourceType="mysite/components/content/text"
```

### Using an XF in a Page

```html
<!-- In a page's parsys, an "Experience Fragment" component references the XF path -->
<!-- The XF is rendered inline at the reference location -->

<!-- Experience Fragment component dialog: author picks XF path -->
<!-- Component renders the XF's HTML inline in the page -->
```

---

## 🔄 Sling Model Exporter — JSON API

**Sling Model Exporter** lets a Sling Model export itself as JSON (or other formats) via a URL with `.model.json` extension — turning your components into REST APIs.

### Use Cases
- **SPA (Single Page Applications):** React/Vue fetches component data as JSON
- **Preview tools:** External tools consume AEM content via JSON
- **Migration:** Export content for importing into another system
- **Headless channels:** Mobile apps consuming AEM content

### Implementation

```java
package com.mysite.core.models;

import com.adobe.cq.export.json.ComponentExporter;
import com.adobe.cq.export.json.ExporterConstants;
import com.fasterxml.jackson.annotation.JsonProperty;
import org.apache.sling.models.annotations.*;
import org.apache.sling.models.annotations.injectorspecific.ValueMapValue;

import javax.annotation.PostConstruct;
import java.util.*;

/**
 * Sling Model with JSON export capability.
 *
 * Key annotations:
 * @Exporter — registers this model for Sling Model Exporter
 * ComponentExporter — interface required for AEM SPA Editor integration
 */
@Model(
    adaptables = Resource.class,
    adapters   = {HeroModel.class, ComponentExporter.class},
    resourceType = "mysite/components/content/hero",
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
@Exporter(
    name      = ExporterConstants.SLING_MODEL_EXPORTER_NAME,  // "jackson"
    extensions = ExporterConstants.SLING_MODEL_EXTENSION       // "model"
    // Together: registers at .model.json
)
public class HeroModel implements ComponentExporter {

    @ValueMapValue
    private String title;

    @ValueMapValue
    private String subtitle;

    @ValueMapValue
    private String ctaLink;

    @ValueMapValue
    private String ctaLabel;

    @ValueMapValue
    private String backgroundImage;

    // Computed
    private String resolvedCtaLink;

    @PostConstruct
    protected void init() {
        if (ctaLink != null && ctaLink.startsWith("/content/")) {
            resolvedCtaLink = ctaLink + ".html";
        } else {
            resolvedCtaLink = ctaLink;
        }
    }

    // ── JSON property getters ──
    // Method name maps to JSON key: getTitle() → "title"

    @JsonProperty("title")
    public String getTitle() { return title; }

    @JsonProperty("subtitle")
    public String getSubtitle() { return subtitle; }

    @JsonProperty("ctaLink")
    public String getCtaLink() { return resolvedCtaLink; }

    @JsonProperty("ctaLabel")
    public String getCtaLabel() { return ctaLabel; }

    @JsonProperty("backgroundImage")
    public String getBackgroundImage() { return backgroundImage; }

    // ComponentExporter interface method — required for SPA Editor integration
    @Override
    public String getExportedType() {
        return "mysite/components/content/hero";
    }
}
```

### Accessing the JSON API

```
Component on page:
/content/mysite/en/home/jcr:content/root/hero

Sling Model Exporter URL:
GET /content/mysite/en/home/jcr:content/root/hero.model.json

Response:
{
    "title": "Welcome to My Site",
    "subtitle": "Discover amazing products",
    "ctaLink": "/en/home.html",
    "ctaLabel": "Shop Now",
    "backgroundImage": "/content/dam/mysite/hero-bg.jpg",
    ":type": "mysite/components/content/hero"
}
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. What are Content Fragments and when do you use them?**

> **Answer:** Content Fragments are structured content pieces stored in AEM's DAM that have NO presentation layer — just pure content. Use them when the same content needs to appear across multiple channels (website, mobile app, email, digital signage) or when content editors need to create structured data (product info, author bios, FAQs) independent of page layout. They're the foundation of AEM's headless capabilities.

**Q2. What is the difference between a Content Fragment and an Experience Fragment?**

> **Answer:**
> - **Content Fragment:** Pure structured content (like a database record). No HTML, no layout. Consumed via APIs (GraphQL, Sling Model Exporter). Channel-agnostic. Perfect for headless.
> - **Experience Fragment:** A visual, fully-assembled HTML fragment with AEM components and layout. Think "mini-page." Can be embedded in AEM pages or exported to Adobe Target/email. Has presentation built in.

**Q3. What is Sling Model Exporter and how does it enable headless delivery?**

> **Answer:** Sling Model Exporter adds `@Exporter(name="jackson", extensions="model")` to a Sling Model, registering it to serve its data as JSON at `<resource-path>.model.json`. It turns any AEM component into a REST API endpoint. Combined with `adapters = ComponentExporter.class` and `resourceType`, it enables the AEM SPA Editor — where React/Angular components can fetch their data from AEM and render it client-side. Enabling headless delivery without changing the Sling Model's core logic.

**Q4. What is a Content Fragment Model?**

> **Answer:** A CF Model is the schema definition for a type of Content Fragment — similar to a database table definition. Created in AEM (`Assets → Content Fragment Models`), it defines the fields (name, type, required, default value) for that content type. For example, a "Product" CF Model might have: productName (text), sku (text), price (number), description (multiline text), inStock (boolean). Authors then create CF instances based on this model.

**Q5. How do you query Content Fragments programmatically in Java?**

> **Answer:** Get the fragment's JCR resource via `ResourceResolver`, adapt to `ContentFragment`, then call `fragment.getElement("fieldName")` → `element.getValue()` → `data.getValue(String.class)`. For typed values use the appropriate type: `Double.class`, `Boolean.class`, `String[].class` for multi-value. Always null-check: `fragment.getElement()` returns null for non-existent elements, and `FragmentData.getValue()` can return null.

**Q6. What is the `getExportedType()` method in `ComponentExporter`?**

> **Answer:** `ComponentExporter` is an interface that Sling Models implement to be compatible with AEM's SPA Editor. `getExportedType()` returns the component's resource type (e.g., `"mysite/components/content/hero"`). This tells the SPA framework (React/Angular) which component to render for this content. Without it, the SPA editor can't map AEM component data to front-end components.

---

## ✅ Best Practices

1. **Use Content Fragments for structured/reusable content** — don't store product data as page properties
2. **Use Experience Fragments for reusable visual blocks** — promotions, footers, modals
3. **Always use `adaptTo(ContentFragment.class)`** for robust field access — not raw ValueMap
4. **Null-check every element** — `getElement()` returns null for missing elements
5. **Use persisted GraphQL queries** — more secure and cacheable than inline queries
6. **Use `Resource.class` adaptable** for Sling Model Exporter (not SlingHttpServletRequest)
7. **Implement `ComponentExporter`** on all models intended for SPA Editor usage
8. **Version your CF Models** — changing a model can break existing fragment instances

---

## 🛠️ Hands-on Practice

**Day 28 — Content Fragments:**
1. Create a CF Model: "Article" with fields: title, author, publishDate, body, tags[]
2. Create 3 Article CF instances under `/content/dam/mysite/articles/`
3. Create a Sling Model that reads a CF given its path (author-configured via dialog)
4. Render the CF data in an HTL template

**Day 29 — Experience Fragments:**
1. Create an XF at `/content/experience-fragments/mysite/header-banner/`
2. Add a Hero component + Text component to it
3. Use the "Experience Fragment" component on 2 different pages to reference the same XF
4. Update the XF and verify the change appears on both pages

**Day 30 — Sling Model Exporter:**
1. Add `@Exporter` + `ComponentExporter` to an existing Sling Model
2. Access it via `.model.json` URL
3. Add a `@JsonProperty("customKey")` annotation to rename a JSON field
4. Write a simple `fetch()` call from the browser console to retrieve the JSON
