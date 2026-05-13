# Content Fragments — Deep Dive
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What Are Content Fragments?

**Content Fragments (CF)** are structured, reusable content pieces stored in AEM's DAM that have **zero presentation layer** — they contain pure content only. They are the foundation of AEM's **headless** content strategy.

### The Core Idea

```
Traditional Page-Based AEM:
Content + Presentation = Page
(HTML tightly coupled to content)

Headless AEM with Content Fragments:
Content Fragment = Pure content (JSON)
                 +
Frontend = React / Vue / Mobile App / Digital Signage
(Presentation is completely separate)
```

### Why Content Fragments?

| Problem | Content Fragment Solution |
|---------|--------------------------|
| Content locked inside AEM pages | CF content is API-accessible via GraphQL/REST |
| Same content copy-pasted across channels | One CF, referenced everywhere |
| No structure enforcement | CF Model defines required fields and types |
| Editors making formatting decisions | CF is structure-only; presentation is dev's job |
| Hard to reuse in mobile apps | Expose via GraphQL endpoint — any frontend can consume |

---

## 📋 Content Fragment Model — Schema Definition

The **CF Model** is the schema — it defines what fields a content fragment of that type must/can have.

### Creating a CF Model

```
AEM → Tools → Assets → Content Fragment Models → Create

OR navigate to:
/conf/mysite/settings/dam/cfm/models/
```

### CF Model Field Types

```
/conf/mysite/settings/dam/cfm/models/product/
  └── jcr:content/
        └── model/
              └── items/ (list of field definitions)
```

```xml
<!-- CF Model definition (simplified) -->
<items jcr:primaryType="nt:unstructured">
  <productName jcr:primaryType="nt:unstructured"
               fieldLabel="Product Name"
               metaType="text-single"
               required="{Boolean}true"
               maxlength="{Long}100"/>

  <description jcr:primaryType="nt:unstructured"
               fieldLabel="Description"
               metaType="text-multi"
               mimeType="text/html"/>  <!-- Rich text -->

  <price jcr:primaryType="nt:unstructured"
         fieldLabel="Price"
         metaType="number"
         valueType="decimal"/>

  <inStock jcr:primaryType="nt:unstructured"
           fieldLabel="In Stock"
           metaType="boolean"
           defaultValue="true"/>

  <launchDate jcr:primaryType="nt:unstructured"
              fieldLabel="Launch Date"
              metaType="calendar"/>

  <categories jcr:primaryType="nt:unstructured"
              fieldLabel="Categories"
              metaType="tags"/>

  <features jcr:primaryType="nt:unstructured"
            fieldLabel="Features"
            metaType="text-single"
            valueType="array"/>   <!-- Multi-value -->

  <relatedProduct jcr:primaryType="nt:unstructured"
                  fieldLabel="Related Product"
                  metaType="fragmentreference"
                  fragmentModels="[/conf/mysite/settings/dam/cfm/models/product]"/>
</items>
```

---

## 📦 CF Instance Structure in JCR

```
/content/dam/mysite/products/laptop-pro-2024/
  ├── laptop-pro-2024 (dam:Asset)
  └── jcr:content (dam:AssetContent)
        ├── contentFragment = "true"            ← Identifies this as a CF
        ├── model → reference to /conf/.../product model
        └── data/
              └── master/ (dam:AssetContent)    ← Default "master" variation
                    ├── productName = "Laptop Pro 2024"
                    ├── sku = "LP-2024-001"
                    ├── price = "1299.99"
                    ├── description = "<p>High-performance laptop...</p>"
                    ├── inStock = "true"
                    └── features = ["M3 chip", "16GB RAM", "512GB SSD", "15-inch display"]
```

---

## 💻 Reading Content Fragments in Java

### Complete Sling Model with CF

