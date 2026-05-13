# Day 16, 17, 18, 19 — Servlets
**Difficulty:** Hard | **4 YOE Focus**

---

## 📖 Topic Explanation

AEM Servlets are OSGi services that handle HTTP requests. They can be registered by **path** or by **resource type + selector + extension**.

---

## 🏗️ Servlet Types

### SlingSafeMethodsServlet
Handles only **read-only** HTTP methods: GET, HEAD, OPTIONS, TRACE
```java
@Component(service = Servlet.class)
@SlingServletResourceTypes(
    resourceTypes = "mysite/components/content/product",
    selectors = "data",
    extensions = "json",
    methods = HttpConstants.METHOD_GET
)
public class ProductDataServlet extends SlingSafeMethodsServlet {

    private static final Logger LOG = LoggerFactory.getLogger(ProductDataServlet.class);

    @Reference
    private ProductService productService;

    @Override
    protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response)
            throws ServletException, IOException {

        response.setContentType("application/json");
        response.setCharacterEncoding("UTF-8");

        String productId = request.getParameter("id");

        try {
            Product product = productService.getProduct(productId);
            String json = new Gson().toJson(product);
            response.getWriter().write(json);
        } catch (Exception e) {
            LOG.error("Error fetching product: {}", productId, e);
            response.setStatus(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
            response.getWriter().write("{\"error\": \"Failed to fetch product\"}");
        }
    }
}
```

**URL:** `/content/mysite/products/laptop.data.json?id=123`

---

### SlingAllMethodsServlet
Handles ALL HTTP methods including POST, PUT, DELETE
```java
@Component(service = Servlet.class)
@SlingServletResourceTypes(
    resourceTypes = "mysite/components/content/form",
    selectors = "submit",
    extensions = "json",
    methods = {HttpConstants.METHOD_GET, HttpConstants.METHOD_POST}
)
public class FormSubmitServlet extends SlingAllMethodsServlet {

    @Override
    protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response)
            throws ServletException, IOException {
        // Return form state or metadata
        response.getWriter().write("{\"status\": \"ready\"}");
    }

    @Override
    protected void doPost(SlingHttpServletRequest request, SlingHttpServletResponse response)
            throws ServletException, IOException {

        response.setContentType("application/json");

        String name = request.getParameter("name");
        String email = request.getParameter("email");

        // Validate
        if (name == null || name.isEmpty()) {
            response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
            response.getWriter().write("{\"error\": \"Name is required\"}");
            return;
        }

        // Process form submission
        // Save to JCR, send email, etc.

        response.setStatus(HttpServletResponse.SC_OK);
        response.getWriter().write("{\"success\": true, \"message\": \"Form submitted\"}");
    }
}
```

---

## 🗺️ Path-Based Servlet

```java
@Component(service = Servlet.class)
@SlingServletPaths("/bin/mysite/api/health")
public class HealthCheckServlet extends SlingSafeMethodsServlet {

    @Override
    protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response)
            throws ServletException, IOException {
        response.setContentType("application/json");
        response.getWriter().write("{\"status\": \"healthy\", \"version\": \"1.0.0\"}");
    }
}
```

**URL:** `/bin/mysite/api/health`

---

## ⚖️ Path vs Resource-Type Servlet

| Aspect | Path-Based | Resource-Type Based |
|--------|-----------|---------------------|
| Registration | `@SlingServletPaths("/bin/...")` | `@SlingServletResourceTypes(resourceTypes=...)` |
| URL Pattern | Fixed path | Content path + selector + extension |
| Security | HIGH RISK — open to all | Context-dependent, tied to content |
| Dispatcher | Needs explicit `/bin/` rules | Inherits content rules |
| Best Practice | ❌ Avoid (except internal APIs) | ✅ Preferred |

---

## 🌐 HTTP Methods in AEM

| Method | Use Case | Servlet Type |
|--------|---------|-------------|
| GET | Read data | `SlingSafeMethodsServlet` |
| POST | Create/submit | `SlingAllMethodsServlet` |
| PUT | Update | `SlingAllMethodsServlet` |
| DELETE | Remove | `SlingAllMethodsServlet` |

### Forward vs Redirect in Servlets
```java
// Forward: Internal, URL doesn't change in browser
request.getRequestDispatcher("/content/error.html").forward(request, response);

// Redirect: External, browser makes new request, URL changes
response.sendRedirect("/content/success.html");
```

