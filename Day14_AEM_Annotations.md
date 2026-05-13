# AEM Annotations — Complete Reference Guide
**Target:** 2–4 YOE | Sling, OSGi, Jackson, CDI & AEM-specific

---

## 🌟 Why Annotations Matter in AEM

AEM development is **annotation-driven**. Almost everything — from registering an OSGi service, to injecting a JCR resource, to exporting JSON — is configured through annotations. Understanding what each annotation does (and why) is critical to writing correct, maintainable AEM code.

This guide covers all major annotation groups:
1. **OSGi (SCR Annotations)** — `@Component`, `@Reference`, `@Activate`, etc.
2. **Sling Models** — `@Model`, `@ValueMapValue`, `@ChildResource`, etc.
3. **Sling Servlet** — `@SlingServletResourceTypes`, `@SlingServletPaths`
4. **Jackson (JSON)** — `@JsonProperty`, `@JsonIgnore`, etc.
5. **AEM-specific** — `@Exporter`, `@WorkflowProcess`

---

## 1️⃣ OSGi / Declarative Services (DS) Annotations

Package: `org.osgi.service.component.annotations.*`

---

### `@Component`

**What it does:** Registers a Java class as an OSGi component (the fundamental unit in OSGi). Without this, the class is just a POJO — OSGi doesn't know it exists.

```java
@Component(
    service = MyService.class,   // What interface this component provides as a service
    immediate = true,            // Start immediately on bundle activation (default: lazy)
    property = {
        "service.description=My Site Content Service",
        "service.vendor=My Site",
        "process.label=My Custom Workflow Step"  // For WorkflowProcess — name shown in UI
    },
    // servicefactory = true    // Create separate instance per bundle requesting it (rarely used)
)
public class MyServiceImpl implements MyService {
    // ...
}
```

| Property | Use When |
|----------|---------|
| `service = X.class` | Implementing an interface you want others to inject |
| `immediate = true` | Must start even without any consumers (Schedulers, event listeners) |
| `property` | Setting metadata (service description, workflow labels, servlet paths) |

---

### `@Designate`

**What it does:** Links a component to its OSGi configuration interface (`@ObjectClassDefinition`). This enables the component to be configured via `/system/console/configMgr` or OSGi config JSON files.

```java
@Component(service = ApiService.class)
@Designate(ocd = ApiServiceImpl.Config.class)  // Links to the Config interface below
public class ApiServiceImpl implements ApiService {

    // Nested config interface — OCD = Object Class Definition
    @ObjectClassDefinition(
        name = "My Site - API Service Configuration",
        description = "Configure the external API connection"
    )
    public @interface Config {

        @AttributeDefinition(
            name = "API Base URL",
            description = "Base URL for the external API"
        )
        String apiBaseUrl() default "https://api.mysite.com";

        @AttributeDefinition(name = "Connection Timeout (ms)")
        int connectionTimeout() default 5000;

        @AttributeDefinition(name = "Enabled")
        boolean enabled() default true;

        @AttributeDefinition(
            name = "Allowed Paths",
            description = "JCR paths this service may access"
        )
        String[] allowedPaths() default {"/content/mysite", "/conf/mysite"};
    }

    @Activate
    protected void activate(Config config) {
        String url     = config.apiBaseUrl();
        int timeout    = config.connectionTimeout();
        boolean active = config.enabled();
    }
}
```

---

### `@Activate`

**What it does:** Marks a method to be called when the OSGi component is first activated (started). Use it to initialize resources, start connections, schedule jobs, and read config values.

```java
@Activate
protected void activate(Config config) {
    // Called ONCE when component starts
    // Receives the merged OSGi config (default + OSGi config file values)
    this.apiUrl = config.apiBaseUrl();
    this.httpClient = HttpClients.createDefault();  // Initialize resources here
    LOG.info("ApiService activated with URL: {}", this.apiUrl);
}

// Alternate signature — receives component properties map
@Activate
protected void activate(Map<String, Object> properties) {
    this.apiUrl = PropertiesUtil.toString(properties.get("apiBaseUrl"), "default-url");
}
```

---

### `@Modified`

**What it does:** Called when the OSGi configuration changes at runtime (via `/system/console/configMgr` or after a new OSGi config file is deployed). The component does NOT restart — `@Modified` fires instead.