```java
package com.mysite.core.models;

import com.adobe.cq.dam.cfm.*;
import org.apache.commons.lang3.StringUtils;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.api.resource.ResourceResolver;
import org.apache.sling.models.annotations.*;
import org.apache.sling.models.annotations.injectorspecific.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.annotation.PostConstruct;
import java.math.BigDecimal;
import java.util.*;

@Model(
    adaptables = Resource.class,
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class ProductFragmentModel {

    private static final Logger LOG = LoggerFactory.getLogger(ProductFragmentModel.class);

    // Path to the CF — configured by author in component dialog
    @ValueMapValue
    private String fragmentPath;

    // Optionally author picks which variation (default: "master")
    @ValueMapValue
    private String variationName;

    @SlingObject
    private ResourceResolver resourceResolver;

    // Populated from the fragment
    private String productName;
    private String sku;
    private BigDecimal price;
    private String description;
    private boolean inStock;
    private List<String> features;
    private String relatedProductName;

    @PostConstruct
    protected void init() {
        if (StringUtils.isBlank(fragmentPath)) {
            LOG.debug("No fragmentPath configured for ProductFragmentModel");
            return;
        }

        // 1. Get the fragment resource
        Resource fragmentResource = resourceResolver.getResource(fragmentPath);
        if (fragmentResource == null) {
            LOG.warn("Content Fragment not found at path: {}", fragmentPath);
            return;
        }

        // 2. Adapt to ContentFragment
        ContentFragment fragment = fragmentResource.adaptTo(ContentFragment.class);
        if (fragment == null) {
            LOG.warn("Resource at '{}' is not a Content Fragment", fragmentPath);
            return;
        }

        // 3. Determine which variation to use
        String variation = StringUtils.defaultIfBlank(variationName, ContentFragment.DEFAULT_VARIATION_NAME);
        LOG.debug("Reading CF '{}' variation '{}'", fragmentPath, variation);

        // 4. Read each element (field)
        productName = readString(fragment, variation, "productName");
        sku         = readString(fragment, variation, "sku");
        description = readString(fragment, variation, "description");
        inStock     = readBoolean(fragment, variation, "inStock", false);
        features    = readStringArray(fragment, variation, "features");

        // Read decimal (price)
        ContentElement priceElement = getElement(fragment, variation, "price");
        if (priceElement != null) {
            String rawPrice = priceElement.getValue().getValue(String.class);
            if (rawPrice != null) {
                try { price = new BigDecimal(rawPrice); }
                catch (NumberFormatException e) { LOG.warn("Invalid price: {}", rawPrice); }
            }
        }

        // Read fragment reference (related product)
        ContentElement relatedEl = getElement(fragment, variation, "relatedProduct");
        if (relatedEl != null) {
            String relatedPath = relatedEl.getValue().getValue(String.class);
            if (StringUtils.isNotBlank(relatedPath)) {
                Resource relatedResource = resourceResolver.getResource(relatedPath);
                if (relatedResource != null) {
                    ContentFragment relatedFragment = relatedResource.adaptTo(ContentFragment.class);
                    if (relatedFragment != null) {
                        relatedProductName = readString(relatedFragment,
                            ContentFragment.DEFAULT_VARIATION_NAME, "productName");
                    }
                }
            }
        }
    }

    // ── Helper Methods ──

    private ContentElement getElement(ContentFragment fragment, String variation, String elementName) {
        try {
            if (ContentFragment.DEFAULT_VARIATION_NAME.equals(variation)) {
                return fragment.getElement(elementName);
            } else {
                ContentVariation var = fragment.getVariation(variation);
                return var != null ? var.getElement(elementName) : fragment.getElement(elementName);
            }
        } catch (ContentFragmentException e) {
            LOG.debug("Variation '{}' not found — using master", variation);
            return fragment.getElement(elementName);
        }
    }

    private String readString(ContentFragment fragment, String variation, String elementName) {
        ContentElement element = getElement(fragment, variation, elementName);
        if (element == null) return null;
        return element.getValue().getValue(String.class);
    }

    private boolean readBoolean(ContentFragment fragment, String variation,
                                String elementName, boolean defaultValue) {
        ContentElement element = getElement(fragment, variation, elementName);
        if (element == null) return defaultValue;
        Boolean val = element.getValue().getValue(Boolean.class);
        return val != null ? val : defaultValue;
    }

    private List<String> readStringArray(ContentFragment fragment, String variation, String elementName) {
        ContentElement element = getElement(fragment, variation, elementName);
        if (element == null) return Collections.emptyList();
        String[] arr = element.getValue().getValue(String[].class);
        return arr != null ? Arrays.asList(arr) : Collections.emptyList();
    }

    // ── Getters ──
    public String getProductName()   { return productName; }
    public String getSku()           { return sku; }
    public BigDecimal getPrice()     { return price; }
    public String getDescription()   { return description; }
    public boolean isInStock()       { return inStock; }
    public List<String> getFeatures(){ return features; }
    public String getRelatedProductName() { return relatedProductName; }
    public boolean isConfigured()    { return productName != null; }
}
```

