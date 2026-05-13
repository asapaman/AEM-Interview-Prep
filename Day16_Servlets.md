# Day 16–19 — Servlets (Complete Guide)
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What is an AEM Servlet?

An AEM Servlet is a Java class that handles HTTP requests. Unlike traditional web servlets (registered at fixed URL paths), AEM Servlets can be registered in two ways:
1. **By Resource Type** (preferred) — triggered when a specific content type + selector/extension is requested
2. **By Path** (use with caution) — registered at a fixed URL path like `/bin/mysite/api`

AEM Servlets extend either:
- `SlingSafeMethodsServlet` — handles only safe (read-only) methods: GET, HEAD
- `SlingAllMethodsServlet` — handles all methods: GET, HEAD, POST, PUT, DELETE

---

## 🔄 Servlet Registration Methods

### Method 1: Resource-Type Based (✅ Preferred)

```java
@Component(service = Servlet.class)
@SlingServletResourceTypes(
    resourceTypes = "mysite/components/content/product",  // Component resource type
    selectors = "data",                                    // URL selector
    extensions = "json",                                   // URL extension
    methods = HttpConstants.METHOD_GET                     // HTTP method
)
public class ProductDataServlet extends SlingSafeMethodsServlet {
    // Handles: GET /content/mysite/products/laptop.data.json
}
```

**How the URL maps:**
```
/content/mysite/products/laptop.data.json
                                 ↑    ↑  ↑
                                 │    │  └── extension: json
                                 │    └───── selector: data
                                 └────────── resource: /content/mysite/products/laptop
                                             (which has sling:resourceType="mysite/components/content/product")
```

### Method 2: Path Based (⚠️ Use With Caution)

```java
@Component(service = Servlet.class)
@SlingServletPaths("/bin/mysite/api/health")
public class HealthCheckServlet extends SlingSafeMethodsServlet {
    // Handles: GET /bin/mysite/api/health
}
```

**Why caution?**
- Must explicitly configure in Dispatcher to allow `/bin/` paths
- Not tied to content → open to unauthorized access
- Security risk — runs without content-level permission checks

### Comparison

| Aspect | Resource-Type | Path-Based |
|--------|--------------|-----------|
| Security | ✅ Inherits content ACLs | ⚠️ Open to all |
| Dispatcher config | Minimal | Requires explicit `/bin/` allow |
| URL structure | Natural (follows content) | Arbitrary |
| Best for | Component data APIs | Admin APIs, health checks |

---

## 📥 SlingSafeMethodsServlet — GET Requests

```java
package com.mysite.core.servlets;

import com.day.cq.wcm.api.Page;
import com.day.cq.wcm.api.PageManager;
import com.google.gson.Gson;
import com.google.gson.GsonBuilder;
import com.mysite.core.services.ProductService;
import org.apache.sling.api.SlingHttpServletRequest;
import org.apache.sling.api.SlingHttpServletResponse;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.api.servlets.HttpConstants;
import org.apache.sling.api.servlets.SlingSafeMethodsServlet;
import org.apache.sling.servlets.annotations.SlingServletResourceTypes;
import org.osgi.service.component.annotations.Component;
import org.osgi.service.component.annotations.Reference;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.servlet.Servlet;
import javax.servlet.ServletException;
import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

/**
 * Servlet that returns product data as JSON.
 *
 * URL: /content/mysite/products/laptop.data.json
 * ↑ Resource with resourceType="mysite/components/content/product"
 *                              selector="data", extension="json"
 */
@Component(service = Servlet.class)
@SlingServletResourceTypes(
    resourceTypes = "mysite/components/content/product",
    selectors = "data",
    extensions = "json",
    methods = HttpConstants.METHOD_GET
)
public class ProductDataServlet extends SlingSafeMethodsServlet {

    private static final long serialVersionUID = 1L;
    private static final Logger LOG = LoggerFactory.getLogger(ProductDataServlet.class);

    private static final Gson GSON = new GsonBuilder().setPrettyPrinting().create();

    @Reference
    private ProductService productService;

    @Override
    protected void doGet(
            SlingHttpServletRequest request,
            SlingHttpServletResponse response)
            throws ServletException, IOException {

        // 1. SET RESPONSE HEADERS (always do this first)
        response.setContentType("application/json");
        response.setCharacterEncoding("UTF-8");

        // 2. ADD CACHE HEADERS (important for CDN/Dispatcher)
        response.setHeader("Cache-Control", "max-age=300, public");

        // 3. GET CURRENT RESOURCE
        Resource resource = request.getResource();

        // 4. READ PARAMETERS
        String productId = request.getParameter("id");
        // Also readable from resource properties:
        // String productId = resource.getValueMap().get("productId", String.class);

        // 5. VALIDATE INPUT
        if (productId == null || productId.trim().isEmpty()) {
            sendError(response, 400, "Missing required parameter: id");
            return;
        }

        // 6. BUSINESS LOGIC
        try {
            Product product = productService.getProduct(productId);

            if (product == null) {
                sendError(response, 404, "Product not found: " + productId);
                return;
            }

            // 7. BUILD RESPONSE
            Map<String, Object> responseData = new HashMap<>();
            responseData.put("id", product.getId());
            responseData.put("name", product.getName());
            responseData.put("price", product.getPrice());
            responseData.put("available", product.isInStock());

            // 8. WRITE RESPONSE
            response.setStatus(200);
            response.getWriter().write(GSON.toJson(responseData));

        } catch (Exception e) {
            LOG.error("Error fetching product data for id: {}", productId, e);
            sendError(response, 500, "Internal server error");
        }
    }

    private void sendError(SlingHttpServletResponse response, int status, String message)
            throws IOException {
        response.setStatus(status);
        Map<String, String> error = new HashMap<>();
        error.put("error", message);
        error.put("status", String.valueOf(status));
        response.getWriter().write(GSON.toJson(error));
    }
}
```

