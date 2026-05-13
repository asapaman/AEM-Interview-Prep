# Day 3 — Complex Components
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What Makes a Component "Complex"?

Simple components have one or two text fields. Complex components involve:

1. **Nested Multifields** — repeatable groups of fields (e.g., a list of cards, each with image + title + link)
2. **External API Integration** — fetching live product/price data
3. **Advanced CTA Handling** — internal vs external links with different logic
4. **Performance Concerns** — caching, lazy loading, WCM mode checks

At 4 YOE, interviewers expect you to describe **real components you built** — not just text and image.

---

## 🔁 Nested Multifield — Complete Implementation

A **multifield** is a repeatable dialog widget. A **nested multifield** is a multifield inside another multifield — for example, "Tabs" where each tab has its own list of "Items".

### The Data Structure We're Building

```
HeroBanner Component:
  ├── title: "Welcome"
  ├── subtitle: "Explore our products"
  └── tabs: [
        ├── tab[0]:
        │     ├── tabTitle: "Electronics"
        │     └── items: [
        │             { label: "Phones", link: "/content/mysite/phones" },
        │             { label: "Laptops", link: "/content/mysite/laptops" }
        │           ]
        └── tab[1]:
              ├── tabTitle: "Clothing"
              └── items: [
                      { label: "Men", link: "/content/mysite/men" }
                    ]
      ]
```

### JCR Storage of a Composite Multifield

```
/content/mysite/home/jcr:content/root/herobanner/
  ├── title = "Welcome"
  ├── subtitle = "Explore our products"
  └── tabs/                          ← multifield root node
        ├── item0/                   ← First tab (auto-named by AEM)
        │     ├── tabTitle = "Electronics"
        │     └── items/             ← Nested multifield
        │           ├── item0/
        │           │     ├── label = "Phones"
        │           │     └── link = "/content/mysite/phones"
        │           └── item1/
        │                 ├── label = "Laptops"
        │                 └── link = "/content/mysite/laptops"
        └── item1/                   ← Second tab
              ├── tabTitle = "Clothing"
              └── items/
                    └── item0/
                          ├── label = "Men"
                          └── link = "/content/mysite/men"
```

### Dialog XML — Nested Multifield

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
          xmlns:cq="http://www.day.com/jcr/cq/1.0"
          xmlns:jcr="http://www.jcp.org/jcr/1.0"
          xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
          jcr:primaryType="nt:unstructured"
          jcr:title="Hero Banner"
          sling:resourceType="cq/gui/components/authoring/dialog">
  <content jcr:primaryType="nt:unstructured"
           sling:resourceType="granite/ui/components/coral/foundation/fixedcolumns">
    <items jcr:primaryType="nt:unstructured">
      <column jcr:primaryType="nt:unstructured"
              sling:resourceType="granite/ui/components/coral/foundation/container">
        <items jcr:primaryType="nt:unstructured">

          <!-- Simple fields -->
          <title jcr:primaryType="nt:unstructured"
                 sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
                 fieldLabel="Title"
                 name="./title"/>

          <!-- OUTER MULTIFIELD: Tabs -->
          <tabs jcr:primaryType="nt:unstructured"
                sling:resourceType="granite/ui/components/coral/foundation/form/multifield"
                composite="{Boolean}true"
                fieldLabel="Tabs">
            <field jcr:primaryType="nt:unstructured"
                   sling:resourceType="granite/ui/components/coral/foundation/container"
                   name="./tabs">
              <items jcr:primaryType="nt:unstructured">

                <!-- Tab title -->
                <tabTitle jcr:primaryType="nt:unstructured"
                          sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
                          fieldLabel="Tab Title"
                          name="tabTitle"/>

                <!-- INNER MULTIFIELD: Items within each tab -->
                <items jcr:primaryType="nt:unstructured"
                       sling:resourceType="granite/ui/components/coral/foundation/form/multifield"
                       composite="{Boolean}true"
                       fieldLabel="Tab Items">
                  <field jcr:primaryType="nt:unstructured"
                         sling:resourceType="granite/ui/components/coral/foundation/container"
                         name="./items">
                    <items jcr:primaryType="nt:unstructured">
                      <label jcr:primaryType="nt:unstructured"
                             sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
                             fieldLabel="Item Label"
                             name="label"/>
                      <link jcr:primaryType="nt:unstructured"
                            sling:resourceType="granite/ui/components/coral/foundation/form/pathfield"
                            fieldLabel="Item Link"
                            name="link"/>
                    </items>
                  </field>
                </items>

              </items>
            </field>
          </tabs>

        </items>
      </column>
    </items>
  </content>
</jcr:root>
```

### Sling Model — Mapping Nested Multifield

```java
package com.mysite.core.models;