```java
@Modified
protected void modified(Config config) {
    // Best practice: just delegate to activate() — same initialization logic
    LOG.info("Config changed — re-initializing ApiService");
    activate(config);
}
```

---

### `@Deactivate`

**What it does:** Called when the OSGi component is stopped (bundle deactivated, server shutdown, config PID deleted). **CRITICAL:** Release ALL resources here to prevent memory leaks.

```java
@Deactivate
protected void deactivate() {
    // ALWAYS clean up in deactivate:
    // 1. Close HTTP clients
    if (httpClient != null) {
        try { httpClient.close(); } catch (IOException e) { /* ignore */ }
    }
    // 2. Unschedule jobs
    scheduler.unschedule("MySchedulerJob");
    // 3. Unregister listeners
    if (eventHandlerRegistration != null) {
        eventHandlerRegistration.unregister();
    }
    // 4. Clear caches
    cache.invalidateAll();
    LOG.info("ApiService deactivated — resources released");
}
```

---

### `@Reference`

**What it does:** Injects another OSGi service into this component. OSGi handles finding and providing the service — you don't instantiate it manually.

```java
// Basic injection — required by default (component won't activate if service missing)
@Reference
private ResourceResolverFactory resolverFactory;

// Optional reference — component activates even if this service is absent
@Reference(cardinality = ReferenceCardinality.OPTIONAL)
private CacheService cacheService;

// Multiple references — collects ALL registered implementations
@Reference(cardinality = ReferenceCardinality.MULTIPLE,
           policy = ReferencePolicy.DYNAMIC)
private volatile List<ContentProcessor> processors;  // volatile for DYNAMIC

// Filtered reference — only inject services matching a filter
@Reference(target = "(site.name=mysite)")
private SiteConfig siteConfig;

// Named reference (when multiple beans of same type)
@Reference(name = "primaryEmailService")
private EmailService emailService;
```

| Cardinality | Meaning |
|-------------|---------|
| `MANDATORY` (default) | Exactly one — component fails to start if missing |
| `OPTIONAL` | Zero or one — component starts even if absent; inject may be null |
| `MULTIPLE` | Zero or more — List of all registered implementations |
| `AT_LEAST_ONE` | One or more — List, but fails if none found |

---

## 2️⃣ Sling Models Annotations

Package: `org.apache.sling.models.annotations.*` and `org.apache.sling.models.annotations.injectorspecific.*`

---

### `@Model`

**What it does:** Declares a class as a Sling Model — a data-binding POJO tied to an AEM resource or request.

```java
@Model(
    // What type can be adapted to this model
    adaptables = {Resource.class, SlingHttpServletRequest.class},

    // What interfaces this model provides (for injection into other models)
    adapters = {HeroModel.class, ComponentExporter.class},

    // Which component resourceType this model is associated with
    resourceType = "mysite/components/content/hero",

    // Injection strategy: OPTIONAL = null if not found (REQUIRED throws if missing)
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class HeroModel { ... }
```

**When to use `Resource.class` vs `SlingHttpServletRequest.class`:**

| Adaptable | Use When | HTL Usage |
|-----------|---------|-----------|
| `Resource.class` | No request context needed (background jobs, nested models) | `data-sly-use.model="..."` adapts from Resource |
| `SlingHttpServletRequest.class` | Need request params, cookies, session | `data-sly-use.model="..."` adapts from Request |
| Both | Maximum compatibility — works in all contexts | Recommended for most UI components |

---

### `@ValueMapValue`

**What it does:** Injects a value from the current resource's JCR properties (ValueMap). This is the standard way to read component dialog field values.

```java
// Field name = JCR property name (case-sensitive)
@ValueMapValue
private String title;             // Reads JCR property "title"

// When JCR property name differs from field name
@ValueMapValue(name = "jcr:title")
private String pageTitle;         // Reads "jcr:title" property

// With default value (if property is absent)
@ValueMapValue(injectionStrategy = InjectionStrategy.OPTIONAL)
private String variant;           // null if not found

// Type coercion — Sling handles conversion
@ValueMapValue
private int columnCount;          // Reads String "3" from JCR, converts to int 3

@ValueMapValue
private boolean showDate;         // Reads "true"/"false" string, converts to boolean

@ValueMapValue
private String[] tags;            // Reads multi-value property as String[]

@ValueMapValue
private Calendar publishDate;     // Reads JCR Date property

@ValueMapValue(name = "fileReference")
private String imageReference;    // DAM asset path from image widget
```

