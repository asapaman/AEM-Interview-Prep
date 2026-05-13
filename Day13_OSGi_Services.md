# Day 12–15 — OSGi & Services (Complete Guide)
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What is OSGi? (Start Here)

**OSGi (Open Services Gateway initiative)** is a Java framework for building **modular applications**. In AEM, all Java code is deployed as OSGi **bundles** (special JAR files), and services communicate via OSGi's **service registry**.

### Why OSGi in AEM?
- **Hot deployment:** Install, update, or remove bundles WITHOUT restarting the entire AEM JVM
- **Modular:** Each bundle is independent with its own classpath
- **Service registry:** Bundles can register services and discover/consume each other's services
- **Configuration:** Externalize configuration via OSGi Config Manager without code changes

### OSGi = Bundle + Component + Service

| Concept | Description | Java Analog |
|---------|------------|-------------|
| **Bundle** | JAR file with OSGi metadata (MANIFEST.MF specifying imports/exports) | Maven module |
| **Component** | Java class managed by OSGi (`@Component`) | Spring Bean |
| **Service** | Component registered in service registry under an interface | Spring Service |
| **Reference** | A dependency on another service | `@Autowired` in Spring |

---

## 🏗️ Complete OSGi Service Example

Let's build a complete, production-ready OSGi service from scratch:

### Step 1: Define the Interface
```java
package com.mysite.core.services;

import java.util.List;

/**
 * Service for fetching product data from an external API.
 *
 * Why define an interface?
 * 1. Code to interface → easier to mock in unit tests
 * 2. Multiple implementations possible (prod API, mock, cache-only)
 * 3. Decoupled from implementation details
 */
public interface ProductService {

    /**
     * Fetches a single product by ID.
     * @param productId The product identifier
     * @return Product data, or null if not found
     */
    Product getProduct(String productId);

    /**
     * Fetches products in a category.
     * @param categoryId The category identifier
     * @return List of products (empty list if none found, never null)
     */
    List<Product> getProductsByCategory(String categoryId);

    /**
     * Checks if the service is healthy/available.
     */
    boolean isHealthy();
}
```

### Step 2: Create the Configuration Interface
```java
package com.mysite.core.services.impl;

import org.osgi.service.metatype.annotations.AttributeDefinition;
import org.osgi.service.metatype.annotations.ObjectClassDefinition;

/**
 * @ObjectClassDefinition = Makes this config appear in OSGi Config Manager UI
 * Factory configs use factoryPid instead of pid
 */
@ObjectClassDefinition(
    name = "My Site - Product Service Configuration",
    description = "Configuration for the external Product API integration"
)
public @interface ProductServiceConfig {

    @AttributeDefinition(
        name = "API Base URL",
        description = "Base URL of the product API (e.g., https://api.mysite.com/v1)"
    )
    String apiBaseUrl() default "https://api.mysite.com/v1";

    @AttributeDefinition(
        name = "API Key",
        description = "Authentication key for the API"
    )
    String apiKey() default "";

    @AttributeDefinition(
        name = "Cache TTL (seconds)",
        description = "How long to cache API responses. Set to 0 to disable caching."
    )
    int cacheTtlSeconds() default 300;  // 5 minutes

    @AttributeDefinition(
        name = "Connection Timeout (ms)",
        description = "HTTP connection timeout in milliseconds"
    )
    int connectionTimeoutMs() default 5000;

    @AttributeDefinition(
        name = "Service Enabled",
        description = "Enable/disable this service without undeploying"
    )
    boolean enabled() default true;
}
```