import org.apache.commons.lang3.StringUtils;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.api.resource.ResourceResolver;
import org.apache.sling.api.resource.ValueMap;
import org.apache.sling.models.annotations.DefaultInjectionStrategy;
import org.apache.sling.models.annotations.Model;
import org.apache.sling.models.annotations.injectorspecific.*;

import javax.annotation.PostConstruct;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

@Model(
    adaptables = Resource.class,
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class HeroBannerModel {

    @ValueMapValue
    private String title;

    @ValueMapValue
    private String subtitle;

    // @ChildResource injects the "tabs" child node's direct children as a List<Resource>
    @ChildResource
    private List<Resource> tabs;

    @SlingObject
    private ResourceResolver resourceResolver;

    // Computed result
    private List<Tab> tabList;

    @PostConstruct
    protected void init() {
        tabList = new ArrayList<>();

        if (tabs == null) return;  // No tabs configured — safe to skip

        for (Resource tabResource : tabs) {
            ValueMap tabProps = tabResource.getValueMap();
            Tab tab = new Tab();
            tab.setTitle(tabProps.get("tabTitle", String.class));

            // Navigate into nested "items" node
            Resource itemsNode = tabResource.getChild("items");
            if (itemsNode != null) {
                List<TabItem> itemList = new ArrayList<>();
                for (Resource itemResource : itemsNode.getChildren()) {
                    ValueMap itemProps = itemResource.getValueMap();
                    String label = itemProps.get("label", String.class);
                    String link  = itemProps.get("link",  String.class);

                    // Resolve internal paths
                    if (StringUtils.isNotBlank(link) && link.startsWith("/content/")) {
                        link = resourceResolver.map(link) + ".html";
                    }

                    if (StringUtils.isNotBlank(label)) {
                        itemList.add(new TabItem(label, link));
                    }
                }
                tab.setItems(itemList);
            }

            tabList.add(tab);
        }
    }

    public String getTitle()       { return title; }
    public String getSubtitle()    { return subtitle; }
    public List<Tab> getTabList()  { return Collections.unmodifiableList(tabList); }
    public boolean isHasTabs()     { return !tabList.isEmpty(); }

    // ── Inner POJOs ──

    public static class Tab {
        private String title;
        private List<TabItem> items = new ArrayList<>();
        public String getTitle()          { return title; }
        public void setTitle(String t)    { this.title = t; }
        public List<TabItem> getItems()   { return items; }
        public void setItems(List<TabItem> i) { this.items = i; }
    }

    public static class TabItem {
        private final String label;
        private final String link;
        public TabItem(String label, String link) {
            this.label = label;
            this.link  = link;
        }
        public String getLabel() { return label; }
        public String getLink()  { return link; }
    }
}
```

### HTL Template

```html
<!-- herobanner.html -->
<sly data-sly-use.model="com.mysite.core.models.HeroBannerModel"/>

<section class="hero-banner" data-sly-test="${model.hasTabs}">
    <h1 class="hero-banner__title">${model.title}</h1>
    <p  class="hero-banner__subtitle">${model.subtitle}</p>

    <!-- Tab buttons -->
    <div class="hero-banner__tabs">
        <button class="tab-btn" data-sly-list.tab="${model.tabList}">
            ${tab.title}
        </button>
    </div>

    <!-- Tab content panels -->
    <div class="hero-banner__panels">
        <div class="tab-panel" data-sly-list.tab="${model.tabList}">
            <h3>${tab.title}</h3>
            <ul>
                <li data-sly-list.item="${tab.items}">
                    <a href="${item.link @ context='uri'}">${item.label}</a>
                </li>
            </ul>
        </div>
    </div>
</section>
```

---

## 🔗 CTA / Link Handling — Internal vs External

One of the most common real-world patterns:

```java
@PostConstruct
protected void init() {
    if (StringUtils.isBlank(ctaLink)) return;

    boolean isExternal = ctaLink.startsWith("http://")
                      || ctaLink.startsWith("https://")
                      || ctaLink.startsWith("//");

    if (isExternal) {
        resolvedCtaLink = ctaLink;           // Use as-is
        openInNewTab = true;                 // External → always new tab
    } else {
        // Internal JCR path → add .html suffix
        // resourceResolver.map() applies AEM URL mappings (e.g., /content/mysite → /)
        resolvedCtaLink = resourceResolver.map(ctaLink) + ".html";
    }
}
```

---

## 🌐 API Integration in a Component

```java
@Model(adaptables = Resource.class, defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class ProductCardModel {

    @ValueMapValue
    private String productId;

    @OSGiService
    private ProductApiService productApiService;  // OSGi service handles caching

    private Product product;

    @PostConstruct
    protected void init() {
        if (StringUtils.isBlank(productId)) return;
        // Service handles caching — don't call raw HTTP here
        product = productApiService.getProduct(productId);
    }

    public boolean isAvailable()  { return product != null && product.isInStock(); }
    public String getProductName(){ return product != null ? product.getName() : null; }
    public String getPrice()      { return product != null ? product.getFormattedPrice() : null; }
}
```

**Key Rule:** Never call raw HTTP inside a Sling Model. Put HTTP calls in an OSGi Service that handles caching, retries, and timeouts.

---

## ⚡ Performance Optimization in Components

```java
@PostConstruct
protected void init() {
    // 1. Skip API calls in Author edit mode (expensive, unnecessary)
    if (wcmMode == WCMMode.EDIT || wcmMode == WCMMode.DESIGN) {
        return;
    }

    // 2. Use OSGi service with built-in caching
    products = productService.getProducts(categoryId);

    // 3. Pre-process data once — not in every getter call
    formattedProducts = products.stream()
        .map(p -> new ProductDto(p.getId(), p.getName(), formatPrice(p.getPrice())))
        .collect(Collectors.toList());
}
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. Describe a complex component you built. What were the challenges?**

> **Answer (template):** "I built a Hero Banner component with configurable tabs, each containing a nested multifield of CTA items. Challenges included:
> 1. Mapping the nested JCR structure (tabs → items child nodes) to typed Java POJOs using `@ChildResource` and manual child iteration in `@PostConstruct`
> 2. Handling internal vs external link resolution — internal paths needed `.html` suffix and URL mapping
> 3. Performance — the component fetched product data from an external API, so I moved the API call to a cached OSGi service and added WCM mode checks to skip it in Edit mode
> 4. Null safety — composite multifields store entries as child nodes; if an author adds no entries, the node doesn't exist, so `@ChildResource` returns null"

**Q2. What is the difference between a composite and non-composite multifield?**

> **Answer:**
> - **Non-composite:** Stores repeated values of a SINGLE field as a multi-value property (e.g., `tags = ["aem", "java", "osgi"]`). Defined with `multifield` widget, each entry is a simple string.
> - **Composite:** Stores repeated GROUPS of fields. Each group becomes a child node with multiple properties. Requires `composite="{Boolean}true"` in the dialog XML. Used when each row needs multiple fields (e.g., label + URL + icon).

**Q3. How do you safely iterate a `@ChildResource` list?**

> **Answer:** Always null-check before iterating. `@ChildResource` returns `null` if the named child node doesn't exist (e.g., author added no multifield entries). With `DefaultInjectionStrategy.OPTIONAL`, null is returned silently:
```java
if (tabs != null) {
    for (Resource tab : tabs) {
        // safe
    }
}
// Or with streams:
Optional.ofNullable(tabs).orElse(Collections.emptyList())
        .forEach(tab -> { ... });
```

**Q4. How do you handle the performance impact of an API call in a component?**

> **Answer:** Four strategies:
> 1. **Cache in OSGi service:** Use Caffeine or simple ConcurrentHashMap with TTL in the service layer
> 2. **WCM Mode check:** Skip API calls in Edit/Design mode — `wcmMode == WCMMode.EDIT`
> 3. **Sling Model Exporter + client-side fetch:** Render component shell server-side, fetch data client-side via AJAX for dynamic content
> 4. **Dispatcher cache:** Ensure the rendered HTML is cached at Dispatcher level so AEM only renders once

**Q5. What is `resourceResolver.map()` and why use it for internal links?**

> **Answer:** `resourceResolver.map(path)` applies AEM's URL mapping rules to transform a JCR path into a user-facing URL. For example:
> - `/content/mysite/en/home` → `/en/home` (if URL mapping removes `/content/mysite`)
> - `/content/mysite/en/home` → `/home` (if there's a domain-level mapping)
> Without `.map()`, links would expose the raw JCR structure to users. Always use it for internal links in Sling Models.

---

## ✅ Best Practices

1. **Never call raw HTTP** in a Sling Model — delegate to a cached OSGi service
2. **Always null-check** `@ChildResource` results before iterating
3. **Check WCM mode** before expensive operations — skip API calls in Edit mode
4. **Pre-process in `@PostConstruct`** — build DTOs/POJOs once, not in every getter
5. **Use `Collections.unmodifiableList()`** when returning lists from models
6. **Resolve internal links** with `resourceResolver.map()` + `.html` suffix
7. **External links** — always set `target="_blank"` and `rel="noopener noreferrer"`

---

## 🛠️ Hands-on Practice

Build a **Hero Banner** component with:
1. Dialog: title, subtitle, background image (pathfield)
2. Outer multifield: "Tabs" (composite) with tabTitle
3. Inner multifield: "Items" per tab (composite) with label + link (pathfield)
4. Sling Model: maps nested structure to POJOs, resolves internal links
5. HTL: renders tabs with JS-driven panel switching