---

### `@ChildResource`

**What it does:** Injects a child node of the current resource. Used for composite multifields (each row is a child node) and nested resource structures.

```java
// Inject a specific named child node
@ChildResource
private Resource header;          // Injects /current-resource/header node

// Inject when field name differs from child node name
@ChildResource(name = "jcr:content")
private Resource jcrContent;

// Inject a list of children (for multifields)
@ChildResource
private List<Resource> tabs;      // Injects all children of "tabs" node

// Inject into a Sling Model directly
@ChildResource
private TabModel featuredTab;     // Adapts "featuredTab" child to TabModel
```

---

### `@SlingObject`

**What it does:** Injects Sling-specific objects that are always available in the request/resource context.

```java
@SlingObject
private Resource resource;                    // The current resource being adapted

@SlingObject
private ResourceResolver resourceResolver;    // The resource resolver (for navigation)

@SlingObject
private SlingHttpServletRequest request;      // Current HTTP request

@SlingObject
private SlingHttpServletResponse response;    // Current HTTP response
```

---

### `@ScriptVariable`

**What it does:** Injects Sling scripting variables (from the scripting bindings). These are objects made available by AEM's rendering engine.

```java
@ScriptVariable
private Page currentPage;         // The current AEM page being rendered

@ScriptVariable
private Page resourcePage;        // The page containing the current resource

@ScriptVariable
private ValueMap pageProperties;  // jcr:content properties of currentPage

@ScriptVariable
private WCMMode wcmMode;          // Current authoring mode (EDIT, PREVIEW, DISABLED)

@ScriptVariable
private Designer designer;        // Legacy: current design (use Content Policies instead)

@ScriptVariable
private ComponentContext componentContext;  // Component rendering context
```

---

### `@OSGiService`

**What it does:** Injects an OSGi service into a Sling Model. Unlike `@Reference` (used in `@Component` classes), `@OSGiService` is for Sling Models.

```java
@OSGiService
private ResourceResolverFactory resolverFactory;

@OSGiService
private QueryBuilder queryBuilder;

@OSGiService
private ContentPolicyManager contentPolicyManager;

@OSGiService
private LanguageManager languageManager;

// With filter for specific implementation
@OSGiService(filter = "(service.type=primary)")
private EmailService emailService;
```

---

### `@Inject`

**What it does:** Generic injection — Sling tries all available injectors (ValueMap, child resource, scripting variable, OSGi service). **Use specific annotations instead** for clarity and performance.

```java
// ❌ Avoid — ambiguous, slower (tries multiple injectors)
@Inject
private String title;

// ✅ Prefer — explicit, fast, clear intent
@ValueMapValue
private String title;
```

---

### `@Self`

**What it does:** Injects the adaptable object itself (the Resource or Request being adapted). Used with `@Via` to adapt to a parent type.

```java
// Inject the adaptable (Resource) itself
@Self
private Resource self;

// Inject a Core Component's Sling Model to delegate behavior
// @Via tells Sling to follow sling:resourceSuperType to find the parent model
@Self
@Via(type = ResourceSuperType.class)
private com.adobe.cq.wcm.core.components.models.Image coreImageModel;

// Use coreImageModel.getSrc(), getSrcUriTemplate(), etc.
// and add your own custom fields on top
```

---

### `@Via`

**What it does:** Specifies a different source for the injection (e.g., a different resource or model path).

```java
// Adapt from the request's underlying resource (when adaptable is Request)
@Self
@Via("resource")
private Resource resourceFromRequest;

// Delegate to Core Component's model (most common use case)
@Self
@Via(type = ResourceSuperType.class)
private com.adobe.cq.wcm.core.components.models.Teaser coreTeaserModel;
```

---

### `@PostConstruct`

**What it does:** Marks a method to be called AFTER all injections are complete. Use for: computing derived values, calling services, null-checking injected fields, building complex data structures.

