# Day 12, 13, 14, 15 — OSGi & Services
**Difficulty:** Hard | **4 YOE Focus**

---

## 📖 Topic Explanation

OSGi (Open Services Gateway initiative) is the module system underlying AEM. It manages the lifecycle of Java bundles (JARs with special metadata) and the registration/lookup of services.

### OSGi Concepts
| Concept | Description |
|---------|------------|
| **Bundle** | JAR file with OSGi manifest. Unit of deployment. |
| **Component** | Java class managed by OSGi (annotated with `@Component`) |
| **Service** | Interface registered in OSGi service registry |
| **Reference** | A dependency on another service (`@Reference`) |
| **Activate/Deactivate** | Lifecycle callbacks |

---

## 🏷️ Core OSGi Annotations

```java
package com.mysite.core.services.impl;

import org.osgi.service.component.annotations.*;
import org.osgi.service.metatype.annotations.Designate;
import org.osgi.service.metatype.annotations.ObjectClassDefinition;
import org.osgi.service.metatype.annotations.AttributeDefinition;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

// @ObjectClassDefinition defines configurable properties (OSGi Config)
@ObjectClassDefinition(name = "My Custom Service Config")
@interface MyServiceConfig {
    @AttributeDefinition(name = "API Endpoint", description = "URL of the external API")
    String apiEndpoint() default "https://api.example.com";

    @AttributeDefinition(name = "Cache TTL (seconds)")
    int cacheTtl() default 300;

    @AttributeDefinition(name = "Enabled")
    boolean enabled() default true;
}

// @Component marks this as an OSGi-managed component
// @Service registers it in the OSGi service registry under the interface
@Component(service = MyCustomService.class, immediate = true)
@Designate(ocd = MyServiceConfig.class)
public class MyCustomServiceImpl implements MyCustomService {

    private static final Logger LOG = LoggerFactory.getLogger(MyCustomServiceImpl.class);

    private String apiEndpoint;
    private int cacheTtl;
    private boolean enabled;

    // @Activate runs when the component is activated (bundle starts or config changes)
    @Activate
    protected void activate(MyServiceConfig config) {
        this.apiEndpoint = config.apiEndpoint();
        this.cacheTtl = config.cacheTtl();
        this.enabled = config.enabled();
        LOG.info("MyCustomService activated. Endpoint: {}", apiEndpoint);
    }

    // @Modified runs when OSGi config is updated (without full deactivate/activate)
    @Modified
    protected void modified(MyServiceConfig config) {
        activate(config);  // Re-use activate logic
    }

    // @Deactivate runs when bundle/component is stopped
    @Deactivate
    protected void deactivate() {
        LOG.info("MyCustomService deactivated");
    }

    @Override
    public String fetchData(String id) {
        if (!enabled) return null;
        // Business logic using apiEndpoint
        return apiEndpoint + "/data/" + id;
    }
}
```

---

## 🔗 Service Interface

```java
// Always define a separate interface!
public interface MyCustomService {
    String fetchData(String id);
    List<Product> getProducts(String category);
}
```

---

## 📥 Referencing Services

### In another OSGi Service
```java
@Component(service = AnotherService.class)
public class AnotherServiceImpl implements AnotherService {

    // @Reference injects another registered OSGi service
    @Reference
    private MyCustomService myCustomService;

    // Optional reference (won't fail if service unavailable)
    @Reference(cardinality = ReferenceCardinality.OPTIONAL)
    private OptionalService optionalService;

    // Multiple services (all implementations)
    @Reference(cardinality = ReferenceCardinality.MULTIPLE,
               policy = ReferencePolicy.DYNAMIC)
    private volatile List<SomeService> someServices;

    public void doWork() {
        String data = myCustomService.fetchData("123");
    }
}
```

### In a Sling Model
```java
@Model(adaptables = Resource.class, defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class MyComponentModel {

    @OSGiService
    private MyCustomService myCustomService;

    @PostConstruct
    protected void init() {
        String result = myCustomService.fetchData("product-123");
    }
}
```

---

## 🔑 OSGi Service vs OSGi Component

