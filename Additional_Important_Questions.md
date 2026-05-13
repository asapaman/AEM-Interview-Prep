# Additional Important Questions — Must Know for 4 YOE Interview
**Focus: Senior-Level Deep Dives**

---

## Q1. Difference Between Static and Editable Templates

| Aspect | Static Templates | Editable Templates |
|--------|-----------------|-------------------|
| **Location** | `/apps/<site>/templates/` | `/conf/<site>/settings/wcm/templates/` |
| **Managed by** | Developers only | Template authors (+ developers) |
| **Component restrictions** | Design dialogs + design mode | Content Policies |
| **Structure locking** | Not supported | Structure layer (locked components) |
| **Initial content** | Not supported | Initial layer |
| **Style System** | ❌ | ✅ |
| **Best Practice** | ❌ Legacy | ✅ Modern standard |

**When asked:** Always say you use Editable Templates in modern projects. Static templates are legacy.

---

## Q2. How Does Dispatcher Caching Work?

### Dispatcher Cache Flow
```
Request → Dispatcher checks /var/cache/<site>/<path>.html
  → Cache HIT → Serve cached file directly
  → Cache MISS → Forward to AEM Publish → Get response → Store in cache → Return
```

### Cache Invalidation
When content is activated on Author:
1. Replication agent sends **activation** signal to Publish
2. Publish sends **cache invalidation** request to Dispatcher
3. Dispatcher marks affected pages as "stale" (deletes `.stat` files)
4. Next request for those pages goes through to Publish and re-caches

### Dispatcher Configuration (Key Files)
```apache
# dispatcher.any - Main config
/cache {
    /docroot "/var/cache"
    /rules {
        /0000 { /type "allow" /url "*" }
        /0001 { /type "deny" /url "*.html" /glob "/content/private/*" }
    }
    /invalidate {
        /0000 { /type "allow" /glob "*" }
    }
    /headers {
        "Cache-Control"
        "Expires"
    }
}
```

### What Dispatcher Does NOT Cache (by default)
- POST requests
- Requests with query strings (unless configured)
- Content under `/bin/`, `/apps/`, `/libs/` (security)
- Responses without extension
- Authenticated requests (by default)

---

## Q3. What Are AEM Run Modes?

Run modes activate specific OSGi configurations based on environment:

```
-Dsling.run.modes=publish,prod
```

### OSGi Config Resolution Order (most specific wins)
```
config.author.prod → config.publish.prod → config.author → config.publish → config.prod → config
```

### Common Run Modes
| Run Mode | Purpose |
|----------|---------|
| `author` | AEM Author instance |
| `publish` | AEM Publish instance |
| `dev` | Development environment |
| `stage` | Staging environment |
| `prod` | Production environment |

### Checking Run Mode in Code
```java
@Reference
private SlingSettingsService slingSettings;

public boolean isPublishMode() {
    return slingSettings.getRunModes().contains("publish");
}
```

---

## Q4. Difference Between cq:dialog and cq:design_dialog

| Aspect | `cq:dialog` | `cq:design_dialog` |
|--------|------------|-------------------|
| **Purpose** | Instance-level properties | Design/template-level properties |
| **Who edits** | Content authors | Template authors (design mode) |
| **Scope** | Per component instance | Shared across all instances |
| **Modern equivalent** | `cq:dialog` (unchanged) | Content Policies (Editable Templates) |
| **UI** | Touch UI dialog | Classic UI (mostly legacy) |

**Key insight for interview:** `cq:design_dialog` is largely replaced by Content Policies in modern AEM. Mention this when asked.

---

## Q5. How Do You Debug Sling Models?

### Debugging Checklist
```
1. Check bundle status: /system/console/bundles
   → Bundle must be "Active"

2. Check model registration: /system/console/adapters
   → Search for your class name

3. Check compilation errors: /system/console/errors
   → Look for your bundle name

4. Enable DEBUG logging via Felix Console:
   → /system/console/slinglog
   → Add logger: org.apache.sling.models = DEBUG

5. Use HTL debugging:
   → <sly data-sly-test="${model == null}">MODEL IS NULL!</sly>

6. Check @Model annotation:
   → adaptables matches usage (Resource vs Request)
   → resourceType matches component path

7. AEM Dev Tools:
   → /crx/de → navigate to content node, check properties
   → Sling Servlet Resolver: /system/console/servletresolver
```

### Common Issues
| Symptom | Likely Cause |
|---------|-------------|
| Model returns null | Wrong adaptable (Resource vs Request) |
| Properties all null | Missing `DefaultInjectionStrategy.OPTIONAL` |
| Class not found | Bundle not active, package not exported |
| NPE in `@PostConstruct` | Null child resource not checked |

---

## Q6. What Is ResourceResolverFactory?

`ResourceResolverFactory` is the OSGi service for obtaining `ResourceResolver` instances.

```java
@Reference
private ResourceResolverFactory resolverFactory;

// ✅ Correct: Service user (for schedulers, services, workflows)
Map<String, Object> params = Collections.singletonMap(
    ResourceResolverFactory.SUBSERVICE, "my-subservice"
);
try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(params)) {
    // Use resolver
}

// ❌ NEVER USE: Admin resolver (deprecated, insecure)
// resolverFactory.getAdministrativeResourceResolver(null);
```