```java
@PostConstruct
protected void init() {
    // Called once, after ALL @ValueMapValue/@ChildResource/etc. injections are done

    // 1. Null-safe: injections are done, so fields are populated (or null if OPTIONAL)
    if (StringUtils.isBlank(title)) {
        title = currentPage.getTitle();  // Fallback to page title
    }

    // 2. Call services
    resolvedLink = resourceResolver.map(rawLink) + ".html";

    // 3. Build complex structures from raw injected data
    tabList = buildTabsFromResources(tabs);
}
```

---

### `@Default`

**What it does:** Provides a default value if the injection returns null (used with `@ValueMapValue`).

```java
@ValueMapValue
@Default(values = "h2")       // Default for String
private String headingType;

@ValueMapValue
@Default(intValues = 3)       // Default for int
private int columns;

@ValueMapValue
@Default(booleanValues = true) // Default for boolean
private boolean showImage;

@ValueMapValue
@Default(values = {"tag1", "tag2"}) // Default for String[]
private String[] defaultTags;
```

---

## 3️⃣ Sling Servlet Annotations

---

### `@SlingServletResourceTypes`

**What it does:** Registers a servlet to handle requests for specific component resource types. **Recommended over path-based** — inherits content node's ACLs.

```java
@Component(service = Servlet.class)
@SlingServletResourceTypes(
    resourceTypes = "mysite/components/content/search",  // Component resourceType
    methods       = {"GET", "POST"},                      // HTTP methods
    selectors     = {"results", "suggest"},               // .results. .suggest. selectors
    extensions    = {"json"}                              // .json extension
    // URL: /content/mysite/en/search.results.json
    //      /content/mysite/en/search.suggest.json
)
public class SearchServlet extends SlingSafeMethodsServlet { ... }
```

---

### `@SlingServletPaths` (Use Sparingly)

**What it does:** Registers a servlet at a fixed URL path. Use only for paths that don't have JCR backing (health checks, webhooks).

```java
@Component(service = Servlet.class)
@SlingServletPaths(
    value = {"/bin/mysite/health", "/bin/mysite/webhook"}
)
public class HealthCheckServlet extends SlingAllMethodsServlet { ... }
// URL: GET /bin/mysite/health
//      POST /bin/mysite/webhook
// ⚠️ MUST add these paths to Dispatcher filter allow rules!
```

---

## 4️⃣ Jackson (JSON Serialization) Annotations

Used with Sling Model Exporter for JSON output.

---

### `@Exporter`

**What it does:** Registers a Sling Model as a JSON (or XML) exporter, making it accessible at `<resource>.model.json`.

```java
@Model(adaptables = Resource.class, adapters = {HeroModel.class, ComponentExporter.class})
@Exporter(
    name       = ExporterConstants.SLING_MODEL_EXPORTER_NAME,  // "jackson"
    extensions = ExporterConstants.SLING_MODEL_EXTENSION        // "model"
)
public class HeroModel implements ComponentExporter { ... }
// Access at: /content/mysite/en/home/jcr:content/root/hero.model.json
```

---

### `@JsonProperty`

**What it does:** Customizes the JSON key name for a getter method.

```java
@JsonProperty("pageTitle")     // JSON key = "pageTitle", not "title"
public String getTitle() { return title; }

@JsonProperty("isActive")
public boolean isEnabled() { return enabled; }  // Without: key would be "enabled"
```

---

### `@JsonIgnore`

**What it does:** Excludes a getter from JSON output.

```java
@JsonIgnore
public String getInternalId() { return internalId; }  // Won't appear in .model.json

@JsonIgnore
private String adminPassword;  // Never expose sensitive fields
```

---

### `@JsonInclude`

**What it does:** Controls when a field is included in JSON output.

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
public String getOptionalField() { return optionalField; }
// Only included in JSON if not null — keeps output clean