| | OSGi Component | OSGi Service |
|--|---------------|-------------|
| Annotation | `@Component` | `@Component(service=...)` |
| Registered? | Managed but NOT in service registry | Registered in service registry |
| Injectable? | ❌ Cannot be `@Reference`d | ✅ Can be `@Reference`d |
| Use case | Listeners, Schedulers | Shared business logic |

---

## 📝 OSGi Configuration (OSGI Config Files)

OSGi configs are stored as JSON/XML in the code repo and deployed as part of the package:

```
ui.config/src/main/content/jcr_root/apps/mysite/osgiconfig/
  ├── config/                                        ← All environments
  │     └── com.mysite.core.services.impl.MyCustomServiceImpl.cfg.json
  ├── config.author/                                 ← Author-only
  │     └── com.mysite.core.services.impl.MyCustomServiceImpl.cfg.json
  └── config.publish/                                ← Publish-only
        └── com.mysite.core.services.impl.MyCustomServiceImpl.cfg.json
```

```json
{
  "apiEndpoint": "https://api.production.com",
  "cacheTtl": 600,
  "enabled": true
}
```

---

## ❓ Interview Questions & Answers

**Q1. List and explain commonly used OSGi annotations.**
> - `@Component`: Marks class as OSGi-managed. Handles lifecycle.
> - `@Activate`: Called when component starts. Read config here.
> - `@Deactivate`: Called when component stops. Release resources.
> - `@Modified`: Called when OSGi config changes (without restart).
> - `@Reference`: Injects another OSGi service.
> - `@Designate`: Links component to a configuration class (`@ObjectClassDefinition`).

**Q2. What is the difference between an OSGi Service and an OSGi Component?**
> A **Component** is any `@Component` class managed by OSGi. A **Service** is a Component that also registers itself in the OSGi Service Registry (`service=MyInterface.class`), making it injectable via `@Reference` or `@OSGiService`. All Services are Components, but not all Components are Services.

**Q3. How do you call one OSGi service from another?**
> Use `@Reference` annotation:
```java
@Reference
private SomeService someService;
```
> OSGi's dependency injection automatically injects the registered implementation. If multiple implementations exist, use `target` filter or ranking.

**Q4. How do you make an OSGi service configurable per environment?**
> Use `@ObjectClassDefinition` + `@Designate` for the config class. Store JSON config files in run-mode folders (`config.author/`, `config.publish/`, `config.prod/`). OSGi Config Manager applies the appropriate config based on active run modes.

**Q5. What happens if an `@Reference`d service is unavailable?**
> By default (`cardinality = MANDATORY`), the component WON'T activate if the referenced service is unavailable. Use `cardinality = ReferenceCardinality.OPTIONAL` if the dependency is optional.

**Q6. What is `immediate = true` in `@Component`?**
> By default, OSGi activates a component lazily (when first requested). `immediate = true` forces activation immediately when the bundle starts. Use for components that need to register listeners at startup (e.g., event handlers, schedulers).

**Q7. How do you avoid memory leaks in OSGi services?**
> - Close all `ResourceResolver` instances in `finally` blocks
> - Unregister listeners in `@Deactivate`
> - For schedulers, cancel the scheduled job in `@Deactivate`
> - Use `volatile` for dynamically referenced service lists
> - Never store request-scoped objects in service fields

---

## ✅ Best Practices

- Always code to **interfaces** — keep impl in separate class
- Use `@Modified` to reload config without restarting bundle
- Use run-mode specific config folders (`config.author`, `config.publish`, `config.prod`)
- Close `ResourceResolver` in `finally` block or use try-with-resources
- Use `LoggerFactory.getLogger(ClassName.class)` — always include class for context
- Register services under their **interface**, not implementation class
- Use `@Reference(cardinality = OPTIONAL)` for non-critical dependencies

---

## 🛠️ Hands-on Tasks

**Day 12**: Create `LoggingService` with `@Activate`, `@Deactivate`, `@Modified`
**Day 13**: Create `ProductService` interface + `ProductServiceImpl` with configurable API URL
**Day 14**: Inject `ProductService` into a Sling Model using `@OSGiService`
**Day 15**: Create `ReportService` that `@Reference`s both `ProductService` and `UserService`
