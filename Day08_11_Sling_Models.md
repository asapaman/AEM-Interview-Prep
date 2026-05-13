# Day 8, 9, 10, 11 — Sling Models (Complete Guide)
**Difficulty:** Medium–Hard | **4 YOE Focus**

---

## 📖 Topic Explanation

Sling Models are POJO classes annotated with `@Model` that adapt JCR content (Resources or Requests) into Java objects for use in HTL templates. They are the backbone of AEM component logic.

---

## 🏗️ Basic Sling Model

```java
package com.mysite.core.models;

import org.apache.sling.api.resource.Resource;
import org.apache.sling.models.annotations.Model;
import org.apache.sling.models.annotations.injectorspecific.ValueMapValue;
import org.apache.sling.models.annotations.DefaultInjectionStrategy;

@Model(
    adaptables = Resource.class,
    adapters = MyComponentModel.class,
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class MyComponentModel {

    @ValueMapValue
    private String title;

    @ValueMapValue
    private String description;

    public String getTitle() { return title; }
    public String getDescription() { return description; }
}
```

### Using in HTL
```html
<sly data-sly-use.model="com.mysite.core.models.MyComponentModel"/>
<h2>${model.title}</h2>
<p>${model.description}</p>
```

---

## 🔑 Adaptables: Resource vs SlingHttpServletRequest

### Resource.class (Preferred when possible)
```java
@Model(adaptables = Resource.class, ...)
public class MyModel {
    @ValueMapValue
    private String title;  // Reads from resource's ValueMap
}
```
**Use when:** Model only needs data from the current JCR resource. Simpler, more cacheable, works with Sling Model Exporter.

### SlingHttpServletRequest.class (When you need request context)
```java
@Model(adaptables = SlingHttpServletRequest.class, ...)
public class MyModel {
    @SlingObject
    private SlingHttpServletRequest request;

    @ScriptVariable
    private Page currentPage;   // Available from request context

    @RequestAttribute
    private String someAttribute;
}
```
**Use when:** Need access to request parameters, session, current page, WCM mode, user info.

### Dual Adaptable (Best of both — recommended pattern)
```java
@Model(
    adaptables = {SlingHttpServletRequest.class, Resource.class},
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class MyModel {
    @ValueMapValue
    private String title;

    @Self  // Works for both adaptables
    @Via(type = ResourceSuperType.class)
    private CoreComponentModel coreModel;
}
```

---

## 📋 All Key Annotations

```java
@Model(adaptables = SlingHttpServletRequest.class,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class FullAnnotationsModel {

    // 1. Read property from resource ValueMap
    @ValueMapValue
    private String title;

    @ValueMapValue(name = "jcr:title")  // Custom JCR property name
    private String pageTitle;

    // 2. Generic injection (tries multiple injectors)
    @Inject
    private String description;

    // 3. Child resource injection
    @ChildResource
    private Resource heroImage;

    @ChildResource(name = "items")
    private List<Resource> listItems;

    // 4. Sling objects
    @SlingObject
    private ResourceResolver resourceResolver;

    @SlingObject
    private Resource resource;

    @SlingObject
    private SlingHttpServletRequest request;

    // 5. Script variables (from request context)
    @ScriptVariable
    private Page currentPage;

    @ScriptVariable
    private Designer currentDesigner;

    // 6. OSGi service injection
    @OSGiService
    private TagManager tagManager;

    @OSGiService
    private MyCustomService myService;

    // 7. Request attribute
    @RequestAttribute
    private String myAttribute;

    // 8. Self-reference (adapter chain)
    @Self
    private MyParentModel parentModel;

    // 9. Via supertype (delegate to parent component)
    @Self
    @Via(type = ResourceSuperType.class)
    private TitleModel titleModel;
}
```

---

## 🔄 @PostConstruct

`@PostConstruct` runs after all injections are complete. Use it for:
- Complex data transformation
- Child resource iteration
- API calls
- Derived property calculation

```java
@Model(adaptables = Resource.class, defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class NavigationModel {

    @SlingObject
    private ResourceResolver resourceResolver;

    @ChildResource
    private List<Resource> navItems;

    private List<NavItem> processedItems;

    @PostConstruct
    protected void init() {
        processedItems = new ArrayList<>();
        if (navItems != null) {
            for (Resource itemResource : navItems) {
                ValueMap props = itemResource.getValueMap();
                NavItem item = new NavItem(
                    props.get("label", String.class),
                    props.get("link", String.class),
                    props.get("openInNewTab", false)
                );
                processedItems.add(item);
            }
        }
    }

    public List<NavItem> getNavItems() { return processedItems; }
}
```