---

## 🔒 Reading Request Body (POST)
```java
@Override
protected void doPost(SlingHttpServletRequest request, SlingHttpServletResponse response)
        throws ServletException, IOException {

    // Option 1: Form parameters
    String name = request.getParameter("name");

    // Option 2: Raw JSON body
    StringBuilder body = new StringBuilder();
    try (BufferedReader reader = request.getReader()) {
        String line;
        while ((line = reader.readLine()) != null) {
            body.append(line);
        }
    }
    JsonObject json = JsonParser.parseString(body.toString()).getAsJsonObject();
    String email = json.get("email").getAsString();
}
```

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between `SlingSafeMethodsServlet` and `SlingAllMethodsServlet`?**
> - `SlingSafeMethodsServlet`: Only handles safe (read-only) HTTP methods: GET, HEAD, OPTIONS, TRACE. Throws `405 Method Not Allowed` for POST/PUT/DELETE.
> - `SlingAllMethodsServlet`: Handles ALL HTTP methods including POST, PUT, DELETE. Extends `SlingSafeMethodsServlet`. Use when you need to handle write operations.

**Q2. What is the difference between path-based and resource-type servlet registration?**
> - **Path-based** (`@SlingServletPaths`): Registered at a fixed URL like `/bin/mysite/api`. Simple but risky — accessible to all without content-level permissions. Dispatcher must explicitly allow `/bin/` path.
> - **Resource-type** (`@SlingServletResourceTypes`): Tied to a resource type + selector + extension. Inherits content-level permissions. Preferred approach.

**Q3. How does Sling select which servlet handles a request?**
> Sling uses a resolution algorithm:
> 1. Find resource at the URL path
> 2. Check registered servlets for matching `resourceType` + `selector` + `extension` + `method`
> 3. Most specific match wins (specific selector > no selector > wildcard)
> 4. Fall back to default GET servlet

**Q4. What is the difference between forward and redirect?**
> - **Forward**: Server-side delegation. Browser URL doesn't change. Same request/response objects are reused. Faster.
> - **Redirect**: Server sends 301/302 to browser. Browser makes a NEW request to new URL. URL changes. New request/response objects.

**Q5. How do you secure a path-based servlet?**
> 1. Use Sling Authentication: require authentication via `@SlingServletPaths` + login redirect
> 2. Check CSRF token for POST requests
> 3. Validate user permissions manually: `session.checkPermission()`
> 4. Use service user for JCR operations (not request's session)
> 5. Rate limiting at Dispatcher level

**Q6. How do you handle CSRF protection in AEM servlets?**
> AEM provides CSRF protection via the **Sling CSRF Protection Filter**. For POST requests from the browser:
```javascript
// Frontend: Get CSRF token
fetch('/libs/granite/csrf/token.json')
  .then(r => r.json())
  .then(data => {
    // Include in POST header
    fetch('/bin/mysite/api', {
      method: 'POST',
      headers: { 'CSRF-Token': data.token }
    });
  });
```
> Validate on server: Sling's `CsrfUtil.isValidRequest(request)`.

**Q7. What is Sling POST Servlet and when do you use it?**
> Sling's built-in POST servlet (`/bin/sling.post.servlet`) handles JCR CRUD operations via HTTP POST without writing custom Java code. It understands special parameters like `_charset_`, `@Delete`, `@MoveFrom`. Used for simple content modifications (e.g., saving form data to JCR).

---

## ✅ Best Practices

- **Always** prefer resource-type based over path-based servlets
- Set `response.setContentType()` and `response.setCharacterEncoding()` explicitly
- Use try-catch + proper HTTP status codes — never let exceptions propagate unhandled
- Validate all input parameters (null check, length, format)
- Use service users for JCR access — never use `request.getResourceResolver()` for JCR writes
- Add CSRF validation for all state-changing (POST/PUT/DELETE) operations
- Log errors with `LOG.error()` — include request context in message

---

## 🛠️ Hands-on Tasks

**Day 16**: Create a GET servlet (resource-type) returning component data as JSON
**Day 17**: Create a path-based health check servlet at `/bin/mysite/health`
**Day 18**: Build a POST servlet that accepts JSON body and saves to JCR
**Day 19**: Extend `SlingSafeMethodsServlet` for read-only endpoint, validate with CSRF