@JsonInclude(JsonInclude.Include.NON_EMPTY)
public List<String> getTags() { return tags; }
// Only included if list is not null AND not empty
```

---

## 5️⃣ OSGi Event Handling

---

### `@EventHandler` (Deprecated — use Whiteboard Pattern)

```java
// Modern way: register as EventHandler service with properties
@Component(
    service = EventHandler.class,
    property = {
        EventConstants.EVENT_TOPIC + "=" + PageEvent.EVENT_TOPIC,
        EventConstants.EVENT_FILTER + "=(type=PageActivated)"
    }
)
public class PageActivationHandler implements EventHandler {
    @Override
    public void handleEvent(Event event) {
        String path = (String) event.getProperty("path");
        LOG.info("Page activated: {}", path);
    }
}
```

---

## 6️⃣ AEM-Specific Annotations

---

### `@WorkflowProcess` (via `process.label` property)

```java
@Component(
    service = WorkflowProcess.class,
    property = {
        "process.label=My Site - Custom Workflow Step"   // Label in Workflow Editor UI
    }
)
public class MyWorkflowProcess implements WorkflowProcess {
    @Override
    public void execute(WorkItem item, WorkflowSession session, MetaDataMap args) {
        // Workflow step logic
    }
}
```

---

### Sling Model with All Common Annotations — Complete Example

```java
package com.mysite.core.models;

import com.adobe.cq.export.json.ComponentExporter;
import com.adobe.cq.export.json.ExporterConstants;
import com.adobe.cq.wcm.core.components.models.Image;
import com.day.cq.wcm.api.Page;
import com.day.cq.wcm.api.policies.ContentPolicyManager;
import com.fasterxml.jackson.annotation.JsonIgnore;
import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.annotation.JsonProperty;
import org.apache.sling.api.SlingHttpServletRequest;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.api.resource.ResourceResolver;
import org.apache.sling.models.annotations.*;
import org.apache.sling.models.annotations.injectorspecific.*;

import javax.annotation.PostConstruct;
import java.util.List;

