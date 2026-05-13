# Day 8–11 — Sling Models (Complete Deep Dive)
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What is a Sling Model?

A **Sling Model** is a Java class annotated with `@Model` that adapts AEM resources (JCR nodes) or HTTP requests into typed Java objects. It acts as the **bridge between JCR content and HTL templates**.

### Why Sling Models?
**Before Sling Models** (old way — JSP scriptlets):
```jsp
<%-- TERRIBLE — business logic in the template! -->
<%
String title = resource.getValueMap().get("title", "");
String link = resource.getValueMap().get("link", "");
if (link != null && link.startsWith("/content/")) {
    link = link + ".html";
}
%>
<h1><%= title %></h1>
<a href="<%= link %>">Click</a>
```

**With Sling Models** (modern way):
```java
// Clean Java class → testable, maintainable, no scriptlets
@Model(adaptables = Resource.class)
public class MyModel {
    @ValueMapValue
    private String title;
    // ... logic in Java
}
```
```html
<!-- Clean HTL template -->
<sly data-sly-use.model="com.mysite.core.models.MyModel"/>
<h1>${model.title}</h1>
```

### Sling Model Lifecycle
```
1. HTL: data-sly-use.model="com.mysite.core.models.MyModel"
        ↓
2. AEM determines adaptable (Resource or Request)
        ↓
3. Sling creates MyModel instance
        ↓
4. Injectors run: @ValueMapValue, @ChildResource, @OSGiService, etc.
        ↓
5. @PostConstruct method runs (all fields now available)
        ↓
6. HTL calls getters: ${model.title} → model.getTitle()
```

---

## 🔑 The @Model Annotation — All Parameters

```java
@Model(
    // What can this model be adapted FROM?
    // Resource.class = adapt from a JCR resource
    // SlingHttpServletRequest.class = adapt from HTTP request
    // Both = handles either
    adaptables = {SlingHttpServletRequest.class, Resource.class},

    // What interface/class does this model implement?
    // Used when you want to access it via its interface
    adapters = {MyComponentModel.class, ComponentExporter.class},

    // The component's resourceType — used by AEM SPA Editor and Model Exporter
    resourceType = "mysite/components/content/mycomponent",

    // OPTIONAL = failed injections = null (component still renders)
    // REQUIRED = failed injections = model returns null (page may break)
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class MyComponentModelImpl implements MyComponentModel {
    // ...
}
```

---

## 🔄 Resource.class vs SlingHttpServletRequest.class

This is one of the **most common interview questions** on Sling Models.

### Resource.class (Preferred Whenever Possible)
```java
@Model(
    adaptables = Resource.class,
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class ArticleModel {

    // Reads from the current resource's ValueMap (JCR properties)
    @ValueMapValue
    private String title;

    @ValueMapValue
    private String author;

    // Injects the resource itself
    @SlingObject
    private Resource resource;

    // Injects ResourceResolver for JCR navigation
    @SlingObject
    private ResourceResolver resourceResolver;
}
```

**When to use Resource.class:**
- ✅ Component only needs data from its own JCR node
- ✅ Works with Sling Model Exporter (`.model.json`)
- ✅ Simpler, more cacheable
- ✅ Can be used in non-request contexts (workflows, schedulers)
- ❌ Cannot access current page, session, WCM mode, or request parameters

### SlingHttpServletRequest.class
```java
@Model(
    adaptables = SlingHttpServletRequest.class,
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class NavigationModel {

    // Still reads JCR properties from current resource
    @ValueMapValue
    private String title;

    // Request object for URL parameters
    @SlingObject
    private SlingHttpServletRequest request;

    // Current page object (from page manager)
    @ScriptVariable
    private Page currentPage;

    // WCM mode (Edit, Preview, Disabled/Publish)
    @ScriptVariable
    private WCMMode wcmMode;

    // Request attribute (set by a parent servlet or filter)
    @RequestAttribute(name = "someKey")
    private String someValue;

    @PostConstruct
    protected void init() {
        // Can check author vs publish mode
        boolean isEditMode = wcmMode == WCMMode.EDIT;

        // Can access query parameters
        String searchTerm = request.getParameter("q");

        // Current page info
        String pagePath = currentPage.getPath();
        String pageTitle = currentPage.getTitle();
    }
}
```

**When to use SlingHttpServletRequest.class:**
- ✅ Need current page information (`Page currentPage`)
- ✅ Need WCM mode (checking if in Author/Edit mode)
- ✅ Need request parameters (`request.getParameter(...)`)
- ✅ Need session/user information
- ❌ Cannot be used in Sling Model Exporter (always use Resource for exportable models)
- ❌ Cannot be used outside request context