---

## 👶 @ChildResource — Mapping Multifields

```java
@Model(adaptables = Resource.class, defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class AccordionModel {

    @ChildResource
    private List<Resource> items;  // Injects child nodes named "items"

    private List<AccordionItem> accordionItems;

    @PostConstruct
    protected void init() {
        accordionItems = new ArrayList<>();
        if (items != null) {
            for (Resource item : items) {
                accordionItems.add(item.adaptTo(AccordionItemModel.class));
            }
        }
    }
}
```

---

## ❓ Interview Questions & Answers

**Q1. What is a Sling Model and why do we use it?**
> A Sling Model is a Java POJO annotated with `@Model` that maps JCR content to typed Java properties. We use it to separate business logic from HTL templates, making components testable and maintainable.

**Q2. What is the difference between `@ValueMapValue` and `@Inject`?**
> - `@ValueMapValue`: Specifically reads from the resource's `ValueMap` (JCR properties). More explicit and reliable.
> - `@Inject`: Generic injector — tries multiple injectors (ValueMap, request attributes, OSGi services, etc.). Less predictable, generally avoid it in favor of specific annotations.

**Q3. When would you use `Resource.class` vs `SlingHttpServletRequest.class` as adaptable?**
> Use `Resource.class` when the model only needs JCR data — simpler, cacheable, works with exporters.
> Use `SlingHttpServletRequest.class` when needing request context: current page, session, request parameters, WCM mode. At 4 YOE, prefer Resource adaptable and inject ResourceResolver via `@SlingObject` if needed.

**Q4. What does `DefaultInjectionStrategy.OPTIONAL` do?**
> By default (REQUIRED strategy), if ANY injection fails, the model adaptation returns `null`. With `OPTIONAL`, failed injections just result in `null` field values without failing the entire model. Always use OPTIONAL to avoid NPEs from missing JCR properties.

**Q5. How do you adapt a Sling Model from HTL?**
> `<sly data-sly-use.model="com.mysite.core.models.MyModel"/>` — HTL automatically adapts from the current resource (if model adapts from Resource) or from the current request (if adapts from SlingHttpServletRequest).

**Q6. How do you use `@Self` with `@Via(ResourceSuperType)`?**
> This pattern delegates to the parent component's model. Useful when extending Core Components:
```java
@Self
@Via(type = ResourceSuperType.class)
private com.adobe.cq.wcm.core.components.models.Title titleModel;
```
> Your custom model gets all Core Component data + adds your own logic.

**Q7. Why is `@PostConstruct` preferred over a constructor for initialization logic?**
> Constructors run before injection. `@PostConstruct` runs after ALL injections are complete, so all `@ValueMapValue`, `@ChildResource`, `@OSGiService`, etc. are available and non-null.

**Q8. How do you debug a Sling Model that returns null?**
> 1. Check if `adaptables` matches how HTL uses it (Resource vs Request)
> 2. Check `@Model` registration: `/system/console/status-adapters`
> 3. Add logging in `@PostConstruct`
> 4. Verify the OSGi bundle is Active: `/system/console/bundles`
> 5. Check for compilation errors in Felix error log

---

## ✅ Best Practices

- Always use `DefaultInjectionStrategy.OPTIONAL`
- Prefer `@ValueMapValue` over `@Inject` for JCR properties
- Use `@ChildResource` for child nodes, never manually do `resource.getChild()`
- Keep `@PostConstruct` focused — one responsibility
- Use `Resource.class` adaptable unless request context is truly needed
- Make Sling Models unit-testable using `MockSlingHttpServletRequest` / `MockResource` (AemContext)
- Define an interface as the `adapters` value and code to the interface

---

## 🛠️ Hands-on Tasks

**Day 8**: Create an Article Sling Model (Resource adaptable) with title, body, author, tags
**Day 9**: Refactor to SlingHttpServletRequest adaptable — add current page title
**Day 10**: Add `@OSGiService` injection for a tag resolution service
**Day 11**: Map a multifield (List of Resources) to List of POJOs in `@PostConstruct`
