# Day 3 — Complex Components
**Difficulty:** Hard | **4 YOE Focus**

---

## 📖 Topic Explanation

At 4 YOE, interviewers expect you to discuss complex, real-world component implementations — not just simple text/image components. Key complexity areas:

1. **Nested Multifields** (array of objects in dialog)
2. **API Integration** (calling external REST APIs)
3. **Performance Optimization** (lazy loading, caching)
4. **CTA/Link handling** (internal vs external links)

---

## 🔁 Nested Multifield

A multifield is a repeatable group of fields in a dialog. A **nested multifield** is a multifield inside another multifield.

### Dialog XML for Nested Multifield
```xml
<tabs jcr:primaryType="nt:unstructured"
      sling:resourceType="granite/ui/components/coral/foundation/form/multifield"
      composite="{Boolean}true"
      fieldLabel="Tabs">
  <field jcr:primaryType="nt:unstructured"
         sling:resourceType="granite/ui/components/coral/foundation/container">
    <items jcr:primaryType="nt:unstructured">
      <tabTitle jcr:primaryType="nt:unstructured"
                sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
                fieldLabel="Tab Title"
                name="tabTitle"/>
      <!-- Nested multifield for tab items -->
      <items jcr:primaryType="nt:unstructured"
             sling:resourceType="granite/ui/components/coral/foundation/form/multifield"
             composite="{Boolean}true"
             fieldLabel="Tab Items">
        <field jcr:primaryType="nt:unstructured"
               sling:resourceType="granite/ui/components/coral/foundation/container">
          <items jcr:primaryType="nt:unstructured">
            <label jcr:primaryType="nt:unstructured"
                   sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
                   fieldLabel="Label"
                   name="label"/>
            <link jcr:primaryType="nt:unstructured"
                  sling:resourceType="granite/ui/components/coral/foundation/form/pathfield"
                  fieldLabel="Link"
                  name="link"/>
          </items>
        </field>
      </items>
    </items>
  </field>
</tabs>
```

### Sling Model for Nested Multifield
```java
@Model(adaptables = Resource.class, defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class HeroBannerModel {

    @ChildResource
    private List<Resource> tabs;

    private List<Tab> tabList;

    @PostConstruct
    protected void init() {
        tabList = new ArrayList<>();
        if (tabs != null) {
            for (Resource tabResource : tabs) {
                Tab tab = new Tab();
                tab.setTitle(tabResource.getValueMap().get("tabTitle", String.class));

                // Read nested multifield
                Resource itemsResource = tabResource.getChild("items");
                if (itemsResource != null) {
                    List<TabItem> items = new ArrayList<>();
                    for (Resource itemRes : itemsResource.getChildren()) {
                        TabItem item = new TabItem();
                        item.setLabel(itemRes.getValueMap().get("label", String.class));
                        item.setLink(itemRes.getValueMap().get("link", String.class));
                        items.add(item);
                    }
                    tab.setItems(items);
                }
                tabList.add(tab);
            }
        }
    }

    public List<Tab> getTabList() { return tabList; }
}
```

---

## 🔗 CTA / Link Handling

Internal vs external link is a critical real-world scenario:

```java
@Model(adaptables = Resource.class, defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class CtaModel {

    @Inject
    private String ctaLink;

    @SlingObject
    private ResourceResolver resourceResolver;

    private String resolvedLink;
    private boolean isExternal;

    @PostConstruct
    protected void init() {
        if (ctaLink != null) {
            isExternal = ctaLink.startsWith("http://") || ctaLink.startsWith("https://");
            if (!isExternal) {
                // Map internal JCR path to URL (uses URL mapping)
                resolvedLink = resourceResolver.map(ctaLink) + ".html";
            } else {
                resolvedLink = ctaLink;
            }
        }
    }

    public String getResolvedLink() { return resolvedLink; }
    public boolean isExternal() { return isExternal; }
}
```

---

## 🌐 API Integration in Component

```java
@Model(adaptables = SlingHttpServletRequest.class,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class ProductListModel {

    @OSGiService
    private ProductApiService productApiService;

    @ValueMapValue
    private String categoryId;

    private List<Product> products;

    @PostConstruct
    protected void init() {
        // Call external API via OSGi service
        products = productApiService.getProductsByCategory(categoryId);
    }

    public List<Product> getProducts() { return products; }
}
```

---

## ❓ Interview Questions & Answers

**Q1. Describe a complex component you built. What challenges did you face?**
> *Sample Answer:* "I built a Hero Banner with tabs, each containing a nested multifield of CTA items. The challenge was mapping the nested JCR structure to Java POJOs using `@ChildResource` and iterating over child resources in `@PostConstruct`. I also had to handle internal/external link differentiation and ensure lazy loading of images for performance."

**Q2. How do you handle a multifield in a Sling Model?**
> Use `@ChildResource` to inject a `List<Resource>` for the multifield nodes. Iterate over them in `@PostConstruct` to build your POJOs. For composite multifields, each item is a child node with properties.

**Q3. How do you optimize a component that calls an external API?**
> 1. Cache the API response in the OSGi service using `Caffeine` or AEM's `MemoryCache`
> 2. Use `@PostConstruct` — call API once per model instantiation
> 3. Consider **WCM Modes** — don't call API in Edit mode if not needed
> 4. Use **Sling Model Exporter** + client-side fetch for dynamic data
> 5. Set appropriate HTTP cache headers on the Dispatcher

**Q4. How do you handle a nested multifield's data in the Sling Model?**
> The nested multifield stores data as child nodes. Use `Resource.getChild("fieldName")` to get the parent, then `resource.getChildren()` to iterate items. Map each to a POJO manually or use `resource.adaptTo(YourModel.class)`.

**Q5. What is the difference between composite and non-composite multifield?**
> - **Non-composite**: Stores repeated values of a SINGLE field as a multi-value property (String[])
> - **Composite**: Stores repeated groups of fields — each group is a separate child node. Requires `composite="{Boolean}true"` in dialog XML.

---

## ✅ Best Practices

- Always use `DefaultInjectionStrategy.OPTIONAL` to avoid null pointer errors
- Cache expensive API calls in the service layer, not in the model
- For composite multifields, always check `null` before iterating child resources
- Separate data fetching (Service) from data modeling (Sling Model) from rendering (HTL)
- Use WCM Mode checks to skip API calls in Author Edit mode
- Test with empty multifield values (no entries) — common NPE source

---

## 🛠️ Hands-on Task

Build a **Hero Banner** component with:
- Background image (pathfield)
- Title, subtitle (textfields)
- Nested multifield: CTA buttons (label + link)
- Sling Model that resolves internal paths to `.html` URLs