@Model(
    adaptables = {Resource.class, SlingHttpServletRequest.class},
    adapters = {HeroModel.class, ComponentExporter.class},
    resourceType = "mysite/components/content/hero",
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
@Exporter(
    name = ExporterConstants.SLING_MODEL_EXPORTER_NAME,
    extensions = ExporterConstants.SLING_MODEL_EXTENSION
)
public class HeroModel implements ComponentExporter {

    // ── Sling Object Injectors ──
    @SlingObject
    private Resource resource;

    @SlingObject
    private ResourceResolver resourceResolver;

    // ── Script Variables (from AEM rendering context) ──
    @ScriptVariable
    private Page currentPage;

    // ── Dialog Values (from JCR node properties) ──
    @ValueMapValue
    private String title;

    @ValueMapValue(name = "jcr:description")
    private String description;

    @ValueMapValue
    @Default(values = "h2")
    private String headingLevel;

    @ValueMapValue
    private String ctaLink;

    @ValueMapValue
    private String ctaLabel;

    @ValueMapValue
    private boolean openInNewTab;

    // ── Child Resources (from child nodes — multifield data) ──
    @ChildResource
    private List<Resource> features;

    // ── Core Component delegation ──
    @Self
    @Via(type = ResourceSuperType.class)
    private Image coreImageModel;

    // ── OSGi Service ──
    @OSGiService
    private ContentPolicyManager contentPolicyManager;

    // ── Computed Fields ──
    private String resolvedCtaLink;
    private boolean hasContent;

    @PostConstruct
    protected void init() {
        // Fallback title to page title
        if (title == null && currentPage != null) {
            title = currentPage.getTitle();
        }

        // Resolve CTA link
        if (ctaLink != null) {
            if (ctaLink.startsWith("/content/")) {
                resolvedCtaLink = resourceResolver.map(ctaLink) + ".html";
            } else {
                resolvedCtaLink = ctaLink;
            }
        }

        hasContent = title != null;
    }

    // ── Getters (exposed as JSON and to HTL) ──

    @JsonProperty("title")
    public String getTitle() { return title; }

    @JsonProperty("description")
    @JsonInclude(JsonInclude.Include.NON_NULL)
    public String getDescription() { return description; }

    @JsonProperty("headingLevel")
    public String getHeadingLevel() { return headingLevel; }

    @JsonProperty("ctaLink")
    @JsonInclude(JsonInclude.Include.NON_NULL)
    public String getCtaLink() { return resolvedCtaLink; }

    @JsonProperty("ctaLabel")
    public String getCtaLabel() { return ctaLabel; }

    @JsonProperty("openInNewTab")
    public boolean isOpenInNewTab() { return openInNewTab; }

    @JsonIgnore  // Internal — don't expose in JSON
    public boolean isHasContent() { return hasContent; }

    @Override
    public String getExportedType() {
        return "mysite/components/content/hero";
    }
}
```

---

## 📋 Quick Reference Card

| Annotation | Package | Purpose |
|-----------|---------|---------|
| `@Component` | `org.osgi.service.component.annotations` | Register OSGi component |
| `@Designate` | `org.osgi.service.component.annotations` | Link component to config |
| `@ObjectClassDefinition` | `org.osgi.service.metatype.annotations` | Define OSGi config schema |
| `@AttributeDefinition` | `org.osgi.service.metatype.annotations` | Define one config field |
| `@Activate` | `org.osgi.service.component.annotations` | Called on component start |
| `@Modified` | `org.osgi.service.component.annotations` | Called on config change |
| `@Deactivate` | `org.osgi.service.component.annotations` | Called on component stop |
| `@Reference` | `org.osgi.service.component.annotations` | Inject OSGi service |
| `@Model` | `org.apache.sling.models.annotations` | Declare Sling Model |
| `@Exporter` | `org.apache.sling.models.annotations` | Enable JSON export |
| `@ValueMapValue` | `...injectorspecific` | Read JCR property |
| `@ChildResource` | `...injectorspecific` | Inject child JCR node |
| `@SlingObject` | `...injectorspecific` | Inject Sling objects |
| `@ScriptVariable` | `...injectorspecific` | Inject AEM render vars |
| `@OSGiService` | `...injectorspecific` | Inject OSGi service in Model |
| `@Inject` | `javax.inject` | Generic injection (avoid) |
| `@Self` | `...injectorspecific` | Inject adaptable itself |
| `@Via` | `org.apache.sling.models.annotations` | Change injection source |
| `@PostConstruct` | `javax.annotation` | Post-injection init method |
| `@Default` | `org.apache.sling.models.annotations` | Default injection value |
| `@JsonProperty` | `com.fasterxml.jackson.annotation` | Rename JSON field |
| `@JsonIgnore` | `com.fasterxml.jackson.annotation` | Exclude from JSON |
| `@JsonInclude` | `com.fasterxml.jackson.annotation` | Conditional JSON inclusion |
| `@SlingServletResourceTypes` | `org.apache.sling.servlets.annotations` | Register resource-type servlet |
| `@SlingServletPaths` | `org.apache.sling.servlets.annotations` | Register path servlet |

---

## ❓ Interview Questions

**Q1. What is the difference between `@Reference` and `@OSGiService`?**
> Both inject OSGi services, but `@Reference` is for `@Component`-annotated OSGi components, while `@OSGiService` is for `@Model`-annotated Sling Models. Functionally similar — just different annotation packages for different class types.

**Q2. What happens if `@Activate` throws an exception?**
> The OSGi component fails to activate. Its state becomes "failed". Other components that `@Reference` it will also fail to activate (cascade failure). Always handle exceptions in `@Activate` and log clearly.

**Q3. Why prefer `@ValueMapValue` over `@Inject` for JCR properties?**
> `@ValueMapValue` is explicit — it tells the reader exactly WHERE the value comes from (the JCR ValueMap). `@Inject` is ambiguous — Sling tries multiple injectors (ValueMap, child resource, OSGi service) in order, which is slower and harder to debug. Use specific annotations always.

**Q4. When is `@PostConstruct` called relative to injections?**
> After ALL injections (`@ValueMapValue`, `@ChildResource`, `@SlingObject`, etc.) have been completed. It's the earliest point where you're guaranteed all declared fields are populated (or null if OPTIONAL). Use it for initialization that depends on injected values.

**Q5. What does `ReferenceCardinality.MULTIPLE` with `ReferencePolicy.DYNAMIC` enable?**
> It creates a dynamic list of all registered implementations of a service interface. As new implementations are deployed (or removed), the list updates at runtime without restarting the consuming component. Requires `volatile` on the list field. Useful for plugin patterns where you want to collect all registered processors/handlers.