---

## 🌐 GraphQL API — AEM Headless

AEM Cloud Service (and 6.5 with add-on) exposes Content Fragments via GraphQL.

### Enabling GraphQL

```
Tools → GraphQL → Endpoints → Create
→ Name: "mysite-graphql"
→ Configuration: /conf/mysite
→ Endpoint URL: /content/cq:graphql/mysite/endpoint.json
```

### GraphQL Schema (Auto-generated from CF Models)

AEM auto-generates GraphQL types from your CF Models. For a "Product" CF Model:

```graphql
# Auto-generated GraphQL types:
type Product {
    _path: ID!
    _metadata: TypedMetaData
    productName: String
    sku: String
    price: Float
    description: MultiLineString
    inStock: Boolean
    features: [String]
    launchDate: Calendar
    relatedProduct: ProductModel  # Nested fragment reference
}

type ProductList {
    items: [Product]
    _references: [Reference]
}
```

### Writing Queries

```graphql
# List all products
query getAllProducts {
    productList {
        items {
            _path
            productName
            sku
            price
            inStock
        }
    }
}

# Get specific product by path
query getProduct($path: String!) {
    productByPath(_path: $path) {
        item {
            productName
            sku
            price
            description { html }
            features
            relatedProduct {
                ... on Product {
                    productName
                    sku
                }
            }
        }
    }
}

# Filter: only in-stock products, ordered by price
query getInStockProducts {
    productList(
        filter: { inStock: { _expressions: [{ value: true }] } }
        sort: "price ASC"
        limit: 10
        offset: 0
    ) {
        items {
            productName
            price
            sku
        }
    }
}
```

### Persisted Queries (Required for Production!)

```bash
# Create a persisted query (via HTTP PUT)
curl -X PUT \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"query":"query getAllProducts { productList { items { productName sku price } } }"}' \
  "https://author.mysite.adobeaemcloud.com/graphql/persist.json/mysite/getAllProducts"

# Execute the persisted query (cacheable by CDN!)
GET /graphql/execute.json/mysite/getAllProducts
```

**Why persisted queries?**
- POST requests with inline queries are not cacheable by CDN/Dispatcher
- Persisted queries are accessible via GET — Fastly CDN can cache responses
- Better security — only pre-approved queries can be executed (no arbitrary GraphQL injection)

### Client-Side Consumption (React)

```javascript
// Using AEM Headless JS SDK
import AEMHeadless from '@adobe/aem-headless-client-js';

const sdk = new AEMHeadless({
    serviceURL: 'https://publish.mysite.adobeaemcloud.com',
    endpoint: '/content/cq:graphql/mysite/endpoint.json'
});

// Execute persisted query
const getProducts = async () => {
    const { data, errors } = await sdk.runPersistedQuery('mysite/getAllProducts');
    if (errors) console.error('GraphQL errors:', errors);
    return data?.productList?.items ?? [];
};

// In a React component
function ProductGrid() {
    const [products, setProducts] = useState([]);

    useEffect(() => {
        getProducts().then(setProducts);
    }, []);

    return (
        <div className="product-grid">
            {products.map(product => (
                <ProductCard key={product._path} product={product} />
            ))}
        </div>
    );
}
```

---

## 🔀 Content Fragment Variations

A CF can have multiple **variations** of its content — different versions for different channels or purposes.