### Always Close ResourceResolver!
```java
// Option 1: try-with-resources (preferred)
try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(params)) {
    Resource r = resolver.getResource("/content/mysite");
} // Auto-closed

// Option 2: explicit finally
ResourceResolver resolver = null;
try {
    resolver = resolverFactory.getServiceResourceResolver(params);
} finally {
    if (resolver != null && resolver.isLive()) {
        resolver.close();
    }
}
```

---

## Q7. How Do You Avoid Memory Leaks in Schedulers?

### 5 Rules to Avoid Memory Leaks

```java
@Component(service = Runnable.class, immediate = true)
public class SafeScheduler implements Runnable {

    @Reference
    private ResourceResolverFactory resolverFactory;

    @Reference
    private Scheduler scheduler;

    private String jobName = "SafeScheduler";

    @Activate
    protected void activate(Map<String, Object> config) {
        ScheduleOptions opts = scheduler.EXPR("0 0 2 * * ?");
        opts.name(jobName);
        opts.canRunConcurrently(false);  // Rule 1: No concurrent runs
        scheduler.schedule(this, opts);
    }

    @Deactivate
    protected void deactivate() {
        scheduler.unschedule(jobName);  // Rule 2: Always unschedule on deactivate
    }

    @Override
    public void run() {
        // Rule 3: Never store ResourceResolver as class field
        Map<String, Object> params = Collections.singletonMap(
            ResourceResolverFactory.SUBSERVICE, "scheduler-service"
        );

        // Rule 4: Always use try-with-resources for ResourceResolver
        try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(params)) {
            doWork(resolver);
        } catch (LoginException e) {
            LOG.error("Login failed", e);
        }
        // Rule 5: Don't hold references to request-scoped objects
    }

    private void doWork(ResourceResolver resolver) {
        // Actual business logic
    }
}
```

---

## Q8. Difference Between Forward and Redirect in Servlets

```java
// FORWARD: Server-side, same request, URL unchanged
// Use for: Internal page composition, error pages
request.getRequestDispatcher("/content/error.html").forward(request, response);

// REDIRECT: Client-side, new request, URL changes  
// Use for: After form POST (PRG pattern), external URLs
response.sendRedirect("/content/success.html");        // 302 (temporary)
response.setStatus(301); response.setHeader("Location", "/new-url.html"); // 301 (permanent)
```

| Aspect | Forward | Redirect |
|--------|---------|---------|
| Request count | 1 | 2 |
| URL changes | ❌ | ✅ |
| Request object | Same | New |
| Performance | Faster | Slower |
| POST-Redirect-GET | Can cause re-submission | ✅ Prevents re-submission |

---

## Q9. What Is Service User Mapping?

Service user mapping links an **OSGi bundle + subservice name** to a **JCR system user**.

```json
// OSGi Config: org.apache.sling.serviceusermapping.impl.ServiceUserMapperImpl.amended-mysite.cfg.json
{
    "user.mapping": [
        "com.mysite.core:read-content=mysite-content-reader",
        "com.mysite.core:write-dam=mysite-dam-writer"
    ]
}
```

Pattern: `bundle-symbolic-name:subservice-name=system-user-id`

Usage in code:
```java
params.put(ResourceResolverFactory.SUBSERVICE, "read-content");
```

---

## Q10. How Does Replication Queue Work?

### Replication Flow
```
Author JCR → Replication Agent → HTTP POST (binary/json) → Replication Servlet on Publish → Publish JCR
```

### Replication Queue States
| State | Meaning |
|-------|---------|
| **IDLE** | No pending items |
| **RUNNING** | Actively processing |
| **BLOCKED** | Queue paused (error) |
| **QUEUED** | Items waiting to be sent |

### Common Replication Issues + Fixes
| Issue | Cause | Fix |
|-------|-------|-----|
| Queue blocked | Publish instance down | Restart Publish, or retry queue |
| Agent 401 error | Wrong credentials | Update transport user in replication agent |
| Content not appearing | Dispatcher cache stale | Flush Dispatcher cache manually |
| Large content slow | Binary content | Use Binary-less replication + Shared Datastore |

### Check Replication Queue
- Author → Tools → Deployment → Replication → Agents on Author
- Or: `/etc/replication/agents.author/publish.html`

### Reverse Replication
Publish → Author direction. Used for:
- User-generated content (UGC) → storing form submissions from Publish to Author
- Statistics collection

---

## 💡 Quick-Fire Q&A Reference

| Question | Quick Answer |
|----------|-------------|
| What port is AEM Author? | `4502` |
| What port is AEM Publish? | `4503` |
| Where are OSGi configs stored? | `/apps/.../osgiconfig/config/` (AEMaaCS) |
| What is Oak? | Apache's JCR implementation (content repo) |
| What is Sling? | URL → JCR resource resolution framework |
| What is Felix? | Apache's OSGi container |
| What is the AEM project archetype? | Maven archetype for AEM project structure |
| What is the `cq:Page` node type? | JCR node type for AEM pages |
| What is `jcr:content`? | Child node of cq:Page storing page properties |
| What does "activation" mean? | Publishing content from Author → Publish |
| What is `nt:unstructured`? | Generic JCR node type (for component content) |
| What is `sling:Folder`? | JCR node type for folder-like nodes |
| What is a Sling Resource? | Abstraction over a JCR node with URL context |
| What is a ValueMap? | Type-safe map of JCR properties |
