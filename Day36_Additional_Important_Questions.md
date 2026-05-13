# Additional Important Questions — Senior-Level Deep Dives
**Target:** 2–4 YOE | AEM 6.5 & Cloud Service

---

## Q1. Static Templates vs Editable Templates (Complete Comparison)

### What Are Static Templates? (Legacy — AEM 5.x to 6.1)

Static templates were defined entirely by developers in `/apps/<site>/templates/`. Authors had NO control over template structure. Component restrictions required "design mode" (a separate authoring mode).

```
/apps/mysite/templates/
  └── homepage/
        ├── .content.xml    ← Template definition
        └── thumbnail.png
```

```xml
<!-- Static Template .content.xml -->
<jcr:root
    jcr:primaryType="cq:Template"
    jcr:title="Homepage"
    ranking="{Long}100"
    allowedPaths="[/content/mysite(/.*)?,/content/mysite]"
    allowedParents="[/apps/mysite/templates/.*]"
    allowedChildren="[/apps/mysite/templates/.*]">
  <jcr:content
      jcr:primaryType="cq:PageContent"
      sling:resourceType="mysite/components/page/homepage">
    <!-- Fixed structure — authors CANNOT change this -->
    <header sling:resourceType="mysite/components/structure/header"/>
    <main sling:resourceType="foundation/components/parsys"/>
    <footer sling:resourceType="mysite/components/structure/footer"/>
  </jcr:content>
</jcr:root>
```

### What Are Editable Templates? (Modern — AEM 6.2+)

Editable Templates are managed in `/conf/<site>/settings/wcm/templates/`. Template authors (NOT developers) can create and modify templates in the AEM UI.

```
/conf/mysite/settings/wcm/templates/
  └── homepage/
        ├── jcr:content/         ← Template metadata
        ├── structure/           ← LOCKED content (header, footer)
        ├── initial/             ← Default content for new pages
        └── policies/            ← Policy assignments
```

### Side-by-Side Comparison

| Aspect | Static Templates | Editable Templates |
|--------|-----------------|-------------------|
| **Location** | `/apps/mysite/templates/` | `/conf/mysite/settings/wcm/templates/` |
| **Who creates** | Developers only | Template authors (AEM UI) |
| **Component control** | Design mode (Classic UI) | Content Policies |
| **Locked structure** | ❌ Not possible | ✅ Structure layer |
| **Initial content** | ❌ Not possible | ✅ Initial layer |
| **Style System** | ❌ Not supported | ✅ Supported |
| **AEM Cloud** | ⚠️ Partially supported | ✅ Required |
| **Best practice** | ❌ Legacy only | ✅ Always use |

### Three Layers of Editable Templates

```
┌─────────────────────────────────────────────┐
│                  STRUCTURE                   │  ← Developer/Template Author
│  [Header - LOCKED]   [Footer - LOCKED]       │     Authors CANNOT edit this
└─────────────────────────────────────────────┘
┌─────────────────────────────────────────────┐
│               INITIAL CONTENT                │  ← Copied to page at creation
│  [Default Hero Text]   [Default CTA]         │     Authors CAN edit afterward
└─────────────────────────────────────────────┘
┌─────────────────────────────────────────────┐
│                  POLICIES                    │  ← Template Author
│  Main Zone: [Text, Image, Video allowed]     │     Defines allowed components
│  Sidebar: [Text, Form allowed]               │
└─────────────────────────────────────────────┘
```

---

## Q2. How Dispatcher Caching Works — Deep Dive

### What Dispatcher Caches

The Dispatcher caches **static HTML files** on the server's filesystem. When a request comes in:

```
Request: GET /content/mysite/en/home.html

Dispatcher checks: /var/cache/mysite/content/mysite/en/home.html
                                    ↑
                                    (mirrors the URL path)

→ File EXISTS and is valid → Serve directly (no AEM involved!)
→ File MISSING or invalidated → Forward to AEM Publish
```

### File-Based Cache Structure
```
/var/cache/                                 ← Dispatcher document root
  └── content/
        └── mysite/
              └── en/
                    ├── home.html           ← Cached page
                    ├── home.html.stat      ← Timestamp file
                    ├── about.html
                    └── products/
                          └── laptop.html
```

### Cache Invalidation Flow (Critical to Know!)

```
1. Editor clicks "Publish" on Author
        ↓
2. Author's Replication Agent sends content to Publish
        ↓
3. Publish instance receives content AND sends cache invalidation
   signal to Dispatcher (via /dispatcher/invalidate.cache endpoint)
        ↓
4. Dispatcher finds all files in cache that match the invalidation path
        ↓
5. Dispatcher deletes .stat files (marks cache as stale)
        ↓
6. Next request for that page → Cache MISS → AEM re-renders → Cache updated
```

