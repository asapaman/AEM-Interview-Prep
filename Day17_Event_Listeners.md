# Event Listeners & Observation in AEM
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What Are Event Listeners in AEM?

**Event Listeners** let your code react to things that happen in AEM — page published, asset uploaded, content changed, replication triggered — without polling. They're the foundation of reactive AEM programming.

### Event Systems in AEM

AEM has **three** event systems you need to know:

| System | API | Best For |
|--------|-----|---------|
| **OSGi EventAdmin** | `EventHandler` service | AEM/Sling-level events (page publish, replication) |
| **JCR Observation** | `javax.jcr.observation.EventListener` | Raw JCR node/property changes |
| **Sling Resource Change Listener** | `ResourceChangeListener` | Modern Sling resource change events |

---

## 1️⃣ OSGi EventAdmin — The Most Common Approach

### How It Works

```
AEM fires a topic-based event → OSGi EventAdmin → All registered EventHandlers receive it

Example events:
- "com/day/cq/wcm/core/page/Event#MODIFIED"  → Page content changed
- "com/day/cq/wcm/core/page/Event#DELETED"   → Page deleted
- "com/day/cq/replication/event"              → Replication (publish/unpublish)
- "com/day/cq/dam/api/DamEvent"               → DAM asset events
- "com/day/cq/auth/jwt/usermanagement/event"  → User management
```

### Page Event Handler — Complete Example

```java
package com.mysite.core.listeners;

import com.day.cq.wcm.api.PageEvent;
import com.day.cq.wcm.api.PageModification;
import org.apache.sling.api.resource.*;
import org.osgi.service.component.annotations.*;
import org.osgi.service.event.Event;
import org.osgi.service.event.EventConstants;
import org.osgi.service.event.EventHandler;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.List;

/**
 * Listens for page events: created, modified, deleted, published, unpublished.
 * Use case: Invalidate an external cache when pages change.
 */
@Component(
    service = EventHandler.class,
    immediate = true,  // Start immediately — don't wait for a consumer
    property = {
        // Subscribe to ALL page events
        EventConstants.EVENT_TOPIC + "=" + PageEvent.EVENT_TOPIC,
        // Optional: filter to specific event types
        // EventConstants.EVENT_FILTER + "=(type=PageActivated)"
    }
)
public class PageChangeEventHandler implements EventHandler {

    private static final Logger LOG = LoggerFactory.getLogger(PageChangeEventHandler.class);

    // OSGi service dependency
    @Reference
    private ResourceResolverFactory resolverFactory;

    @Reference
    private CacheInvalidationService cacheService;  // Your custom service

    @Override
    public void handleEvent(Event event) {
        // Extract page modifications from the event
        List<PageModification> modifications = PageEvent.fromEvent(event).getModifications();

        for (PageModification modification : modifications) {
            String pagePath = modification.getPath();
            PageModification.ModificationType type = modification.getType();

            LOG.info("Page event: {} on path: {}", type, pagePath);

            // Only process pages under our site
            if (pagePath == null || !pagePath.startsWith("/content/mysite")) {
                continue;
            }

            // React to specific event types
            switch (type) {
                case CREATED:
                    LOG.info("New page created: {}", pagePath);
                    break;

                case MODIFIED:
                    LOG.info("Page modified: {} — invalidating cache", pagePath);
                    cacheService.invalidate(pagePath);
                    break;

                case DELETED:
                    LOG.info("Page deleted: {}", pagePath);
                    cacheService.remove(pagePath);
                    break;

                case PUBLISHED:  // = Activated
                    LOG.info("Page published: {} — warming CDN cache", pagePath);
                    triggerCacheWarm(pagePath);
                    break;

                case UNPUBLISHED:  // = Deactivated
                    LOG.info("Page unpublished: {}", pagePath);
                    cacheService.purge(pagePath);
                    break;

                default:
                    LOG.debug("Unhandled page event type: {}", type);
            }
        }
    }

    private void triggerCacheWarm(String pagePath) {
        // Implementation to pre-warm cache for newly published page
    }
}
```

### DAM Asset Event Handler