---

## 📤 SlingAllMethodsServlet — POST/PUT/DELETE

```java
package com.mysite.core.servlets;

import com.google.gson.*;
import com.mysite.core.services.FormService;
import org.apache.sling.api.SlingHttpServletRequest;
import org.apache.sling.api.SlingHttpServletResponse;
import org.apache.sling.api.servlets.SlingAllMethodsServlet;
import org.apache.sling.servlets.annotations.SlingServletPaths;
import org.osgi.service.component.annotations.Component;
import org.osgi.service.component.annotations.Reference;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.servlet.Servlet;
import javax.servlet.ServletException;
import java.io.BufferedReader;
import java.io.IOException;

/**
 * Form submission servlet.
 *
 * Handles:
 *   GET  /bin/mysite/forms/contact → Return form metadata/state
 *   POST /bin/mysite/forms/contact → Process form submission
 */
@Component(service = Servlet.class)
@SlingServletPaths("/bin/mysite/forms/contact")
public class ContactFormServlet extends SlingAllMethodsServlet {

    private static final long serialVersionUID = 1L;
    private static final Logger LOG = LoggerFactory.getLogger(ContactFormServlet.class);
    private static final Gson GSON = new Gson();

    @Reference
    private FormService formService;

    @Override
    protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response)
            throws ServletException, IOException {

        response.setContentType("application/json");
        response.setCharacterEncoding("UTF-8");

        // Return form configuration/state
        JsonObject formMeta = new JsonObject();
        formMeta.addProperty("status", "ready");
        formMeta.addProperty("fields", "[name, email, message]");

        response.getWriter().write(GSON.toJson(formMeta));
    }

    @Override
    protected void doPost(SlingHttpServletRequest request, SlingHttpServletResponse response)
            throws ServletException, IOException {

        response.setContentType("application/json");
        response.setCharacterEncoding("UTF-8");

        // 1. VALIDATE CSRF TOKEN (important for security!)
        // AEM's CSRF filter handles this if properly configured
        // Or manually: CsrfUtil.isValidRequest(request)

        // 2. READ REQUEST BODY
        JsonObject body = null;
        try {
            StringBuilder sb = new StringBuilder();
            try (BufferedReader reader = request.getReader()) {
                String line;
                while ((line = reader.readLine()) != null) {
                    sb.append(line);
                }
            }
            body = JsonParser.parseString(sb.toString()).getAsJsonObject();
        } catch (Exception e) {
            sendJsonError(response, 400, "Invalid JSON body");
            return;
        }

        // 3. VALIDATE REQUIRED FIELDS
        String name = getStringField(body, "name");
        String email = getStringField(body, "email");
        String message = getStringField(body, "message");

        if (name == null || name.isEmpty()) {
            sendJsonError(response, 400, "Name is required");
            return;
        }
        if (email == null || !email.matches("^[A-Za-z0-9+_.-]+@(.+)$")) {
            sendJsonError(response, 400, "Valid email is required");
            return;
        }
        if (message == null || message.isEmpty()) {
            sendJsonError(response, 400, "Message is required");
            return;
        }

        // 4. PROCESS SUBMISSION
        try {
            boolean success = formService.submitContactForm(name, email, message);

            if (success) {
                JsonObject successResponse = new JsonObject();
                successResponse.addProperty("success", true);
                successResponse.addProperty("message", "Thank you! We'll respond within 24 hours.");
                response.setStatus(200);
                response.getWriter().write(GSON.toJson(successResponse));
            } else {
                sendJsonError(response, 500, "Failed to process submission");
            }

        } catch (Exception e) {
            LOG.error("Error processing contact form submission", e);
            sendJsonError(response, 500, "Internal server error");
        }
    }

    private String getStringField(JsonObject body, String field) {
        JsonElement element = body.get(field);
        return (element != null && !element.isJsonNull()) ? element.getAsString().trim() : null;
    }

    private void sendJsonError(SlingHttpServletResponse response, int status, String message)
            throws IOException {
        response.setStatus(status);
        JsonObject error = new JsonObject();
        error.addProperty("error", message);
        error.addProperty("status", status);
        response.getWriter().write(GSON.toJson(error));
    }
}
```