### Step 3: Implement the Service
```java
package com.mysite.core.services.impl;

import com.mysite.core.services.ProductService;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.osgi.service.component.annotations.*;
import org.osgi.service.metatype.annotations.Designate;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

/**
 * @Component — Marks this class as an OSGi-managed component
 *   service = ProductService.class → Registers in service registry under this interface
 *   immediate = true → Activate immediately when bundle starts (not lazily)
 *
 * @Designate — Links this component to its config class
 */
@Component(
    service = ProductService.class,
    immediate = true
)
@Designate(ocd = ProductServiceConfig.class)
public class ProductServiceImpl implements ProductService {

    private static final Logger LOG = LoggerFactory.getLogger(ProductServiceImpl.class);

    // Config values (populated in @Activate)
    private String apiBaseUrl;
    private String apiKey;
    private int cacheTtlSeconds;
    private boolean enabled;

    // Simple in-memory cache
    private final Map<String, CacheEntry> cache = new ConcurrentHashMap<>();

    /**
     * @Activate — Called when the OSGi component starts (bundle activation or config change)
     *
     * This method is called:
     * 1. When the bundle first starts
     * 2. After @Modified (config update)
     * NOT called on every request — only once at startup!
     */
    @Activate
    protected void activate(ProductServiceConfig config) {
        this.apiBaseUrl = config.apiBaseUrl();
        this.apiKey = config.apiKey();
        this.cacheTtlSeconds = config.cacheTtlSeconds();
        this.enabled = config.enabled();

        LOG.info("ProductService activated. URL: {}, Cache TTL: {}s, Enabled: {}",
            apiBaseUrl, cacheTtlSeconds, enabled);

        // Clear cache on config change
        cache.clear();
    }

    /**
     * @Modified — Called when OSGi config changes WITHOUT restarting the component
     *
     * Use case: An admin updates the API URL in Felix Console Config Manager
     * → @Modified fires, component reloads config WITHOUT a full restart
     */
    @Modified
    protected void modified(ProductServiceConfig config) {
        LOG.info("ProductService config modified — reloading");
        activate(config);  // Re-use activate logic (common pattern)
    }

    /**
     * @Deactivate — Called when the component/bundle is stopped
     *
     * CRITICAL: Release all resources here!
     * - Close HTTP clients
     * - Unregister listeners
     * - Clear caches
     * - Stop background threads
     *
     * Failing to do this = MEMORY LEAKS
     */
    @Deactivate
    protected void deactivate() {
        LOG.info("ProductService deactivated — clearing resources");
        cache.clear();
        // Close any persistent connections or thread pools here
    }

    @Override
    public Product getProduct(String productId) {
        if (!enabled || productId == null) return null;

        // Check cache first
        String cacheKey = "product:" + productId;
        Product cached = getFromCache(cacheKey);
        if (cached != null) {
            LOG.debug("Cache HIT for product: {}", productId);
            return cached;
        }

        LOG.debug("Cache MISS for product: {} — fetching from API", productId);

        try {
            String url = apiBaseUrl + "/products/" + productId;
            Product product = callApi(url, Product.class);
            if (product != null) {
                putInCache(cacheKey, product);
            }
            return product;
        } catch (Exception e) {
            LOG.error("Failed to fetch product: {}", productId, e);
            return null;
        }
    }

    @Override
    public List<Product> getProductsByCategory(String categoryId) {
        if (!enabled || categoryId == null) return Collections.emptyList();

        String cacheKey = "category:" + categoryId;
        List<Product> cached = getFromCache(cacheKey);
        if (cached != null) return cached;

        try {
            String url = apiBaseUrl + "/categories/" + categoryId + "/products";
            List<Product> products = callApiList(url);
            putInCache(cacheKey, products);
            return products;
        } catch (Exception e) {
            LOG.error("Failed to fetch products for category: {}", categoryId, e);
            return Collections.emptyList();  // Return empty, not null!
        }
    }

    @Override
    public boolean isHealthy() {
        return enabled && apiBaseUrl != null && !apiBaseUrl.isEmpty();
    }

    // ── Private helpers ──

    private <T> T getFromCache(String key) {
        CacheEntry entry = cache.get(key);
        if (entry != null && !entry.isExpired(cacheTtlSeconds)) {
            return (T) entry.getValue();
        }
        return null;
    }

    private void putInCache(String key, Object value) {
        if (cacheTtlSeconds > 0) {
            cache.put(key, new CacheEntry(value));
        }
    }

    private <T> T callApi(String url, Class<T> type) throws Exception {
        // HTTP call implementation
        // Use Apache HttpClient, OkHttp, or AEM's HttpClientBuilderFactory
        return null; // Simplified
    }

    private List<Product> callApiList(String url) throws Exception {
        return Collections.emptyList(); // Simplified
    }

    // ── Inner classes ──

    private static class CacheEntry {
        private final Object value;
        private final long timestamp;

        CacheEntry(Object value) {
            this.value = value;
            this.timestamp = System.currentTimeMillis();
        }

        boolean isExpired(int ttlSeconds) {
            return (System.currentTimeMillis() - timestamp) > (ttlSeconds * 1000L);
        }

        Object getValue() { return value; }
    }
}
```