### Dispatcher Configuration Key Sections

```apache
# /etc/httpd/conf.dispatcher.d/available_farms/default.farm

/renders {
    # AEM Publish instance(s)
    /renderer0 {
        /hostname "aem-publish.internal"
        /port "4503"
        /timeout "60000"
    }
}

/cache {
    /docroot "/var/www/html"

    /rules {
        # Cache all .html pages
        /0001 { /type "allow" /url "*.html" }
        # Don't cache these
        /0002 { /type "deny"  /url "/content/dam/*.html" }
        /0003 { /type "deny"  /url "*.nocache.html" }
    }

    /invalidate {
        # Who can invalidate the cache?
        /0000 { /type "allow" /glob "*" }
    }

    /allowedClients {
        # Which IPs can send invalidation requests?
        /0001 { /type "allow" /glob "10.0.0.*" }    # AEM Publish IP range
    }
}

/filter {
    # Security: Block dangerous paths
    /0001 { /type "deny"  /url "/system/*" }
    /0002 { /type "deny"  /url "/crx/*" }
    /0003 { /type "deny"  /url "/apps/*" }
    /0004 { /type "deny"  /url "/libs/*" }
    /0005 { /type "allow" /url "/libs/granite/csrf/token.json" }
    # Allow public content
    /0100 { /type "allow" /method "GET" /url "/content/mysite/*" }
    /0101 { /type "allow" /url "/etc.clientlibs/*" }
}
```

### What Dispatcher Does NOT Cache (By Default)

| Not Cached | Reason |
|-----------|--------|
| POST requests | Not idempotent — each POST may have different effects |
| Requests with query strings | e.g., `/page.html?campaign=summer` |
| Authenticated requests | User-specific content shouldn't be shared |
| Requests with cookies | Dynamic/personalized content |
| Error responses (4xx, 5xx) | Don't cache failures |

### Dispatcher Flush (Cache Clear)

```
Tools → Deployment → Replication → Agents on Author
→ "Dispatcher Flush" agent
→ Click "Test Connection" → should return 200
→ Activate any page → watch flush log
```

---

## Q3. AEM Run Modes — Complete Guide

### What Are Run Modes?

Run modes are **labels** applied to an AEM instance that activate specific OSGi configurations. They answer the question: "Which configs should be active for THIS instance?"

### Setting Run Modes

**Method 1: JVM System Property (most common)**
```bash
java -jar aem-quickstart.jar -Dsling.run.modes=publish,prod,us-east
```

**Method 2: Quickstart JAR naming (for development)**
```bash
# File named: aem-65-publish.jar
# AEM reads "publish" as a run mode from the filename
```

**Method 3: sling.properties file**
```properties
# crx-quickstart/conf/sling.properties
sling.run.modes=author,dev
```

**Method 4: AEM Cloud (set in Cloud Manager environment)**

### Run Mode Config Resolution

```
Config locations checked (most specific to least specific):
config.author.prod.us-east    ← Most specific (author + prod + us-east)
config.author.prod             ← Author + production
config.author                  ← Author only
config.prod                    ← Production only
config                         ← All instances (least specific)
```

**Example:** Instance with run modes `author,prod,us-east` will use config from `config.author.prod.us-east/` if it exists, otherwise falls back down the chain.

### Checking Run Modes in Code

```java
@Reference
private SlingSettingsService slingSettings;

public void checkEnvironment() {
    Set<String> runModes = slingSettings.getRunModes();

    boolean isAuthor = runModes.contains("author");
    boolean isPublish = runModes.contains("publish");
    boolean isProd = runModes.contains("prod");

    LOG.info("Instance type: {}", isAuthor ? "AUTHOR" : "PUBLISH");

    if (isProd) {
        // Production-specific behavior
    }
}
```

### Checking in HTL
```html
<sly data-sly-use.slingSettings="org.apache.sling.settings.SlingSettingsService"/>
<!-- Show only on Author -->
<div data-sly-test="${slingSettings.runModes contains 'author'}">
    Author-only content
</div>
```

---

## Q4. Dispatcher Caching — Request Path Patterns & Gotchas

### URL Patterns That Break Caching

```
# These patterns PREVENT Dispatcher caching:
/content/page.html?param=value    ← Query string = not cached by default
/content/page.nocache.html        ← .nocache selector convention
/bin/mysite/api                   ← /bin/ paths (no extension)
```

### Enabling Query String Caching (with Caution)

```apache
# dispatcher.any — Allow specific query params to be cached
/cache {
    /ignoreUrlParams {
        /0001 { /type "allow" /glob "utm_*" }   # Ignore UTM params
        /0002 { /type "allow" /glob "debug" }    # Ignore debug param
        # All other params: NOT ignored → cache miss for each unique combo
    }
}
```