---

## 🔒 CSRF Protection in Servlets

```javascript
// Frontend JavaScript: Get CSRF token before POST
async function submitForm(formData) {
    // Step 1: Get CSRF token from AEM
    const tokenResponse = await fetch('/libs/granite/csrf/token.json');
    const { token } = await tokenResponse.json();

    // Step 2: Include token in POST request
    const response = await fetch('/bin/mysite/forms/contact', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'CSRF-Token': token           // AEM's CSRF header name
        },
        body: JSON.stringify(formData)
    });

    return response.json();
}
```

```java
// Server-side CSRF validation in servlet
import com.adobe.granite.csrf.api.CSRFTokenManager;

@Component(service = Servlet.class)
@SlingServletPaths("/bin/mysite/api/action")
public class SecureActionServlet extends SlingAllMethodsServlet {

    @Reference
    private CSRFTokenManager csrfTokenManager;

    @Override
    protected void doPost(SlingHttpServletRequest request, SlingHttpServletResponse response)
            throws ServletException, IOException {

        // Validate CSRF token
        if (!csrfTokenManager.isValidRequest(request)) {
            response.setStatus(403);
            response.getWriter().write("{\"error\": \"CSRF validation failed\"}");
            return;
        }

        // Process request safely
    }
}
```

---

## ↩️ Forward vs Redirect

```java
// FORWARD — Server-side, same request object, URL stays the same
// Use for: composing pages from multiple resources, internal error pages
request.getRequestDispatcher("/content/mysite/error.html")
       .forward(request, response);
// Browser sees original URL in address bar

// REDIRECT — Browser makes new request, URL changes
// Use for: post-form submission (PRG pattern), external URLs, permanent moves
response.sendRedirect("/content/mysite/success.html");     // 302 Temporary
// OR
response.setStatus(HttpServletResponse.SC_MOVED_PERMANENTLY);  // 301
response.setHeader("Location", "/content/mysite/new-page.html");
```

**Post/Redirect/Get (PRG) Pattern:**
```
User submits form (POST /submit)
    ↓
Server processes form
    ↓
Server sends 302 redirect to /thank-you
    ↓
Browser makes GET /thank-you
    ↓
User sees thank you page
    ↓
User hits refresh → only GET /thank-you repeats (safe!)
    ↓
Without PRG: refresh would re-submit the POST → duplicate submissions!
```

---

## 📖 Reading JCR Data in a Servlet (with Service User)