```java
package com.mysite.core.listeners;

import com.day.cq.dam.api.DamEvent;
import org.osgi.service.component.annotations.*;
import org.osgi.service.event.*;

/**
 * Listens for DAM asset events: uploaded, modified, deleted, published.
 * Use case: Trigger metadata sync with a PIM (Product Information Manager).
 */
@Component(
    service = EventHandler.class,
    immediate = true,
    property = {
        EventConstants.EVENT_TOPIC + "=" + DamEvent.EVENT_TOPIC
    }
)
public class AssetEventHandler implements EventHandler {

    private static final Logger LOG = LoggerFactory.getLogger(AssetEventHandler.class);

    @Reference
    private MetadataSyncService metadataSyncService;

    @Override
    public void handleEvent(Event event) {
        DamEvent damEvent = DamEvent.fromEvent(event);
        if (damEvent == null) return;

        String assetPath = damEvent.getAssetPath();
        DamEvent.Type eventType = damEvent.getType();

        // Only handle image assets under our product folder
        if (!assetPath.startsWith("/content/dam/mysite/products/")) return;

        LOG.info("DAM event: {} on asset: {}", eventType, assetPath);

        switch (eventType) {
            case ASSET_INGESTION:   // Asset uploaded for the first time
            case METADATA_UPDATED:  // Metadata changed in DAM UI
                metadataSyncService.syncToExternalPIM(assetPath);
                break;

            case ASSET_REMOVED:
                metadataSyncService.removeFromExternalPIM(assetPath);
                break;

            case PUBLISHED:
                LOG.info("Asset published — CDN URL ready: {}", assetPath);
                break;

            default:
                break;
        }
    }
}
```

### Replication Event Handler

```java
@Component(
    service = EventHandler.class,
    immediate = true,
    property = {
        EventConstants.EVENT_TOPIC + "=com/day/cq/replication"
    }
)
public class ReplicationEventHandler implements EventHandler {

    @Override
    public void handleEvent(Event event) {
        String path    = (String) event.getProperty("path");
        String action  = (String) event.getProperty("type");  // "Activate" or "Deactivate"
        boolean success = Boolean.TRUE.equals(event.getProperty("succeeded"));

        LOG.info("Replication {} on {} — success: {}", action, path, success);

        if ("Activate".equals(action) && success && path != null
                && path.startsWith("/content/mysite")) {
            // Notify an external CDN to purge/warm the cached URL
            cdnService.notifyPublished(path);
        }
    }
}
```

---

## 2️⃣ Sling Resource Change Listener (Modern Approach)

`ResourceChangeListener` is the Sling-level alternative to JCR Observation. More abstracted, easier to use.

```java
package com.mysite.core.listeners;

import org.apache.sling.api.resource.observation.ResourceChange;
import org.apache.sling.api.resource.observation.ResourceChangeListener;
import org.osgi.service.component.annotations.*;

import java.util.List;

/**
 * Listens for changes to resources (nodes) at specific paths.
 * Preferred over JCR Observation for most Sling-level use cases.
 */
@Component(
    service = ResourceChangeListener.class,
    immediate = true,
    property = {
        // Watch these paths
        ResourceChangeListener.PATHS + "=/content/mysite",
        ResourceChangeListener.PATHS + "=/conf/mysite",

        // Watch specific change types
        ResourceChangeListener.CHANGES + "=ADDED",
        ResourceChangeListener.CHANGES + "=CHANGED",
        ResourceChangeListener.CHANGES + "=REMOVED",

        // Only fire for persistent (saved) changes (not transient/in-memory)
        // ResourceChangeListener.PROPERTY_NAMES_HINT + "=jcr:title,sling:resourceType"
    }
)
public class ContentChangeListener implements ResourceChangeListener {

    private static final Logger LOG = LoggerFactory.getLogger(ContentChangeListener.class);

    @Reference
    private SearchIndexService searchIndexService;

    @Override
    public void onChange(List<ResourceChange> changes) {
        for (ResourceChange change : changes) {
            String path = change.getPath();
            ResourceChange.ChangeType type = change.getType();

            LOG.debug("Resource change: {} on {}", type, path);

            // Only care about page content changes
            if (path.contains("/jcr:content")) {
                switch (type) {
                    case ADDED:
                    case CHANGED:
                        // Re-index page in search
                        searchIndexService.reindex(extractPagePath(path));
                        break;
                    case REMOVED:
                        searchIndexService.deindex(extractPagePath(path));
                        break;
                }
            }
        }
    }

    private String extractPagePath(String contentPath) {
        // /content/mysite/en/home/jcr:content → /content/mysite/en/home
        int jcrContentIndex = contentPath.indexOf("/jcr:content");
        return jcrContentIndex > 0 ? contentPath.substring(0, jcrContentIndex) : contentPath;
    }
}
```

