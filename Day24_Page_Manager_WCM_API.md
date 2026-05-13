# Page Manager & WCM APIs — AEM Java SDK
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 The WCM Java API Layer

AEM provides a rich Java API for working with Pages, Templates, Tags, Designs, and content management operations. These APIs wrap the underlying JCR with AEM-specific abstractions.

### Key API Classes

| Class/Interface | Package | Purpose |
|----------------|---------|---------|
| `PageManager` | `com.day.cq.wcm.api` | CRUD operations on pages |
| `Page` | `com.day.cq.wcm.api` | Represents an AEM page |
| `Template` | `com.day.cq.wcm.api` | Page template |
| `TagManager` | `com.day.cq.tagging` | Tag creation and management |
| `Tag` | `com.day.cq.tagging` | Single tag |
| `Designer` | `com.day.cq.wcm.api` | Design (legacy — use Content Policies) |
| `WCMMode` | `com.day.cq.wcm.api` | Current authoring mode |
| `ComponentContext` | `com.day.cq.wcm.api.components` | Component rendering context |

---

## 📄 PageManager API — Complete Reference

### Getting PageManager

```java
// Option 1: From ResourceResolver (in Sling Models / Servlets)
@SlingObject
private ResourceResolver resourceResolver;

PageManager pageManager = resourceResolver.adaptTo(PageManager.class);

// Option 2: From ScriptVariable (in Sling Models)
@ScriptVariable
private PageManager pageManager;  // Injected by AEM rendering engine
```

### Reading Pages

```java
// Get a page by path
Page page = pageManager.getPage("/content/mysite/en/home");

if (page != null) {
    // Basic page info
    String path        = page.getPath();           // "/content/mysite/en/home"
    String name        = page.getName();           // "home"
    String title       = page.getTitle();          // From jcr:content/jcr:title
    String navTitle    = page.getNavigationTitle(); // From jcr:content/navTitle (fallback: title)
    String pageTitle   = page.getPageTitle();      // From jcr:content/pageTitle (browser tab)
    String description = page.getDescription();   // From jcr:content/jcr:description
    Calendar created   = page.getProperties().get("jcr:created", Calendar.class);

    // Template
    Template template  = page.getTemplate();
    String templatePath = template != null ? template.getPath() : null;

    // Tags
    Tag[] tags = page.getTags();  // from jcr:content/cq:tags
    for (Tag tag : tags) {
        LOG.info("Tag: {} (ID: {})", tag.getTitle(), tag.getTagID());
    }

    // jcr:content resource
    Resource contentResource = page.getContentResource();
    ValueMap props = page.getProperties();  // shortcut to jcr:content ValueMap

    // Custom property from jcr:content
    String pageType = props.get("pageType", String.class);
    boolean isHidden = props.get("hideInNav", false);

    // Parent page
    Page parent = page.getParent();           // Direct parent
    Page grandParent = page.getParent(2);    // 2 levels up

    // Depth in tree
    int depth = page.getDepth();  // /content = 1, /content/mysite = 2, etc.

    // Check if page has a specific tag
    boolean hasSaleTag = page.hasTag("mysite:campaigns/sale");
}
```

### Creating, Moving, and Deleting Pages

```java
// ⚠️ All write operations need a service user with write permissions!

// Create a page
Page newPage = pageManager.create(
    "/content/mysite/en",                    // Parent path
    "new-article",                           // Page name (URL segment)
    "/conf/mysite/settings/wcm/templates/article",  // Template path
    "My New Article"                         // Title
);
LOG.info("Created page at: {}", newPage.getPath());

// Copy a page
Page copiedPage = pageManager.copy(
    sourcePage,                              // Source Page object
    "/content/mysite/en/new-location",       // Destination path
    "after-page-name",                       // Insert after this sibling (null = last)
    true,                                    // Deep: copy children too
    false,                                   // resolveConflicts
    false                                    // keepAuthorization (keep ACLs)
);

// Move a page
Page movedPage = pageManager.move(
    sourcePage,                              // Page to move
    "/content/mysite/en/new-path",           // New parent path
    null,                                    // Before page (null = last)
    true,                                    // Shallow (false = move children too)
    false,                                   // resolveConflicts
    new String[]{"/content/mysite/en/old-path"}  // Old path for redirects
);

// Delete a page
pageManager.delete(page, false);  // false = don't delete if has live children

// Commit (must call this after operations!)
// Note: PageManager operations typically auto-save, but check documentation
resourceResolver.commit();
```

### Traversing Page Tree