```java
@Reference
private ResourceResolverFactory resolverFactory;

private void readContentInServlet(String path) {
    // NEVER use request.getResourceResolver() for writes
    // Use service user for background JCR access

    Map<String, Object> params = new HashMap<>();
    params.put(ResourceResolverFactory.SUBSERVICE, "content-reader-service");

    try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(params)) {
        Resource resource = resolver.getResource(path);
        if (resource != null) {
            String title = resource.getValueMap().get("jcr:title", String.class);
            LOG.info("Found page: {}", title);
        }
    } catch (LoginException e) {
        LOG.error("Service user login failed for subservice: content-reader-service", e);
    }
    // try-with-resources automatically closes resolver
}
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is the difference between `SlingSafeMethodsServlet` and `SlingAllMethodsServlet`?**

> **Answer:** `SlingSafeMethodsServlet` only handles HTTP "safe" methods (GET, HEAD, OPTIONS, TRACE) — methods that should NOT change server state. If a POST/DELETE request comes in, it returns `405 Method Not Allowed` automatically. `SlingAllMethodsServlet` extends `SlingSafeMethodsServlet` and adds support for unsafe methods (POST, PUT, DELETE, PATCH). Use `SlingSafeMethodsServlet` for read-only data APIs, and `SlingAllMethodsServlet` for write operations like form submissions.

**Q2. What is the difference between path-based and resource-type servlet registration?**

> **Answer:** Path-based (`@SlingServletPaths`) registers a servlet at a fixed URL like `/bin/mysite/api`. It's accessible to anyone (unless protected by Dispatcher or authentication). Resource-type (`@SlingServletResourceTypes`) registers based on a combination of resource type + selector + extension. It's more secure because it inherits the permissions of the content being accessed. Resource-type registration is preferred for component APIs; path-based for admin/internal APIs.

**Q3. What is the Sling POST Servlet?**

> **Answer:** Sling's built-in POST servlet (`SlingPostServlet`) handles JCR content creation/modification via HTTP POST without any custom Java code. It's registered at `/bin/sling.post.servlet` and at content paths. It understands special parameters: `_charset_` (encoding), `:operation` (type of operation), `@Delete`, `@MoveFrom`. Used internally by AEM for dialog saves.

**Q4. How do you handle authentication in a servlet?**

> **Answer:** For internal/admin APIs, use the Sling Authentication framework to require login. In Dispatcher, use filter rules to block unauthenticated access. For AJAX endpoints from authenticated pages, ensure the session cookie is sent. For cross-origin requests, configure CORS. Always validate CSRF tokens for state-changing (POST/PUT/DELETE) requests. Use service users for any JCR access within the servlet — never use admin credentials.

**Q5. How would you return paginated results from a servlet?**

```java
@Override
protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response)
        throws ServletException, IOException {
    int page = Integer.parseInt(request.getParameter("page") != null
        ? request.getParameter("page") : "1");
    int pageSize = Integer.parseInt(request.getParameter("pageSize") != null
        ? request.getParameter("pageSize") : "10");

    int offset = (page - 1) * pageSize;

    List<Product> allProducts = productService.getAllProducts();
    int total = allProducts.size();
    List<Product> pageProducts = allProducts.subList(
        Math.min(offset, total),
        Math.min(offset + pageSize, total)
    );

    JsonObject result = new JsonObject();
    result.addProperty("page", page);
    result.addProperty("pageSize", pageSize);
    result.addProperty("total", total);
    result.addProperty("totalPages", (int) Math.ceil((double) total / pageSize));
    result.add("items", GSON.toJsonTree(pageProducts));

    response.getWriter().write(GSON.toJson(result));
}
```

**Q6. What is `request.getRequestDispatcher().forward()` vs `request.getRequestDispatcher().include()`?**

> **Answer:**
> - **forward()**: Transfers COMPLETE control to another resource. Current servlet stops writing to response. The forwarded resource writes the complete response.
> - **include()**: Includes the output of another resource INTO the current response. Current servlet can write BEFORE and AFTER the include. Used for modular page composition (e.g., including a header/footer).

---

## ✅ Best Practices

1. **Always prefer resource-type** servlet registration over path-based
2. **Set Content-Type header** before writing the response body
3. **Use try-catch** around all business logic and return proper HTTP status codes
4. **Validate all input** parameters — never trust client-provided data
5. **Use CSRF protection** for all POST/PUT/DELETE endpoints
6. **Use service users** for JCR access — never request.getResourceResolver() for writes
7. **Set appropriate Cache-Control headers** — helps Dispatcher/CDN cache GET responses
8. **Log with context** — include request path, resource path, user in error logs
9. **Return consistent JSON** error responses with `status` and `error` fields

---

## 🛠️ Hands-on Practice

**Day 16**: Create a GET servlet (resource-type) returning component data as JSON. Test with `.selector.json` URL.

**Day 17**: Create a path-based health check servlet at `/bin/mysite/health`. Return system status JSON.

**Day 18**: Build a POST servlet that accepts JSON body, validates fields, saves to JCR using service user.

**Day 19**: Add CSRF token validation to your POST servlet. Test from browser using Fetch API.