### 💡 Best of Both Worlds (Recommended Pattern)
```java
@Model(
    adaptables = {SlingHttpServletRequest.class, Resource.class},
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class FlexibleModel {
    // This model can be adapted from EITHER a request OR a resource
    // @ValueMapValue works with both (reads current resource's properties)
    @ValueMapValue
    private String title;

    // This will be null when adapted from Resource (no request context)
    @ScriptVariable
    private Page currentPage;

    @PostConstruct
    protected void init() {
        // Null check is important when dual adaptable
        if (currentPage != null) {
            // Request context available
        }
    }
}
```

---

## 📋 Complete Annotations Reference

### @ValueMapValue — Read JCR Properties
```java
// Reads property named same as field
@ValueMapValue
private String title;

// Reads property with different JCR name
@ValueMapValue(name = "jcr:title")
private String pageTitle;

// With default value (field default)
@ValueMapValue
private String theme = "light";  // Default if property is missing

// Read a different node's property using via
@ValueMapValue
@Via("jcr:content")
private String contentTitle;
```

### @Inject — Generic Injector (Use Specific Over Generic)
```java
// @Inject tries multiple injectors — less predictable
// AVOID in favor of specific annotations like @ValueMapValue
@Inject
private String title;  // Works but @ValueMapValue is better

// @Inject is still useful for some cases:
@Inject
private List<String> tags;  // Multi-value property
```

### @ChildResource — Inject Child JCR Nodes
```java
// Inject a single child resource
@ChildResource
private Resource heroImage;   // Gets the node named "heroImage"

// Inject with custom name
@ChildResource(name = "cq:responsive")
private Resource responsiveGrid;

// Inject list of child resources (for multifields!)
@ChildResource
private List<Resource> items;   // Gets child node named "items"
// Each child of "items" becomes a Resource in the list

// IMPORTANT: Always check for null!
// @ChildResource returns null if child doesn't exist
```

### @SlingObject — Framework Objects
```java
@SlingObject
private Resource resource;              // Current resource

@SlingObject
private ResourceResolver resourceResolver; // For JCR navigation

@SlingObject
private SlingHttpServletRequest request;   // HTTP request (Request adaptable only)

@SlingObject
private SlingHttpServletResponse response; // HTTP response
```

### @ScriptVariable — Page Editor Variables
```java
@ScriptVariable
private Page currentPage;           // The current AEM page

@ScriptVariable
private PageManager pageManager;    // Create/move/copy pages

@ScriptVariable
private Designer currentDesigner;   // Design info (legacy)

@ScriptVariable
private WCMMode wcmMode;            // EDIT, PREVIEW, DISABLED

@ScriptVariable
private ComponentContext componentContext; // Component context
```

### @OSGiService — Inject OSGi Services
```java
@OSGiService
private TagManager tagManager;     // AEM tag management

@OSGiService
private ModelFactory modelFactory; // Create Sling Models programmatically

@OSGiService
private MyCustomService myService; // Your custom OSGi service

// Optional service (won't fail if service not available)
@OSGiService(injectionStrategy = InjectionStrategy.OPTIONAL)
private OptionalFeatureService optFeature;
```

### @Self — Adapt the Same Resource to Another Model
```java
// Adapt current resource to another Sling Model
@Self
private ImageModel imageModel;

// Adapt to parent component's model (Core Component delegation pattern)
@Self
@Via(type = ResourceSuperType.class)
private com.adobe.cq.wcm.core.components.models.Image coreImageModel;
```

### @RequestAttribute — Read Request Attributes
```java
// Get an attribute set by a parent servlet/filter
@RequestAttribute(name = "productData")
private ProductData productData;
```

---

## 🔄 @PostConstruct — Initialization After Injection

```java
@Model(adaptables = Resource.class, defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class ProductCardModel {

    @ValueMapValue
    private String productId;

    @ValueMapValue
    private String title;

    @ValueMapValue
    private String price;

    @SlingObject
    private ResourceResolver resourceResolver;

    @OSGiService
    private ProductApiService productService;

    @ChildResource
    private List<Resource> tags;

    // These are COMPUTED in @PostConstruct, not injected
    private String formattedPrice;
    private List<String> tagNames;
    private Product productDetails;
    private boolean isAvailable;

    @PostConstruct
    protected void init() {
        // 1. NULL CHECKS FIRST
        if (productId == null || productId.isEmpty()) {
            return; // Can't do anything without productId
        }

        // 2. FORMAT DATA
        if (price != null) {
            formattedPrice = "₹ " + String.format("%.2f", Double.parseDouble(price));
        }

        // 3. PROCESS CHILD RESOURCES
        tagNames = new ArrayList<>();
        if (tags != null) {
            for (Resource tagResource : tags) {
                String tagValue = tagResource.getValueMap().get("tag", String.class);
                if (tagValue != null) {
                    tagNames.add(tagValue);
                }
            }
        }

        // 4. CALL EXTERNAL SERVICE (cached in service layer)
        try {
            productDetails = productService.getProduct(productId);
            isAvailable = productDetails != null && productDetails.isInStock();
        } catch (Exception e) {
            // Log and gracefully degrade
            isAvailable = false;
        }
    }

    // Getters
    public String getTitle() { return title; }
    public String getFormattedPrice() { return formattedPrice; }
    public List<String> getTagNames() { return tagNames; }
    public boolean isAvailable() { return isAvailable; }
}
```