```java
// Iterate all children of a page
Page parent = pageManager.getPage("/content/mysite/en");
Iterator<Page> children = parent.listChildren();

while (children.hasNext()) {
    Page child = children.next();
    LOG.info("Child page: {}", child.getPath());
}

// Filter: only pages that are NOT hidden in nav
Iterator<Page> navPages = parent.listChildren(
    new PageFilter(false, false)  // noInvalid=false, noHidden=false
);

// Custom filter: only article pages
PageFilter articleFilter = new PageFilter() {
    @Override
    public boolean includes(Page page) {
        return "article".equals(page.getProperties().get("pageType", String.class));
    }
};
Iterator<Page> articles = parent.listChildren(articleFilter);

// Get absolute depth-first traversal
Iterator<Page> allDescendants = parent.listChildren(null, true);  // true = recursive
```

---

## 🏷️ TagManager API

### Reading Tags

```java
TagManager tagManager = resourceResolver.adaptTo(TagManager.class);

// Get a tag by ID
Tag tag = tagManager.resolve("mysite:campaigns/summer-2024");
if (tag != null) {
    String title      = tag.getTitle();            // "Summer 2024"
    String tagId      = tag.getTagID();            // "mysite:campaigns/summer-2024"
    String path       = tag.getPath();             // "/content/cq:tags/mysite/campaigns/summer-2024"
    Tag parent        = tag.getParent();           // "campaigns" tag
    Iterator<Tag> children = tag.listChildren();   // Sub-tags
}

// List all tags in a namespace
Tag namespace = tagManager.resolve("mysite:");
if (namespace != null) {
    Iterator<Tag> allMysiteTags = namespace.listChildren();
}
```

### Creating Tags Programmatically

```java
// Create a new tag
Tag newTag = tagManager.createTag(
    "mysite:events/webinar-2024",    // Tag ID (namespace:path)
    "Webinar 2024",                  // Display title
    "Tags for 2024 webinar content", // Description
    true                             // Auto-save: commit immediately
);

// Create tag hierarchy (creates parent tags if missing)
// "mysite:products/electronics/laptops" creates:
// mysite: (if missing)
//   products/ (if missing)
//     electronics/ (if missing)
//       laptops ← creates this
Tag laptopTag = tagManager.createTag(
    "mysite:products/electronics/laptops",
    "Laptops",
    "Laptop product category"
);
```

### Applying Tags to Content

```java
// Apply tags to a page
Resource contentResource = page.getContentResource();
ModifiableValueMap mvm = contentResource.adaptTo(ModifiableValueMap.class);
if (mvm != null) {
    mvm.put("cq:tags", new String[]{
        "mysite:campaigns/summer-2024",
        "mysite:products/electronics/laptops"
    });
    resourceResolver.commit();
}

// Or use TagManager.setTags() for proper tag handling
Tag[] tagsToApply = {
    tagManager.resolve("mysite:campaigns/summer-2024"),
    tagManager.resolve("mysite:products/electronics")
};
tagManager.setTags(contentResource, tagsToApply, true);  // true = merge with existing
```

---

## 🎨 WCMMode — Authoring Context

`WCMMode` tells your component whether it's being rendered in the editor, preview, or live.

```java
import com.day.cq.wcm.api.WCMMode;

// In Sling Model
@ScriptVariable
private WCMMode wcmMode;

// OR: Read from request
WCMMode mode = WCMMode.fromRequest(slingRequest);

// Check mode in @PostConstruct — important for performance
@PostConstruct
protected void init() {
    boolean isEditMode    = WCMMode.EDIT.equals(wcmMode);     // Author: Edit
    boolean isPreviewMode = WCMMode.PREVIEW.equals(wcmMode);  // Author: Preview
    boolean isDisabled    = WCMMode.DISABLED.equals(wcmMode); // Publish / anonymous

    // Skip expensive operations in edit mode (e.g., external API calls)
    if (!isDisabled) {
        // We're in Author — maybe show placeholder instead of real API data
        this.products = Collections.emptyList();  // Save API quota in Author
    } else {
        // We're on Publish — load real data
        this.products = productService.loadProducts();
    }
}
```

```html
<!-- WCMMode check in HTL -->
<sly data-sly-use.wcmmode="com.day.cq.wcm.api.WCMMode"/>

<!-- Show author-only edit helper -->
<div class="edit-hint" data-sly-test="${wcmmode.EDIT}">
    Click to configure this component
</div>

<!-- Check in Granite/Coral UI (simpler) -->
<div data-sly-test="${wcmmode.disabled}">
    <!-- This renders on Publish only -->
</div>
```

---

## 🔄 Page API — Common Patterns

### Navigation Breadcrumb