---

## 3️⃣ JCR Observation (Low-Level)

**Use only when** you need raw JCR-level events that ResourceChangeListener doesn't cover.

```java
package com.mysite.core.listeners;

import javax.jcr.*;
import javax.jcr.observation.*;
import org.apache.sling.jcr.api.SlingRepository;
import org.osgi.service.component.annotations.*;

@Component(immediate = true)
public class JcrObservationListener {

    private static final Logger LOG = LoggerFactory.getLogger(JcrObservationListener.class);

    @Reference
    private SlingRepository slingRepository;

    private Session observationSession;   // Dedicated session for observation
    private EventListener eventListener;

    @Activate
    protected void activate() throws RepositoryException {
        // Create a dedicated session for observation (NEVER share with request sessions)
        observationSession = slingRepository.loginAdministrative(null);
        // ⚠️ In production: use loginService with a service user instead

        ObservationManager observationManager =
            observationSession.getWorkspace().getObservationManager();

        eventListener = (EventIterator events) -> {
            while (events.hasNext()) {
                try {
                    Event event = events.nextEvent();
                    String path = event.getPath();
                    int type    = event.getType();

                    LOG.debug("JCR event: {} at {}", typeToString(type), path);
                } catch (RepositoryException e) {
                    LOG.error("Error processing JCR event", e);
                }
            }
        };

        // Register the listener
        observationManager.addEventListener(
            eventListener,
            Event.NODE_ADDED | Event.NODE_REMOVED | Event.PROPERTY_CHANGED | Event.PROPERTY_ADDED,
            "/content/mysite",  // Root path to observe
            true,               // Deep: also observe all descendants
            null,               // UUID filter (null = all nodes)
            null,               // Node type filter (null = all types)
            false               // noLocal: false = receive events from this session too
        );

        LOG.info("JCR Observation registered on /content/mysite");
    }

    @Deactivate
    protected void deactivate() {
        // CRITICAL: Always unregister and close in @Deactivate!
        if (observationSession != null && observationSession.isLive()) {
            try {
                ObservationManager om = observationSession.getWorkspace().getObservationManager();
                om.removeEventListener(eventListener);
            } catch (RepositoryException e) {
                LOG.error("Error removing event listener", e);
            }
            observationSession.logout();
            observationSession = null;
        }
    }

    private String typeToString(int type) {
        switch (type) {
            case Event.NODE_ADDED:        return "NODE_ADDED";
            case Event.NODE_REMOVED:      return "NODE_REMOVED";
            case Event.PROPERTY_ADDED:    return "PROPERTY_ADDED";
            case Event.PROPERTY_CHANGED:  return "PROPERTY_CHANGED";
            case Event.PROPERTY_REMOVED:  return "PROPERTY_REMOVED";
            default:                      return "UNKNOWN(" + type + ")";
        }
    }
}
```

---

## ⚡ Custom OSGi Events — Publish and Subscribe

You can publish your own custom events and have multiple handlers react to them.

```java
// PUBLISHER — fire a custom event
@Component(service = MyEventPublisher.class)
public class MyEventPublisher {

    @Reference
    private EventAdmin eventAdmin;

    public void notifyProductUpdated(String productId, String newPrice) {
        Dictionary<String, Object> properties = new Hashtable<>();
        properties.put("productId",  productId);
        properties.put("newPrice",   newPrice);
        properties.put("timestamp",  System.currentTimeMillis());

        Event event = new Event(
            "com/mysite/events/product/UPDATED",  // Custom topic
            properties
        );

        eventAdmin.postEvent(event);   // Async — non-blocking
        // OR: eventAdmin.sendEvent(event)  // Sync — blocks until all handlers complete
    }
}

// SUBSCRIBER — react to the custom event
@Component(
    service = EventHandler.class,
    property = {
        EventConstants.EVENT_TOPIC + "=com/mysite/events/product/UPDATED"
    }
)
public class ProductUpdateHandler implements EventHandler {

    @Override
    public void handleEvent(Event event) {
        String productId = (String) event.getProperty("productId");
        String newPrice  = (String) event.getProperty("newPrice");
        LOG.info("Product {} updated to price: {}", productId, newPrice);
        // Trigger downstream updates (search index, cache, etc.)
    }
}
```

---

## ☁️ AEM 6.5 vs Cloud — Event Handling Differences