---

## 👶 @ChildResource — Mapping Multifield Data

This is **critical for real-world AEM development**. Multifields (repeatable dialog rows) create child nodes in JCR.

### Dialog: Multifield for "Links"
```xml
<links
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/multifield"
    composite="{Boolean}true"
    fieldLabel="Navigation Links">
  <field
      jcr:primaryType="nt:unstructured"
      sling:resourceType="granite/ui/components/coral/foundation/container">
    <items jcr:primaryType="nt:unstructured">
      <label
          jcr:primaryType="nt:unstructured"
          sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
          fieldLabel="Label"
          name="label"/>
      <url
          jcr:primaryType="nt:unstructured"
          sling:resourceType="granite/ui/components/coral/foundation/form/pathfield"
          fieldLabel="URL"
          name="url"/>
    </items>
  </field>
</links>
```

### JCR Result After Author Adds 2 Links:
```
/content/mysite/home/jcr:content/root/navigation/
  └── links/
        ├── item0/          ← First multifield entry
        │     ├── label = "Home"
        │     └── url = "/content/mysite/en/home"
        └── item1/          ← Second multifield entry
              ├── label = "About"
              └── url = "/content/mysite/en/about"
```

### Sling Model Mapping:
```java
// Inner POJO for each link
public static class NavLink {
    private String label;
    private String url;

    public NavLink(String label, String url) {
        this.label = label;
        this.url = url;
    }

    public String getLabel() { return label; }
    public String getUrl() { return url; }
}

// Main model
@Model(adaptables = Resource.class, defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class NavigationModel {

    @SlingObject
    private ResourceResolver resourceResolver;

    // @ChildResource(name = "links") injects the "links" node's children
    @ChildResource
    private List<Resource> links;

    private List<NavLink> navLinks;

    @PostConstruct
    protected void init() {
        navLinks = new ArrayList<>();
        if (links != null) {
            for (Resource linkResource : links) {
                ValueMap props = linkResource.getValueMap();
                String label = props.get("label", String.class);
                String url = props.get("url", String.class);

                // Resolve internal path to .html URL
                if (url != null && url.startsWith("/content/")) {
                    url = resourceResolver.map(url) + ".html";
                }

                if (label != null && url != null) {
                    navLinks.add(new NavLink(label, url));
                }
            }
        }
    }

    public List<NavLink> getNavLinks() { return navLinks; }
}
```

### HTL Using the Model:
```html
<nav class="navigation">
    <ul class="nav__list">
        <li class="nav__item" data-sly-list.link="${model.navLinks}">
            <a href="${link.url @ context='uri'}" class="nav__link">
                ${link.label}
            </a>
        </li>
    </ul>
</nav>
```

---

## 🐛 Debugging Sling Models — Step by Step

When a Sling Model returns `null` or data is missing:

```
Step 1: Check bundle status
→ http://localhost:4502/system/console/bundles
→ Search your bundle name → Must be "Active"
→ If "Installed" or "Resolved" → Check for compilation errors

Step 2: Check model registration
→ http://localhost:4502/system/console/status-adapters.txt
→ Search for your model class name

Step 3: Check OSGi error log
→ /system/console/errors
→ Look for your bundle name in errors

Step 4: Enable debug logging
→ /system/console/slinglog
→ Add logger: org.apache.sling.models.impl = DEBUG

Step 5: Common issues checklist
□ Wrong adaptable (using Resource when Request is needed for @ScriptVariable)
□ Missing DefaultInjectionStrategy.OPTIONAL (model returns null silently)
□ @ChildResource returns null → child node doesn't exist in JCR
□ @OSGiService is null → service not registered (bundle not active)
□ Getter method missing or misspelled (HTL calls getXxx() / isXxx())
□ Package not exported in bundle manifest
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is a Sling Model? Why do we use it instead of JSP scriptlets?**

> **Answer:** A Sling Model is an OSGi-registered Java POJO annotated with `@Model` that adapts AEM resources or requests into typed objects used in HTL templates. We use Sling Models instead of JSP scriptlets because:
> 1. **Separation of concerns** — Business logic in Java, presentation in HTML
> 2. **Testability** — Models can be unit-tested with `AemContext` (AEM Mocks)
> 3. **Clean code** — No mixing of Java and HTML
> 4. **Type safety** — Strongly typed properties vs generic ValueMap lookups
> 5. **Dependency injection** — OSGi services injected automatically

**Q2. What is the difference between `@ValueMapValue` and `@Inject`?**

> **Answer:** `@ValueMapValue` is a specific injector that reads from the resource's `ValueMap` (JCR properties). It's explicit and predictable — you know exactly where the value comes from. `@Inject` is a generic injector that tries multiple injection strategies in order (ValueMap, OSGi services, Sling objects, etc.). Because `@Inject` is ambiguous, it's slower and less predictable. Prefer `@ValueMapValue` for JCR properties, `@OSGiService` for services, and `@SlingObject` for Sling framework objects over `@Inject`.

**Q3. What is `DefaultInjectionStrategy.OPTIONAL` and why should you always use it?**

> **Answer:** When a model uses `DefaultInjectionStrategy.REQUIRED` (the default), if ANY injection fails (e.g., a JCR property is missing), the entire model adaptation returns `null`. With `OPTIONAL`, failed injections simply result in `null` field values, but the model is still returned successfully. Always use `OPTIONAL` because:
> 1. Not all component properties are required at all times (author may not fill every field)
> 2. Missing JCR properties are normal and expected
> 3. A null model crashes the page for all users; a model with some null fields can gracefully degrade

**Q4. When would you use `Resource.class` vs `SlingHttpServletRequest.class` as adaptable?**

> **Answer:** Use `Resource.class` when the model only needs data from the current JCR resource — simpler, works with Sling Model Exporter, can be used outside request contexts. Use `SlingHttpServletRequest.class` when you need request-specific context: current page (`@ScriptVariable Page currentPage`), WCM mode, request parameters, or user session. As a rule of thumb: if you can achieve what you need with `Resource.class`, use it.

**Q5. Why is `@PostConstruct` needed? Can't we do initialization in the constructor?**

> **Answer:** Injection (all the `@ValueMapValue`, `@ChildResource`, `@OSGiService` etc.) happens AFTER the constructor runs. The constructor only creates the object — fields are still null. `@PostConstruct` runs after ALL injections are complete, so all your `@ValueMapValue` fields are populated and `@OSGiService` references are available. This is where you should do: null checks, data transformation, derived property calculation, and API calls.

**Q6. How do you map a composite multifield to a list of POJOs?**

> **Answer:** Use `@ChildResource` to inject `List<Resource>`, where each Resource in the list is one multifield entry (a child node). In `@PostConstruct`, iterate the list, read each resource's `ValueMap` for the field values, and build your POJO list. Check for null before iterating because `@ChildResource` returns null if the multifield node doesn't exist (no entries added by author).

**Q7. What is `@Self` with `@Via(type = ResourceSuperType.class)` used for?**

> **Answer:** This pattern is used when extending Core Components. It adapts the current resource to the PARENT component's Sling Model:
```java
@Self
@Via(type = ResourceSuperType.class)
private com.adobe.cq.wcm.core.components.models.Image coreImageModel;
```
> Your custom model delegates to the Core Component model, getting all its data (URL resolution, responsive image handling, etc.) while adding your own custom fields. This is the proper way to extend Core Components in Java.

---

## ✅ Best Practices

1. **Always** use `DefaultInjectionStrategy.OPTIONAL`
2. **Prefer specific** annotations over `@Inject` (`@ValueMapValue`, `@OSGiService`, `@SlingObject`)
3. **Null check** everything from `@ChildResource` before iterating
4. **Use `Resource.class`** adaptable unless you truly need request context
5. **Define an interface** and use it as the `adapters` value — program to interface
6. **Keep `@PostConstruct` focused** — complex work belongs in dedicated OSGi services
7. **Never store** `ResourceResolver` or `Session` as class fields in a service — they're request-scoped
8. **Write unit tests** using `AemContext` (io.wcm.testing.mock.aem or io.wcm.testing.mock.sling)
9. **Avoid calling** external APIs in `@PostConstruct` without caching — it slows down every page render

---

## 🛠️ Hands-on Practice

**Day 8**: Create an Article model with `title`, `author`, `publishDate`, `tags[]` (ValueMapValue)
**Day 9**: Refactor to SlingHttpServletRequest adaptable — add `currentPageTitle` from `@ScriptVariable`
**Day 10**: Add `@OSGiService` for a tag translation service
**Day 11**: Map a multifield (links) with label + URL to `List<LinkItem>` POJOs in `@PostConstruct`
