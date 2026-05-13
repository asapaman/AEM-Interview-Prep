# AEM Testing — Unit & Integration Tests
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 Why AEM Testing Matters

Testing AEM code is fundamentally different from testing plain Java because:
- AEM code depends on the JCR repository, OSGi services, Sling resolvers, and WCM APIs
- You can't run these in a standard JUnit test without mocking the entire AEM runtime
- **AEM Mocks (io.wcm)** solves this — it provides an in-memory AEM environment for unit tests

> In interviews, the ability to write testable code and actual unit tests with `AemContext` is a strong signal of a 4 YOE candidate vs a 2 YOE candidate.

---

## 🧰 Testing Stack

| Library | Purpose | Version |
|---------|---------|---------|
| **JUnit 5** | Test framework | 5.x |
| **AEM Mocks (io.wcm)** | In-memory AEM environment | `io.wcm.testing.aem-mock.junit5` |
| **Mockito** | Mocking external dependencies | 4.x |
| **AssertJ** | Fluent assertions (optional) | 3.x |
| **Sling Mocks** | Underlying Sling mock infrastructure | included in AEM Mocks |

### Maven Dependencies

```xml
<!-- pom.xml — test dependencies -->
<dependencies>
    <!-- AEM Mocks — the core of AEM unit testing -->
    <dependency>
        <groupId>io.wcm</groupId>
        <artifactId>io.wcm.testing.aem-mock.junit5</artifactId>
        <version>5.3.0</version>
        <scope>test</scope>
    </dependency>

    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.9.0</version>
        <scope>test</scope>
    </dependency>

    <!-- Mockito -->
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <version>4.x.x</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <version>4.x.x</version>
        <scope>test</scope>
    </dependency>

    <!-- AssertJ (optional but nicer assertions) -->
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.23.1</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## 🏗️ AemContext — The Foundation

`AemContext` is the central object in AEM unit tests. It provides:
- An in-memory JCR repository (Oak MemoryNodeStore)
- A mock ResourceResolver
- A mock OSGi component context
- Sling Model registration and adaptation
- Request/Response mocks

```java
// Basic setup — two ways:

// Method 1: @ExtendWith (recommended)
@ExtendWith(AemContextExtension.class)
class MyModelTest {
    private final AemContext context = new AemContext(ResourceResolverType.JCR_MOCK);
    // ResourceResolverType options:
    // JCR_MOCK    → Fast, in-memory, no real JCR. Enough for most tests.
    // JCR_OAK     → Full Oak repository (slower, for complex query/index tests)
    // RESOURCERESOLVER_MOCK → Fastest, very limited (no JCR at all)
}