---

## Q5. How to Debug a Sling Model — Complete Checklist

```
Symptom: Component not rendering / data is null

STEP 1: Check if bundle is active
→ /system/console/bundles
→ Search your bundle name
→ Status must be "Active"
→ If "Installed" or "Resolved": click bundle → check Dependencies tab
→ If "Installed": missing dependent bundle
→ If "Resolved": component won't activate — check /system/console/components

STEP 2: Check model registration
→ /system/console/status-adapters.txt
→ Ctrl+F your model class name
→ If not found: @Model annotation missing, wrong package, or bundle not active

STEP 3: Check OSGi components
→ /system/console/components
→ Search your model class
→ Status must be "active"

STEP 4: Enable debug logging
→ /system/console/slinglog
→ Add logger: com.mysite.core.models = DEBUG (or TRACE)
→ Tail log: tail -f crx-quickstart/logs/error.log

STEP 5: Check @Model adaptable
→ If using data-sly-use in HTL, HTL adapts from REQUEST by default
→ If @Model adaptables = Resource.class, but component is inside a request context...
→ Solution: Use adaptables = {SlingHttpServletRequest.class, Resource.class}

STEP 6: Check @ValueMapValue field names
→ JCR property name must EXACTLY match field name (or use name="jcr:propertyName")
→ e.g., dialog field name="./jcr:title" → Java field @ValueMapValue(name="jcr:title")

STEP 7: Verify content in CRXDE
→ Navigate to the component's JCR node
→ Verify the dialog values are saved with the expected property names
→ Compare with @ValueMapValue field names in Java

STEP 8: Add debug output to HTL
<sly data-sly-use.model="com.mysite.core.models.MyModel"/>
Model is: ${model}
Title: ${model.title}
HasContent: ${model.hasContent}
```

---

## Q6. What is ResourceResolverFactory? Complete Patterns

```java
/**
 * ResourceResolverFactory is the central OSGi service for getting
 * a ResourceResolver — the gateway to JCR content.
 */

// Pattern 1: Service User (for background services, schedulers)
@Reference
private ResourceResolverFactory resolverFactory;

public void backgroundWork() {
    Map<String, Object> params = Collections.singletonMap(
        ResourceResolverFactory.SUBSERVICE, "my-subservice-name"
    );
    try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(params)) {
        // READ
        Resource page = resolver.getResource("/content/mysite/en/home");
        if (page != null) {
            String title = page.getChild("jcr:content")
                              .getValueMap().get("jcr:title", String.class);
        }

        // WRITE — use ModifiableValueMap
        Resource target = resolver.getResource("/content/mysite/data/submissions");
        Resource newNode = resolver.create(target, "entry-001",
            Map.of("jcr:primaryType", "nt:unstructured",
                   "name", "John",
                   "submittedAt", Calendar.getInstance()));

        // COMMIT changes (nothing is saved until you commit!)
        resolver.commit();

    } catch (LoginException e) {
        LOG.error("Service user '{}' failed to log in. Check ServiceUserMapper config.",
            "my-subservice-name", e);
    } catch (PersistenceException e) {
        LOG.error("JCR write failed", e);
    }
}

// Pattern 2: In a Servlet — use the request's resolver (user's own session)
@Override
protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response) {
    // request.getResourceResolver() is already the authenticated user's session
    ResourceResolver resolver = request.getResourceResolver();  // DON'T close this one!
    Resource page = resolver.getResource("/content/mysite/home");
    // resolver is managed by Sling's request lifecycle — don't close it here
}
```

---

## Q7. Memory Leak Prevention — 5 Golden Rules

```java
@Component(service = Runnable.class, immediate = true)
public class SafeScheduler implements Runnable {

    @Reference
    private Scheduler scheduler;

    @Reference
    private ResourceResolverFactory resolverFactory;

    private String jobName = "MySafeScheduler";

    @Activate
    protected void activate(Map<String, Object> config) {
        // RULE 1: Use canRunConcurrently(false) to prevent pile-up
        ScheduleOptions opts = scheduler.EXPR("0 0 2 * * ?");
        opts.name(jobName);
        opts.canRunConcurrently(false);
        scheduler.schedule(this, opts);
    }

    @Deactivate
    protected void deactivate() {
        // RULE 2: Always unschedule in @Deactivate
        scheduler.unschedule(jobName);
        LOG.info("Scheduler '{}' unscheduled", jobName);
    }

    @Override
    public void run() {
        LOG.info("Scheduler starting...");

        // RULE 3: NEVER store ResourceResolver as class field
        // RULE 4: ALWAYS use try-with-resources for ResourceResolver
        Map<String, Object> params = Collections.singletonMap(
            ResourceResolverFactory.SUBSERVICE, "scheduler-reader"
        );

        try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(params)) {
            doWork(resolver);
        } catch (LoginException e) {
            LOG.error("Cannot get ResourceResolver", e);
        } catch (Exception e) {
            LOG.error("Scheduler execution failed", e);
        }
        // resolver is auto-closed here

        LOG.info("Scheduler finished.");
    }

    private void doWork(ResourceResolver resolver) {
        // RULE 5: Don't hold references to request-scoped or session-scoped objects
        // Only use the passed-in resolver within this method scope
        Resource content = resolver.getResource("/content/mysite");
        if (content != null) {
            LOG.info("Content root exists: {}", content.getPath());
        }
    }
}
```