---

## 🔗 Referencing Services (@Reference)

### In Another OSGi Service
```java
@Component(service = OrderService.class)
public class OrderServiceImpl implements OrderService {

    // Mandatory reference — OrderService WON'T activate if ProductService unavailable
    @Reference
    private ProductService productService;

    // Optional reference — OrderService activates even if service unavailable
    @Reference(cardinality = ReferenceCardinality.OPTIONAL)
    private NotificationService notificationService;

    // Multiple references — gets ALL registered implementations
    @Reference(
        cardinality = ReferenceCardinality.MULTIPLE,
        policy = ReferencePolicy.DYNAMIC,
        policyOption = ReferencePolicyOption.GREEDY
    )
    private volatile List<PaymentGateway> paymentGateways;

    public void processOrder(Order order) {
        // Use ProductService (always available due to mandatory reference)
        Product product = productService.getProduct(order.getProductId());

        // Null-check optional services
        if (notificationService != null) {
            notificationService.sendConfirmation(order.getEmail());
        }
    }
}
```

### In a Sling Model
```java
@Model(adaptables = Resource.class, defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class ProductListModel {

    @OSGiService
    private ProductService productService;

    @ValueMapValue
    private String categoryId;

    private List<Product> products;

    @PostConstruct
    protected void init() {
        if (productService != null && categoryId != null) {
            products = productService.getProductsByCategory(categoryId);
        } else {
            products = Collections.emptyList();
        }
    }

    public List<Product> getProducts() { return products; }
}
```

---

## ⚙️ OSGi Configuration Files

### AEM 6.5 Configuration Storage
```
ui.config/src/main/content/jcr_root/
  apps/mysite/config/                          ← All environments
    com.mysite.core.services.impl.ProductServiceImpl.cfg.json

  apps/mysite/config.author/                  ← Author only
    com.mysite.core.services.impl.ProductServiceImpl.cfg.json

  apps/mysite/config.publish/                 ← Publish only
    com.mysite.core.services.impl.ProductServiceImpl.cfg.json

  apps/mysite/config.author.prod/             ← Author + Production
    com.mysite.core.services.impl.ProductServiceImpl.cfg.json
```

### Configuration File Format (.cfg.json — AEM 6.5.5+ and Cloud)
```json
{
    "apiBaseUrl": "https://api.mysite.com/v1",
    "apiKey": "$[secret:PRODUCT_API_KEY]",
    "cacheTtlSeconds": 300,
    "connectionTimeoutMs": 5000,
    "enabled": true
}
```

> 🔑 In AEM Cloud Service, use `$[secret:ENV_VAR_NAME]` syntax for sensitive values. The actual value is configured in Cloud Manager as an environment variable — it's never stored in the code repository.

### Environment Variables in AEM Cloud
```json
{
    "apiBaseUrl": "$[env:PRODUCT_API_BASE_URL;default=https://api.mysite.com/v1]",
    "apiKey": "$[secret:PRODUCT_API_KEY]"
}
```