// Method 2: @Rule (JUnit 4, avoid in new code)
// @Rule public AemContext context = new AemContext();
```

---

## 💻 Testing Sling Models — Complete Examples

### Example 1: Basic Sling Model Test

The model under test:
```java
// HeroModel.java
@Model(adaptables = Resource.class, defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class HeroModel {
    @ValueMapValue private String title;
    @ValueMapValue private String ctaLink;
    @ValueMapValue @Default(values = "h2") private String headingLevel;

    @PostConstruct
    protected void init() {
        if (ctaLink != null && ctaLink.startsWith("/content/")) {
            ctaLink = ctaLink + ".html";
        }
    }

    public String getTitle() { return title; }
    public String getCtaLink() { return ctaLink; }
    public String getHeadingLevel() { return headingLevel; }
    public boolean isConfigured() { return title != null; }
}
```

The test:
```java
package com.mysite.core.models;

import io.wcm.testing.mock.aem.junit5.AemContext;
import io.wcm.testing.mock.aem.junit5.AemContextExtension;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.apache.sling.api.resource.Resource;

import static org.junit.jupiter.api.Assertions.*;

@ExtendWith(AemContextExtension.class)  // Enable AEM Mocks
class HeroModelTest {

    // AemContext is injected by AemContextExtension
    private final AemContext context = new AemContext();

    @BeforeEach
    void setUp() {
        // Register the model class so AemContext knows about it
        context.addModelsForClasses(HeroModel.class);
    }

    @Test
    void testTitleAndCtaLink() {
        // Arrange: Create a fake JCR resource with properties
        context.currentResource(context.create().resource(
            "/content/mysite/en/home/jcr:content/hero",
            "title",      "Welcome to My Site",
            "ctaLink",    "/content/mysite/en/products",
            "ctaLabel",   "Shop Now"
        ));

        // Act: Adapt the resource to the model (same as AEM does in production)
        HeroModel model = context.request().adaptTo(HeroModel.class);

        // Assert
        assertNotNull(model);
        assertEquals("Welcome to My Site", model.getTitle());
        assertEquals("/content/mysite/en/products.html", model.getCtaLink());  // .html appended
        assertEquals("h2", model.getHeadingLevel());  // Default value
        assertTrue(model.isConfigured());
    }

    @Test
    void testDefaultHeadingLevel() {
        context.currentResource(context.create().resource(
            "/content/mysite/en/home/jcr:content/hero",
            "title", "Test Title"
            // No headingLevel — should use default "h2"
        ));

        HeroModel model = context.request().adaptTo(HeroModel.class);

        assertNotNull(model);
        assertEquals("h2", model.getHeadingLevel());
    }

    @Test
    void testExternalCtaLink() {
        context.currentResource(context.create().resource(
            "/content/mysite/en/home/jcr:content/hero",
            "title",   "External Link Test",
            "ctaLink", "https://external.com/path"  // External URL — no .html should be added
        ));

        HeroModel model = context.request().adaptTo(HeroModel.class);

        assertNotNull(model);
        assertEquals("https://external.com/path", model.getCtaLink());  // Unchanged
    }

    @Test
    void testNotConfiguredWhenNoTitle() {
        context.currentResource(context.create().resource(
            "/content/mysite/en/home/jcr:content/hero"
            // No title
        ));

        HeroModel model = context.request().adaptTo(HeroModel.class);

        assertNotNull(model);
        assertFalse(model.isConfigured());
        assertNull(model.getTitle());
    }
}
```

---

### Example 2: Testing with OSGi Service Mocks

```java
package com.mysite.core.services;

import io.wcm.testing.mock.aem.junit5.AemContext;
import io.wcm.testing.mock.aem.junit5.AemContextExtension;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

@ExtendWith({AemContextExtension.class, MockitoExtension.class})
class ProductServiceImplTest {

    private final AemContext context = new AemContext();

    // ── External dependency we want to mock ──
    @Mock
    private ExternalApiClient apiClient;

    private ProductServiceImpl productService;

    @BeforeEach
    void setUp() {
        // Register the mock as an OSGi service in AemContext
        context.registerService(ExternalApiClient.class, apiClient);

        // Register the service under test (it will get apiClient injected via @Reference)
        productService = context.registerInjectActivateService(
            new ProductServiceImpl(),
            // OSGi config properties for @Activate
            "apiBaseUrl",     "https://api.mysite.com",
            "connectionTimeout", "5000"
        );
    }

    @Test
    void testGetProduct_returnsProductFromApi() {
        // Arrange: mock the external API response
        when(apiClient.fetchProduct("LP-2024-001"))
            .thenReturn(new ApiProduct("LP-2024-001", "Laptop Pro 2024", 1299.99));

        // Act
        Product product = productService.getProduct("LP-2024-001");

        // Assert
        assertNotNull(product);
        assertEquals("LP-2024-001", product.getSku());
        assertEquals("Laptop Pro 2024", product.getName());
        assertEquals(1299.99, product.getPrice(), 0.01);

        // Verify the API was called exactly once
        verify(apiClient, times(1)).fetchProduct("LP-2024-001");
    }

    @Test
    void testGetProduct_returnsNullOnApiFailure() {
        // Arrange: simulate API exception
        when(apiClient.fetchProduct(anyString()))
            .thenThrow(new RuntimeException("API unavailable"));

        // Act
        Product product = productService.getProduct("LP-2024-001");

        // Assert: service should handle exception gracefully, return null
        assertNull(product);
    }

    @Test
    void testGetProduct_usesCacheOnSecondCall() {
        when(apiClient.fetchProduct("LP-2024-001"))
            .thenReturn(new ApiProduct("LP-2024-001", "Laptop Pro 2024", 1299.99));

        // Call twice
        productService.getProduct("LP-2024-001");
        productService.getProduct("LP-2024-001");

        // Verify API was only called ONCE (second call used cache)
        verify(apiClient, times(1)).fetchProduct("LP-2024-001");
    }
}
```

---

### Example 3: Loading Test Fixtures from JSON

For complex tests, create JSON fixture files instead of building resources programmatically:

```json
// src/test/resources/com/mysite/core/models/HeroModelTest/hero-resource.json
{
  "jcr:primaryType": "nt:unstructured",
  "sling:resourceType": "mysite/components/content/hero",
  "title": "Welcome to My Site",
  "ctaLink": "/content/mysite/en/products",
  "ctaLabel": "Shop Now",
  "headingLevel": "h1"
}
```

```java
@Test
void testWithJsonFixture() {
    // Load JSON fixture into AemContext
    context.load().json(
        "/com/mysite/core/models/HeroModelTest/hero-resource.json",
        "/content/test/hero"
    );

    context.currentResource("/content/test/hero");
    HeroModel model = context.request().adaptTo(HeroModel.class);

    assertEquals("Welcome to My Site", model.getTitle());
    assertEquals("h1", model.getHeadingLevel());
}
```

---

### Example 4: Testing a Sling Model with Page Context

```java
@Test
void testModelWithPageContext() {
    // Load a full page structure
    context.load().json("/page-structure.json", "/content/mysite/en/home");

    // Set the current page
    context.currentPage("/content/mysite/en/home");
    context.currentResource("/content/mysite/en/home/jcr:content/root/hero");

    // Now @ScriptVariable Page currentPage will be injected
    HeroModel model = context.request().adaptTo(HeroModel.class);

    assertNotNull(model);
    // If title is null in the component, model should fall back to page title
    assertEquals("Home", model.getTitle());
}
```

---

### Example 5: Testing an OSGi Service with `@Activate`

```java
@Test
void testServiceActivation() {
    // Register service with specific OSGi configuration values
    ApiServiceImpl service = context.registerInjectActivateService(
        new ApiServiceImpl(),
        "apiUrl",         "https://custom-api.mysite.com",
        "timeout",        3000,
        "enabled",        true,
        "allowedPaths",   new String[]{"/content/mysite", "/conf/mysite"}
    );

    // The service's @Activate was called with these values
    assertTrue(service.isEnabled());
    assertEquals("https://custom-api.mysite.com", service.getApiUrl());
}

@Test
void testServiceDisabled() {
    ApiServiceImpl service = context.registerInjectActivateService(
        new ApiServiceImpl(),
        "enabled", false
    );

    assertFalse(service.isEnabled());
    // Verify disabled service returns empty/null results
    assertNull(service.fetchData("/content/mysite/en/home"));
}
```

---

### Example 6: Testing a Sling Servlet

```java
@Test
void testServlet_returnsJson() throws Exception {
    // Set up the resource the servlet is registered against
    context.currentResource(context.create().resource(
        "/content/mysite/en/search",
        "sling:resourceType", "mysite/components/content/search"
    ));

    // Set request parameters
    context.request().addRequestParameter("q", "laptop");

    // Instantiate and call the servlet
    SearchServlet servlet = new SearchServlet();
    context.registerInjectActivateService(servlet);

    MockSlingHttpServletResponse response = context.response();
    servlet.doGet(context.request(), response);

    // Assert response
    assertEquals(200, response.getStatus());
    assertEquals("application/json;charset=UTF-8", response.getContentType());

    String json = response.getOutputAsString();
    assertTrue(json.contains("\"results\""));
}
```

---

## 📋 AemContext Quick Reference

```java
// CREATE resources
context.create().resource("/path", "prop1", "val1", "prop2", "val2");
context.create().page("/content/mysite/en/home", "/conf/mysite/templates/page");
context.create().asset("/content/dam/img.jpg", "/src/test/resources/img.jpg", "image/jpeg");

// SET current request context
context.currentResource("/path/to/resource");
context.currentPage("/content/mysite/en/home");
context.request().addRequestParameter("key", "value");
context.request().addHeader("X-Custom-Header", "value");

// LOAD JSON fixtures
context.load().json("/fixture.json", "/content/path");
context.load().binaryFile("/image.jpg", "/content/dam/image.jpg");

// REGISTER services and models
context.addModelsForClasses(MyModel.class);
context.addModelsForPackage("com.mysite.core.models");
context.registerService(MyService.class, mockServiceInstance);
context.registerInjectActivateService(new MyServiceImpl(), "prop", "value");

// ADAPT
MyModel model = context.request().adaptTo(MyModel.class);
MyModel model = context.currentResource().adaptTo(MyModel.class);
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is `AemContext` and why is it needed for AEM unit testing?**

> **Answer:** `AemContext` from the io.wcm testing library provides a complete in-memory AEM environment for unit tests — without requiring a running AEM instance. It includes an in-memory JCR repository (Oak), mock ResourceResolver, mock Sling request/response, and OSGi component context. Without it, AEM code (which depends on JCR, Sling, and OSGi) can't be tested in isolation.

**Q2. How do you test a Sling Model that uses `@ScriptVariable Page currentPage`?**

> **Answer:** Use `context.currentPage("/path/to/page")` before adapting the model. AemContext registers `currentPage` as a script variable that gets injected by the `@ScriptVariable` annotator. You also need to call `context.currentResource()` to set the resource being adapted.

**Q3. How do you mock an OSGi service dependency in a Sling Model test?**

> **Answer:** Create a Mockito mock of the service interface, register it via `context.registerService(ServiceInterface.class, mockInstance)`, then register the Sling Model class. When the model is adapted, AemContext injects the mock via `@OSGiService`. Use `when(...).thenReturn(...)` to define mock behavior.

**Q4. What is `registerInjectActivateService` and when do you use it?**

> **Answer:** It creates an instance of an OSGi component, injects all its `@Reference` dependencies (from previously registered services), and calls its `@Activate` method with the provided configuration properties. Use it when testing OSGi service implementations — it simulates the full OSGi activation lifecycle without a real OSGi container.

**Q5. What is the difference between `JCR_MOCK` and `JCR_OAK` AemContext types?**

> **Answer:** `JCR_MOCK` is an in-memory JCR mock that's very fast and sufficient for most tests — it supports basic resource creation, property reading, and Sling model adaptation. `JCR_OAK` spins up a full embedded Oak repository — it's slower but required for tests that use JCR queries (Query Builder, XPath, JCR-SQL2), JCR observation, Oak indexes, or version history. Use `JCR_MOCK` by default; switch to `JCR_OAK` only when queries are needed.

---

## ✅ Best Practices

1. **Test all @PostConstruct logic** — this is where most business logic lives
2. **Test null/edge cases** — what happens when `title` is null? When service is unavailable?
3. **Use JSON fixtures** for complex resource structures — easier to maintain than programmatic creation
4. **Mock external dependencies** (HTTP clients, databases) — unit tests must be fast and isolated
5. **Test one behavior per test** — each `@Test` method tests ONE specific scenario
6. **Name tests clearly** — `testTitle_whenNoTitleSet_returnsFallbackFromPage()` beats `testModel()`
7. **Aim for 80%+ coverage** on Sling Models and OSGi Services
8. **Test with Cloud Manager** — SonarQube quality gate blocks builds below coverage threshold

---

## 🛠️ Hands-on Practice

1. Write a unit test for `HeroModel` covering: normal case, no title (null), external CTA link, internal CTA link
2. Write a unit test for an OSGi service that has an `@Reference` to an `ExternalApiClient` — mock the client
3. Write a test that loads a JSON fixture and validates model properties
4. Write a test for a servlet that verifies the response status and JSON content
5. Configure SonarQube code coverage gate: find the Jacoco configuration in your project's `pom.xml`