| Aspect | AEM 6.5 | AEM Cloud |
|--------|---------|-----------|
| **OSGi EventAdmin** | Available | Available |
| **ResourceChangeListener** | Available | Available |
| **JCR Observation** | Available | Available but limited |
| **Deployment count** | Single JVM | Multiple JVM replicas |
| **Event distribution** | Local JVM only | ⚠️ Events don't cross JVM replicas! |
| **Cross-instance events** | N/A (single AEM) | Use Adobe I/O Events instead |

> **⚠️ AEM Cloud Important:** EventHandlers fire only on the JVM instance that processes the event. In AEM Cloud, there can be multiple Author/Publish replicas. For cross-instance event handling, use **Adobe I/O Events** (cloud-native pub/sub).

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is the difference between OSGi EventAdmin and JCR Observation?**

> **Answer:** OSGi EventAdmin is a higher-level, topic-based pub/sub system — great for AEM application events (page published, asset uploaded) via the `EventHandler` service. JCR Observation is lower-level, operating directly on JCR node/property changes — useful when you need raw JCR-level change events with specific type/path filtering. For most AEM development, prefer OSGi EventAdmin or `ResourceChangeListener` over raw JCR Observation — they're simpler and integrate better with Sling.

**Q2. What is `ResourceChangeListener` and how does it differ from JCR Observation?**

> **Answer:** `ResourceChangeListener` is Sling's abstraction over JCR Observation — it batches changes and delivers them as a `List<ResourceChange>`. It's simpler to implement (just one method: `onChange()`), integrates with Sling's resource model, and is the modern preferred approach. JCR Observation fires per-event and operates at the raw JCR level (nodes, properties, types). For content change reactions in AEM 6.5+, always use `ResourceChangeListener`.

**Q3. What memory leak risk exists with JCR Observation?**

> **Answer:** If you register a JCR `EventListener` and don't unregister it in `@Deactivate`, the listener keeps the session alive indefinitely — causing a session/connection leak and consuming memory. Always: 1) Store the `observationSession` and `eventListener` as fields, 2) In `@Deactivate`, call `observationManager.removeEventListener(eventListener)` then `observationSession.logout()`. Same applies to the dedicated session — never share it with request sessions.

**Q4. Why do AEM Cloud Event Handlers need special consideration?**

> **Answer:** AEM Cloud runs multiple JVM replicas for scalability. An OSGi EventHandler fires only on the JVM instance that receives the triggering event — other replicas don't receive it. If you're doing local cache invalidation in an EventHandler, only one replica's cache gets invalidated. For cross-instance operations, use Adobe I/O Events, which is Adobe's cloud-native event streaming system.

**Q5. What is `eventAdmin.postEvent()` vs `sendEvent()`?**

> **Answer:** `postEvent()` is asynchronous — the event is queued and handlers receive it on a separate thread; the caller returns immediately. `sendEvent()` is synchronous — the caller blocks until all registered handlers have finished processing the event. Use `postEvent()` for most cases (don't block the calling thread). Use `sendEvent()` only when you must ensure all handlers complete before proceeding (rare, but useful for audit-critical operations).

---

## ✅ Best Practices

1. **Always unregister in `@Deactivate`** — JCR Observation listeners are memory leaks if not cleaned up
2. **Use dedicated sessions for JCR Observation** — never share with request-scoped sessions
3. **Prefer `ResourceChangeListener` over JCR Observation** — simpler, more idiomatic
4. **Path-filter your event handlers** — don't process events from paths outside your concern
5. **Keep event handlers fast** — long-running operations in handlers block the event queue; offload to an async job/thread pool
6. **Use `postEvent()` for custom events** — async is almost always preferred
7. **In AEM Cloud** — use Adobe I/O Events for anything that needs to cross JVM instances

---

## 🛠️ Hands-on Practice

1. Create a `PageChangeEventHandler` that logs every page modification under `/content/mysite` — test by modifying a page in Author
2. Create a `ResourceChangeListener` that fires when anything changes under `/content/mysite/en` — verify it catches property changes
3. Create a custom event publisher and handler — fire `com/mysite/events/test` and have a handler log it
4. (JCR Observation) Write a `JcrObservationListener` — verify proper cleanup in `@Deactivate` by checking `/system/console/components` after bundle restart
5. Test the memory leak: create a listener WITHOUT proper `@Deactivate` cleanup — observe the growing session count in JMX