---

## 🔄 OSGi Component Lifecycle

```
Bundle Installed
    ↓
Bundle Resolved (dependencies satisfied)
    ↓
Bundle Starting
    ↓
Components Activated (@Activate called)
    ↓
[Running — handling requests, processing data]
    ↓
Config Updated → @Modified called → Component keeps running
    ↓
Bundle Stopping
    ↓
Components Deactivated (@Deactivate called) ← RELEASE RESOURCES HERE
    ↓
Bundle Uninstalled
```

---

## ⚡ @Reference vs @OSGiService

| | `@Reference` (in OSGi Component) | `@OSGiService` (in Sling Model) |
|--|---|---|
| **Used in** | OSGi `@Component` classes | Sling Model classes |
| **Timing** | Injected at component activation | Injected at model instantiation |
| **Optional** | `cardinality = OPTIONAL` | `injectionStrategy = OPTIONAL` |
| **Multiple** | `cardinality = MULTIPLE` | Not directly supported |

---

## 🚨 Common OSGi Mistakes & Fixes

### Mistake 1: Storing ResourceResolver as Field
```java
// ❌ WRONG — ResourceResolver is request-scoped, NOT service-scoped
@Component(service = DataService.class)
public class DataServiceImpl implements DataService {

    @Reference
    private ResourceResolverFactory resolverFactory;

    // NEVER do this — creates memory leaks and stale references
    private ResourceResolver sharedResolver;

    @Activate
    protected void activate() {
        // DON'T create resolver here and store it as field!
        sharedResolver = resolverFactory.getServiceResourceResolver(...);
    }
}

// ✅ CORRECT — Get and close resolver in each method
public class DataServiceImpl implements DataService {

    @Reference
    private ResourceResolverFactory resolverFactory;

    public String readData(String path) {
        Map<String, Object> params = Map.of(SUBSERVICE, "data-reader");
        try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(params)) {
            Resource resource = resolver.getResource(path);
            return resource != null ? resource.getValueMap().get("data", String.class) : null;
        } catch (LoginException e) {
            LOG.error("Cannot get ResourceResolver", e);
            return null;
        }
    }
}
```

### Mistake 2: Not Handling Missing @Reference Service
```java
// ❌ WRONG — Component won't start if PaymentService unavailable
@Reference
private PaymentService paymentService;  // Mandatory by default

// ✅ CORRECT — Make optional if not critical
@Reference(cardinality = ReferenceCardinality.OPTIONAL)
private PaymentService paymentService;

// Then always null-check:
public void processPayment(Order order) {
    if (paymentService == null) {
        LOG.warn("PaymentService not available — skipping payment processing");
        return;
    }
    paymentService.charge(order);
}
```