```java
@Model(adaptables = SlingHttpServletRequest.class, ...)
public class BreadcrumbModel {

    @ScriptVariable
    private Page currentPage;

    @ScriptVariable
    private PageManager pageManager;

    private List<Page> breadcrumbPages = new ArrayList<>();

    @PostConstruct
    protected void init() {
        // Walk up the page tree from current page to root
        Page page = currentPage;
        int rootLevel = 2;  // /content/mysite = level 2, our "homepage" level

        while (page != null && page.getDepth() >= rootLevel) {
            if (!page.getProperties().get("hideInNav", false)) {
                breadcrumbPages.add(0, page);  // Add at beginning (reverse order)
            }
            page = page.getParent();
        }
    }

    public List<Page> getBreadcrumbPages() { return breadcrumbPages; }
}
```

```html
<!-- breadcrumb.html -->
<sly data-sly-use.model="com.mysite.core.models.BreadcrumbModel"/>
<nav aria-label="Breadcrumb">
    <ol class="breadcrumb">
        <li class="breadcrumb__item" data-sly-list.page="${model.breadcrumbPages}">
            <a class="breadcrumb__link"
               href="${page.path}.html @ context='uri'">
                ${page.navigationTitle || page.title}
            </a>
        </li>
    </ol>
</nav>
```

### Navigation Menu

```java
@Model(adaptables = SlingHttpServletRequest.class, ...)
public class NavigationModel {

    @ScriptVariable
    private PageManager pageManager;

    private List<Page> navigationPages;

    @PostConstruct
    protected void init() {
        Page navigationRoot = pageManager.getPage("/content/mysite/en");
        if (navigationRoot == null) return;

        navigationPages = new ArrayList<>();
        Iterator<Page> children = navigationRoot.listChildren(
            new PageFilter(false, false)  // Include all pages (don't filter hidden)
        );

        while (children.hasNext()) {
            Page child = children.next();
            // Only include pages not hidden in navigation
            if (!child.getProperties().get("hideInNav", false)) {
                navigationPages.add(child);
            }
        }
    }

    public List<Page> getNavigationPages() { return navigationPages; }
}
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. How do you get a `PageManager` in a Sling Model?**

> **Answer:** Two ways: 1) `resourceResolver.adaptTo(PageManager.class)` when you have a `ResourceResolver` injected. 2) `@ScriptVariable private PageManager pageManager` for injection from AEM's scripting context (available in models adapted from `SlingHttpServletRequest`). The `@ScriptVariable` approach is simpler for UI-rendering models; the `adaptTo()` approach works in background services.

**Q2. What is the difference between `page.getTitle()` and `page.getNavigationTitle()`?**

> **Answer:** `getTitle()` returns `jcr:content/jcr:title` — the main content title (shown in page content, metadata). `getNavigationTitle()` returns `jcr:content/navTitle` if set, otherwise falls back to `jcr:title`. Use `getNavigationTitle()` for nav menus — editors often want a shorter label in navigation than in the content. Similarly, `getPageTitle()` returns `jcr:content/pageTitle` for the browser `<title>` tag.

**Q3. What is `WCMMode` and why is it important for performance?**

> **Answer:** `WCMMode` indicates the current rendering context: EDIT (Author editing), PREVIEW (Author preview), DESIGN (legacy), or DISABLED (Publish/anonymous). It's critical for performance because components shouldn't make expensive external API calls in Author mode — authors rarely need real data (and it burns API quotas). Check `WCMMode.DISABLED` in `@PostConstruct` and skip expensive operations when not on Publish. Also used to show author-specific UI (placeholders, edit hints) only in Edit mode.

**Q4. How do you create a page programmatically?**

> **Answer:** Use `pageManager.create(parentPath, pageName, templatePath, title)`. This requires the service user to have `jcr:addChildNodes` on the parent path and the template must exist at the specified path. After creation, call `resourceResolver.commit()` to persist. The `create()` method returns the new `Page` object. Optionally, adapt to `ModifiableValueMap` on the page's `jcr:content` resource to set additional properties.

---

## ✅ Best Practices

1. **Use `PageFilter` when iterating children** — filter hidden/invalid pages unless you specifically need them
2. **Use `page.getNavigationTitle()`** for nav menus — not `page.getTitle()`
3. **Check `WCMMode.DISABLED`** before making API calls — skip in Author mode
4. **Always null-check `pageManager.getPage()`** — returns null for non-existent paths
5. **Use `TagManager.setTags()`** for applying tags — handles tag merging correctly
6. **Store tag IDs, not titles** — tag titles can change; IDs are stable references

---

## 🛠️ Hands-on Practice

1. Write a Sling Model that reads the current page's title, description, tags, and parent page title
2. Write a `BreadcrumbModel` that builds a breadcrumb from the current page to the site root (depth=2)
3. Write a `NavigationModel` that reads first-level child pages from `/content/mysite/en` — filter hidden pages
4. Add `WCMMode` check to an existing model — in Author mode, return mock/empty data; in Publish mode, return real data
5. Write a Groovy script that uses `pageManager.create()` to create 5 article pages under `/content/mysite/en/articles/`