---

## Q8. Replication Queue — Troubleshooting Guide

### Replication Flow
```
Author Page Activation
        ↓
Replication Agent (configured in Tools → Deployment → Replication)
        ↓
HTTP POST to Publish: /bin/receive?sling:authRequestLogin=1
        ↓
Publish stores content in JCR
        ↓
Publish sends cache invalidation to Dispatcher
        ↓
Dispatcher clears cached HTML for that page
        ↓
Next request: fresh content served
```

### Replication Queue States

| State | Meaning | Action |
|-------|---------|--------|
| **IDLE** | No pending items — healthy | None needed |
| **RUNNING** | Actively sending content | Normal |
| **QUEUED** | Items waiting (publish busy) | Wait or check publish |
| **BLOCKED** | Error — queue stopped | Check logs, fix error, retry |

### Diagnosing a Blocked Queue

```
Tools → Deployment → Replication → Agents on Author → Publish Agent → Edit

TEST: Click "Test Connection"
→ 200 OK → Connection fine — check content or auth
→ 401 Unauthorized → Wrong transport user credentials
→ 503 Connection Refused → Publish instance down
→ Timeout → Network issue or publish overloaded

Check Replication Queue:
→ Author → Tools → Deployment → Distribution → Queue Details
→ See failed items, retry or delete

Check Error Log:
→ /crx-quickstart/logs/error.log on Author
→ Search for "ReplicationException"

Common Fixes:
1. Restart the blocked agent: Edit agent → Clear queue → Test Connection
2. Check Publish is running: curl -I http://publish:4503/content/mysite/home.html
3. Check transport user credentials in the agent settings
4. Check Dispatcher is not blocking /bin/receive
```

---

## 💡 Quick-Fire Q&A — Memorize These

| Question | Answer |
|----------|--------|
| Default Author port? | 4502 |
| Default Publish port? | 4503 |
| Where are components stored? | `/apps/<site>/components/` |
| Where is authored content? | `/content/` |
| Where are editable templates? | `/conf/<site>/settings/wcm/templates/` |
| Where are OSGi configs? | `/apps/<site>/osgiconfig/config*/` |
| Where are system users? | `/home/users/system/` |
| What is Oak? | JCR implementation (Apache Jackrabbit Oak) |
| What is Felix? | OSGi container (Apache Felix) |
| What is Sling? | REST web framework for URL→JCR mapping |
| What is a ValueMap? | Type-safe map of JCR node properties |
| What is nt:unstructured? | Generic node type for component content |
| What is cq:Page? | Node type for AEM pages |
| What is jcr:content? | Child node storing page properties |
| What does "activation" mean? | Publishing content from Author to Publish |
| What is AEM Cloud immutable? | Code layer can't be changed post-deployment |
| What is @PostConstruct for? | Runs after all injections — for initialization |
| When does @Activate run? | Once when OSGi bundle/component starts |
| When does @Deactivate run? | When bundle stops — release resources here! |
| What does allowProxy do? | Allows clientlibs to be served via /etc.clientlibs/ |

---

## 🎯 Power Phrases for Your Interview

Use these to sound like a senior developer:

- *"I always use `DefaultInjectionStrategy.OPTIONAL` to avoid null injection failures silently crashing the model..."*
- *"For service users, I follow the principle of least privilege — each subservice gets only the specific JCR paths it needs..."*
- *"In AEM as a Cloud Service, the immutable architecture means all code changes must go through Cloud Manager pipeline — there's no Package Manager in production..."*
- *"I prefer resource-type servlet registration over path-based for better security and content-level permission inheritance..."*
- *"When extending Core Components, I use `@Self @Via(type = ResourceSuperType.class)` to delegate to the core model and add only my custom properties..."*
- *"In `@Deactivate`, I always close HTTP clients, cancel scheduled jobs, and clear caches to prevent memory leaks..."*
- *"For OSGi configuration, I store configs in run-mode specific folders — `config.author.prod` for author production, `config.publish` for all publish instances..."*
- *"The Dispatcher is critical for performance — it caches rendered HTML, so AEM only renders a page once until the cache is invalidated by a publish event..."*