### Mistake 3: Not Unregistering in @Deactivate
```java
// ❌ WRONG — Event listener never unregistered = memory leak
@Component
public class ContentEventHandler {
    @Reference
    private EventAdmin eventAdmin;

    private EventHandler handler;

    @Activate
    protected void activate() {
        handler = event -> LOG.info("Event received: {}", event.getTopic());
        // Registered but never unregistered!
    }

    // ✅ CORRECT
    @Deactivate
    protected void deactivate() {
        if (handler != null) {
            // Unregister listener, cancel timers, close connections
        }
    }
}
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is the difference between an OSGi Component and an OSGi Service?**

> **Answer:** An **OSGi Component** is any Java class annotated with `@Component` and managed by the OSGi framework (lifecycle: activate/deactivate). An **OSGi Service** is a Component that ALSO registers itself in the OSGi Service Registry under a specific interface: `@Component(service = MyInterface.class)`. Services can be injected into other components via `@Reference`. All Services are Components, but not all Components are Services (e.g., an event listener might be a Component but not register as a Service).

**Q2. When is `@Activate` called? What should you do there?**

> **Answer:** `@Activate` is called when the OSGi bundle/component starts — once at startup (or after a config change if `@Modified` is not defined). Use it to: read OSGi configuration values (`config.apiUrl()`), initialize connections, set up caches, register event listeners. Do NOT: perform JCR operations (no ResourceResolver at this point), or do heavy work that should be per-request.

**Q3. What is `@Modified` and when would you use it?**

> **Answer:** `@Modified` is called when the component's OSGi configuration changes (via Felix Console or OSGi config files) WITHOUT fully deactivating and reactivating the component. Use it when you want to reload configuration dynamically without a service restart. The common pattern is to call `activate(config)` from `@Modified` since they both need to process the new config.

**Q4. What is `@Deactivate` and why is it critical?**

> **Answer:** `@Deactivate` is called when the bundle/component is stopping. It's **critical** for resource cleanup: close HTTP clients, cancel scheduled tasks, clear caches, unregister listeners. Failing to release resources in `@Deactivate` causes memory leaks that accumulate over time and crash the JVM.

**Q5. What is `@Reference` and how does it work with `cardinality`?**

> **Answer:** `@Reference` injects another registered OSGi service. `cardinality` controls what happens if the service is unavailable:
> - `MANDATORY` (default): Component won't activate without it
> - `OPTIONAL`: Component activates even if service is null
> - `AT_LEAST_ONE`: Requires at least one implementation
> - `MULTIPLE`: Gets all registered implementations as a `List`

**Q6. How do you make an OSGi service configurable per environment (dev/stage/prod)?**

> **Answer:** Use `@ObjectClassDefinition` for the config interface and `@Designate(ocd = MyConfig.class)` on the component. Store config JSON files in run-mode-specific folders: `config/` (all), `config.author/`, `config.publish/`, `config.prod/`. The OSGi framework applies the most specific matching config at startup. In AEM Cloud, use `$[env:VAR_NAME]` and `$[secret:VAR_NAME]` references in config files, with actual values set in Cloud Manager.

**Q7. Why should OSGi services code to interfaces, not implementations?**

> **Answer:** Three reasons:
> 1. **Testability:** In unit tests, mock the interface without the real implementation (no API calls, no JCR access)
> 2. **Multiple implementations:** Different implementations for prod (real API) vs dev (mock data) vs test (cached)
> 3. **OSGi best practice:** Register and reference services by interface — the service registry is interface-based

---

## ✅ Best Practices

1. **Always code to interfaces** — register `service = MyInterface.class`, not the Impl class
2. **Use `@Modified`** to reload config without component restart (avoid unnecessary downtime)
3. **Always cleanup** in `@Deactivate` — close connections, cancel timers, clear caches
4. **Use environment-specific config folders** (`config.author`, `config.publish`, `config.prod`)
5. **Never store `ResourceResolver` as a service field** — always get-and-close per-operation
6. **Use `ReferenceCardinality.OPTIONAL`** for non-critical services
7. **Log at appropriate levels**: DEBUG for operational data, INFO for lifecycle events, WARN for non-critical issues, ERROR for failures
8. **In AEM Cloud**: Use `$[secret:VAR]` for API keys — never hardcode credentials

---

## 🛠️ Hands-on Practice

**Day 12**: Create `LoggingService` with `@Activate`, `@Modified`, `@Deactivate`. Log config values at each lifecycle event.

**Day 13**: Create `ContentApiService` interface + `ContentApiServiceImpl` with configurable base URL, API key, timeout. Store config in `config.author/` and `config.publish/` with different URLs.

**Day 14**: Inject `ContentApiService` into a Sling Model using `@OSGiService`. Call it in `@PostConstruct`, handle null gracefully.

**Day 15**: Create `ReportingService` that `@Reference`s both `ContentApiService` and `UserService`. Make `UserService` reference `OPTIONAL`.