```
Fragment: /content/dam/mysite/products/laptop-pro/
  └── Variations:
        ├── master     (default — full content)
        ├── summary    (short version for listing pages)
        ├── social     (social media text, 280 chars)
        └── mobile     (simplified, no rich text)
```

### Creating and Using Variations

```
CF Editor → Variations tab → + Create Variation
→ Name: "summary"
→ Title: "Summary Version"
→ Edit the summary variation with shorter content

In Java (reading specific variation):
ContentVariation variation = fragment.getVariation("summary");
if (variation != null) {
    ContentElement el = variation.getElement("description");
}
```

---

## ☁️ AEM 6.5 vs Cloud — CF Differences

| Feature | AEM 6.5 | AEM Cloud Service |
|---------|---------|-----------------|
| **CF Models** | `/conf/<site>/settings/dam/cfm/models/` | Same |
| **GraphQL** | Add-on package required | Built-in |
| **Persisted Queries** | Available with add-on | Built-in |
| **Preview** | Limited | CF Preview with separate URL |
| **REST API** | Assets HTTP API | Assets HTTP API + newer CF API |
| **Asset Compute** | Custom DAM workflow steps | Adobe I/O Runtime workers |

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is a Content Fragment Model and how does it relate to a Content Fragment?**

> **Answer:** A CF Model is the schema — it defines the structure (fields, types, validation) for a type of content. A Content Fragment is an instance — actual content data that follows the model's schema. The relationship is like a database table (model) and a record (CF instance). Models are defined once by developers or admins; editors create many CF instances from that model.

**Q2. What is the difference between CF variations and separate CF instances?**

> **Answer:** Variations are alternative versions of the SAME content fragment for different contexts (summary, mobile, social). They share the same CF Model but have different field values. Separate instances are completely independent CFs. Use variations when the same fundamental content item needs different lengths or formats. Use separate instances when the content is fundamentally different data.

**Q3. Why use persisted GraphQL queries instead of inline queries?**

> **Answer:** Three reasons: 1) Persisted queries use HTTP GET, which CDN/Dispatcher can cache — dramatically reducing load on AEM for frequently accessed content. 2) Security — only pre-approved queries can run; prevents arbitrary data exploration via POST queries. 3) Performance — server can pre-validate and optimize the query plan.

**Q4. How do you access a CF in Java vs in a Headless frontend?**

> **Answer:** In Java: Resolve the resource via `ResourceResolver`, adapt to `ContentFragment`, then call `getElement("fieldName")` → `getValue().getValue(String.class)`. In a Headless frontend: Use the GraphQL API with persisted queries via HTTP GET. The Java approach is for server-rendered AEM pages; the GraphQL approach is for SPAs, mobile apps, and external systems.

**Q5. What is `ContentFragment.DEFAULT_VARIATION_NAME`?**

> **Answer:** It's the constant `"master"` — the default variation name for all Content Fragments. When reading CF content programmatically, use this constant rather than hardcoding the string `"master"` to ensure compatibility if Adobe ever changes the default variation name.

---

## ✅ Best Practices

1. **Model design:** Keep CF Models focused — one model per content type (Product, Author, FAQ)
2. **Null-check every element** — `getElement()` returns null for non-existent fields
3. **Use persisted queries in production** — caching and security
4. **Use variations for channel-specific content** (not separate fragments)
5. **Fragment references** for related content — avoid copying data
6. **Store CFs in DAM** under organized paths: `/content/dam/<site>/content-type/`
7. **Enable preview** in Cloud for CF preview before publishing
8. **Version control CF Models** — schema changes can break existing fragments

---

## 🛠️ Hands-on Practice

1. Create a CF Model "Author Profile" with: name, bio (rich text), photo (content reference), expertise tags[]
2. Create 3 Author Profile CF instances in `/content/dam/mysite/authors/`
3. Write a Sling Model that reads an Author Profile CF and renders it on a page
4. Create a "summary" variation for each author (shorter bio)
5. Write a GraphQL query to list all authors with name and expertise
6. Test the persisted query via curl
