# Mock Interview — 75 Rapid-Fire Q&A
**Target:** 2–4 YOE | Last-Day Revision Before Your Interview

---

> 📌 **How to use this file:** Cover the answer and try to answer each question out loud. Then uncover. Say the key phrases aloud — you'll remember them better under interview pressure.

---

## 🏛️ SECTION 1: Architecture & Fundamentals (15 Questions)

**Q1. What are the main layers of AEM's architecture?**
> JCR (data storage) → Sling (REST framework, resource resolution) → OSGi (plugin system, services) → AEM WCM (authoring layer). Dispatcher + Apache httpd sits in front for caching and security.

**Q2. What is the role of Apache Sling in AEM?**
> Sling maps HTTP URLs to JCR content via `sling:resourceType` and resolves which script/servlet to render the resource. It implements REST principles — every piece of content is a resource with a URL.

**Q3. What is the difference between Author and Publish instances?**
> Author is for content creation (restricted access, authoring UI, non-cached). Publish is for serving public content (Dispatcher-cached, optimized for high traffic). Content flows Author → Replication → Publish.

**Q4. What does the Dispatcher do?**
> Three roles: Caching (stores rendered HTML as flat files), Load balancing (distributes to multiple Publish instances), and Security (blocks requests to sensitive paths like `/system/console`, `/crx/de`).

**Q5. What is a JCR node type?**
> A node type defines the structure constraints of a JCR node — which properties and child nodes it can have. E.g., `cq:Page` must have a `jcr:content` child of type `cq:PageContent`. `nt:unstructured` allows anything.

**Q6. What is sling:resourceType?**
> A property on a JCR node that tells Sling which component (script/servlet) to use for rendering. Sling looks in `/apps/<resourceType>/` for a rendering script. This is the core of AEM's component model.

**Q7. What is sling:resourceSuperType?**
> The "inheritance" property — if a script is not found for a resourceType, Sling walks up the `sling:resourceSuperType` chain until it finds one. Enables the component extension/delegation pattern.

**Q8. What is the difference between run modes?**
> Run modes configure AEM for specific contexts: `author`/`publish` (instance type), `dev`/`stage`/`prod` (environment). Used for environment-specific OSGi configs. `config.author.prod` applies only to Author in Production.

**Q9. Explain the AEM request flow end-to-end.**
> Browser → CDN (cache miss?) → Dispatcher/Apache (cache miss?) → AEM Publish → Sling URL decomposition → Resource resolution via `sling:resourceType` → Sling Model injection → HTL rendering → Response cached by Dispatcher.

**Q10. What is the difference between `/apps` and `/libs`?**
> `/libs` contains Adobe-provided content (default components, system configs). `/apps` contains your project code. Sling's resource resolver searches `/apps` BEFORE `/libs` — so creating `/apps/core/...` overlays `/libs/core/...`.

**Q11. What is Dispatcher cache invalidation?**
> When AEM publishes content, a replication agent sends an HTTP DELETE to Dispatcher's `/dispatcher/invalidate.cache`. Dispatcher marks the `.stat` file — the next request for that path fetches fresh content from AEM.

**Q12. What is `statfileslevel` in Dispatcher?**
> Controls how deep in the directory tree the `.stat` file is placed during invalidation. Higher = more granular = fewer pages invalidated per publish = better cache hit ratio.

**Q13. What is a Closed User Group (CUG)?**
> A CUG restricts read access to a content tree on the Publish tier to specific groups. Users not in the group get a 401 and are redirected to login. Configured via Page Properties → Permissions.

**Q14. What is `/etc/map` in Sling?**
> URL mapping configuration that maps JCR paths to clean public URLs. `resolver.map("/content/mysite/en/home")` uses these rules to return the clean URL. Avoids exposing `/content/...` paths to users.

**Q15. What is the difference between TarMK and MongoMK?**
> TarMK stores content in `.tar` files on local filesystem — default for Publish and standalone Author. MongoMK uses MongoDB and enables multiple Author instances to share the same repository (clustering). AEM Cloud uses Azure Blob Storage (neither).

---

## 🏗️ SECTION 2: Components & Dialogs (10 Questions)

**Q16. What is the minimal structure of an AEM component?**
> `.content.xml` (component definition with `jcr:primaryType="cq:Component"`, `sling:resourceSuperType`, `componentGroup`) and `<componentname>.html` (HTL rendering script). Optionally: `_cq_dialog/` for authoring dialog.

**Q17. What is the difference between `_cq_dialog` and `_cq_design_dialog`?**
> `_cq_dialog` is the **author dialog** for per-instance configuration (shown when author clicks on a component). `_cq_design_dialog` is legacy for **design** (policy) configuration — replaced by Content Policies in modern AEM.

**Q18. What are the three layers of an Editable Template?**
> **Structure** — locked layout elements editors can't move/delete. **Initial Content** — default content that appears in new pages but can be edited. **Policy** — content policy that restricts allowed components and defines Style System options.

**Q19. What is a Content Policy?**
> A reusable configuration that restricts a component's allowed behavior (e.g., which components are in a parsys, image sizes, heading types). Defined once, shared across templates. Replaces `design_dialog` in modern AEM.

**Q20. What is the AEM Style System?**
> Lets authors apply pre-defined CSS classes to components without touching code. Developers define styles in Content Policies (each style = CSS class name), authors pick them from the component toolbar. Reduces need for separate "variant" components.

**Q21. What is a parsys?**
> A "paragraph system" — a container component that allows authors to drag and drop other components into it. The main editable area on most pages. In modern AEM, implemented as a `responsivegrid` (layout container).

**Q22. What is a Core Component and why use it?**
> Adobe's open-source library of production-ready components (Image, Title, Text, Navigation, etc.). Use them because they're WCAG-accessible, Style System-ready, maintained by Adobe, and save weeks of development time.

**Q23. What is the delegation pattern for Core Components?**
> Set `sling:resourceSuperType` to the core component. In your Sling Model, inject the core model with `@Self @Via(type = ResourceSuperType.class)`. Delegate all interface methods to the core model, override only what you need. Never copy core component code.

**Q24. What is `allowProxy` in a clientlib?**
> When `allowProxy=true`, the clientlib is served via `/etc.clientlibs/` instead of `/apps/` — Dispatcher allows `/etc.clientlibs/*` but blocks `/apps/*`. Essential for security — all clientlibs should have `allowProxy=true`.

**Q25. What is the difference between embedding and depending on clientlibs?**
> `embed` physically merges the embedded library's JS/CSS into your library (single HTTP request). `dependencies` loads the other library as a separate request (two requests, but shared caching). Use embed for small utilities, dependencies for large shared libraries.

---

## ☕ SECTION 3: Sling Models (12 Questions)

**Q26. What is a Sling Model?**
> A Java POJO annotated with `@Model` that binds JCR/Sling data to Java fields via injection annotations. AEM's recommended way to provide data to HTL templates, replacing Scriptlets. Adapts from `Resource` or `SlingHttpServletRequest`.

**Q27. When should you use `Resource.class` vs `SlingHttpServletRequest.class` as adaptable?**
> Use `Resource.class` when you only need JCR data (most components). Use `SlingHttpServletRequest.class` when you need request context (params, cookies, session, `@ScriptVariable Page currentPage`). Declare both for maximum compatibility.

**Q28. Why use `DefaultInjectionStrategy.OPTIONAL`?**
> Prevents `ModelAdaptionException` if an injection fails. With OPTIONAL, a failed injection sets the field to null (safe) vs REQUIRED which throws and crashes the component (and potentially the page). Always use OPTIONAL in production.

**Q29. What does `@PostConstruct` do?**
> Method annotated with `@PostConstruct` is called AFTER all injection annotations have been processed. Use it to compute derived values, call OSGi services, null-check injected fields, and build complex data structures from injected raw data.

**Q30. What is the difference between `@ValueMapValue` and `@Inject`?**
> `@ValueMapValue` explicitly reads from the resource's JCR ValueMap (component dialog values). `@Inject` is generic — Sling tries all injectors in order (slower, ambiguous). Always use specific annotations (`@ValueMapValue`, `@OSGiService`, `@SlingObject`, `@ChildResource`) over generic `@Inject`.

**Q31. What is `@ChildResource` used for?**
> Injects a named child JCR node as a `Resource` or adaptable Sling Model. Used for multifield data where each row is stored as a child node, and for nested component structures.

**Q32. How do you map a composite multifield to a List of Java POJOs?**
> Use `@ChildResource private List<Resource> items` to inject all children of the multifield node. Then in `@PostConstruct`, iterate and adapt each child: `item.adaptTo(MyMultifieldModel.class)`. Each child model reads its own properties via `@ValueMapValue`.

**Q33. What does `@Self @Via(type = ResourceSuperType.class)` do?**
> Injects an instance of the PARENT component's Sling Model (following sling:resourceSuperType). Enables delegation: your model gets the parent's model instance and can call all its interface methods.

**Q34. What is `@ScriptVariable` and what objects does it inject?**
> Injects variables from AEM's rendering context: `Page currentPage`, `PageManager pageManager`, `WCMMode wcmMode`, `ValueMap pageProperties`, `ComponentContext componentContext`. These are populated by AEM's rendering engine and available only when adapting from `SlingHttpServletRequest`.

**Q35. What is `@OSGiService`?**
> Injects an OSGi service into a Sling Model (equivalent to `@Reference` in OSGi `@Component` classes). Used to call external APIs, query services, or content policies from within a model.

**Q36. How do you debug a Sling Model that isn't adapting?**
> 1) `/system/console/status-slingjmodelregistry` — is the class registered? 2) Check `adaptables` and `resourceType` are correct. 3) Enable DEBUG logging for the model class. 4) Check error.log for exceptions in `@PostConstruct`. 5) Verify the JCR properties exist and names match field names exactly.

**Q37. What happens if a `@ValueMapValue` field has a different name than the JCR property?**
> Use `@ValueMapValue(name = "jcr:propertyName")` to specify the exact JCR property name. Otherwise, Sling uses the Java field name as the property name by default.

---

## ⚙️ SECTION 4: OSGi Services (10 Questions)

**Q38. What is the OSGi lifecycle of a component?**
> Registered → Installed → Resolved → Activating (`@Activate`) → Active → Deactivating (`@Deactivate`) → Installed. Configuration changes trigger `@Modified`. Component stays in Active state during normal operation.

**Q39. What does `@Designate` do?**
> Links an OSGi `@Component` to its configuration interface (`@ObjectClassDefinition`). Without it, the component has no configurable properties in `/system/console/configMgr`.

**Q40. What is the difference between `@Activate`, `@Modified`, and `@Deactivate`?**
> `@Activate`: called when component starts (first time + on config file deploy). `@Modified`: called when OSGi config changes at runtime (component doesn't restart). `@Deactivate`: called when component stops — ALWAYS release resources here to prevent memory leaks.

**Q41. What causes an OSGi component to fail to activate?**
> 1) Unsatisfied `@Reference` (dependency service not available/failed). 2) Exception thrown in `@Activate`. 3) Missing OSGi config file for a required config. 4) Missing OSGi package import (bundle not exported). View failure reason in `/system/console/components`.

**Q42. What is `ReferenceCardinality.MULTIPLE` with `ReferencePolicy.DYNAMIC`?**
> Collects ALL registered implementations of a service interface as a dynamic `List`. As bundles deploy/undeploy, the list updates at runtime. Requires `volatile`. Useful for plugin patterns (all processors, all handlers).

**Q43. What is `immediate = true` in `@Component`?**
> Forces the component to start immediately when its bundle activates — even without any consumers. Default is lazy (start only when first requested). Use for event listeners, schedulers, background services that must run independently.

**Q44. What is the OSGi Config PID?**
> The Persistent Identity of an OSGi configuration — typically the fully-qualified class name of the component. The `.cfg.json` file name must match the PID. Multiple configs of the same component use the format: `<PID>-<name>.cfg.json`.

**Q45. How do you inject a service user in an OSGi service?**
> Use `@Reference private ResourceResolverFactory resolverFactory` then `resolverFactory.getServiceResourceResolver(Map.of(ResourceResolverFactory.SUBSERVICE, "my-subservice"))`. Map the subservice to a system user via `ServiceUserMapperImpl.amended-<name>.cfg.json`.

**Q46. What is the OSGi ConfigurationAdmin?**
> The OSGi service that manages configurations. AEM reads `*.cfg.json` files at startup and applies them via ConfigurationAdmin. You can interact with it directly in code, but usually the `@Designate` annotation handles this automatically.

**Q47. How do you create an OSGi Scheduled Task?**
> Implement `Runnable`, register as a `Runnable.class` service. Inject `Scheduler` OSGi service. In `@Activate`, call `scheduler.schedule(this, schedulerOptions)` with a `SchedulerOptions.cron()` expression. In `@Deactivate`, call `scheduler.unschedule(jobName)`.

---

## 🌐 SECTION 5: Servlets, HTL & Security (10 Questions)

**Q48. What is the difference between resource-type and path-based servlets?**
> Resource-type servlet: registered via `@SlingServletResourceTypes` for a component's resourceType — inherits JCR ACLs, doesn't need Dispatcher rules. Path servlet: registered via `@SlingServletPaths` at a fixed URL (e.g., `/bin/mysite/api`) — needs explicit Dispatcher allow rules. Prefer resource-type.

**Q49. What is the difference between `SlingSafeMethodsServlet` and `SlingAllMethodsServlet`?**
> `SlingSafeMethodsServlet`: only handles GET and HEAD — safe read-only operations. `SlingAllMethodsServlet`: handles all HTTP methods including POST, DELETE, PUT. Use `SlingSafeMethodsServlet` for read-only data endpoints.

**Q50. What is context-aware escaping in HTL?**
> HTL automatically escapes output based on WHERE it's placed: `${value}` in HTML → HTML entity encoding. `${value @ context='uri'}` in href → URL encoding. `${value @ context='attribute'}` in attribute → attribute escaping. `context='unsafe'` disables escaping — NEVER use.

**Q51. Name 5 HTL statements (data-sly-*).**
> `data-sly-use` (adapt model/template), `data-sly-test` (conditional render), `data-sly-list` (iterate array), `data-sly-resource` (include child resource), `data-sly-include` (include script), `data-sly-template` (define template), `data-sly-call` (call template), `data-sly-text` (set text content), `data-sly-attribute` (set attributes), `data-sly-element` (change tag name).

**Q52. What is `<sly>`?**
> A virtual HTL tag that renders no HTML output itself — used to wrap `data-sly-*` statements without adding HTML elements. `<sly data-sly-use.model="..."/>` loads a model without adding a `<sly>` element to the DOM.

**Q53. What is the CSRF protection in AEM and how do you handle it in POST servlets?**
> AEM uses `CsrfFilter` that blocks POST requests without a valid CSRF token. Get the token via `GET /libs/granite/csrf/token.json` → returns `{"token":"..."}`. Include in POST as `X-CSRF-Token: <token>` header or `:cq_csrf_token` form field. Disable CSRF protection for trusted-internal endpoints via OSGi config.

**Q54. What happens if you call `response.sendRedirect()` vs `request.getRequestDispatcher().forward()`?**
> `sendRedirect()`: sends 302 to browser — browser makes a NEW request to the redirected URL. URL in browser changes. `forward()` (internal): server-side forward — same request, no new HTTP request, URL doesn't change in browser. Use redirect for external URLs or after POST; use forward for internal resource composition.

**Q55. What is a Sling filter and how is it different from a servlet?**
> A Sling filter (`SlingFilter` or `javax.servlet.Filter`) intercepts ALL matching requests before they reach the servlet/renderer. Used for: authentication, logging, request modification, response compression, WCM mode injection. Different from a servlet which handles specific resource/selector/extension combinations.

**Q56. What sensitive paths must Dispatcher block?**
> `/system/console` (OSGi console), `/crx/de` (CRXDE), `/libs/granite/*` (admin tools), `/bin/wcm/*` (WCM commands), `*.infinity.json` (JCR dump), `/content/*.json` (data exposure), `/mnt/overlay/*` (overlay browser).

**Q57. What is `rep:policy` in JCR?**
> A special child node (`jcr:primaryType="rep:ACL"`) that stores the Access Control List for a JCR node. Contains `rep:GrantACE` and `rep:DenyACE` entries defining which principals can do what. Managed by Oak's security framework.

---

## 📦 SECTION 6: DAM, Query & Cloud (10 Questions)

**Q58. What node type is a DAM asset?**
> `dam:Asset`. Its content is stored under `jcr:content` (type: `dam:AssetContent`), with metadata under `jcr:content/metadata` and renditions under `jcr:content/renditions/`.

**Q59. How do you adapt a resource to an Asset?**
> `Asset asset = resource.adaptTo(Asset.class)`. Returns null if the resource is not a `dam:Asset`. Use `asset.getMimeType()`, `asset.getMetadataValue(DamConstants.DC_TITLE)`, `asset.getRenditions()`.

**Q60. What is the DAM Update Asset workflow?**
> AEM's built-in workflow that runs on every asset upload: extracts metadata (EXIF/IPTC/XMP), generates thumbnails, creates web renditions, applies smart tags. In AEM Cloud, replaced by Asset Compute (serverless processing workers).

**Q61. What is Query Builder?**
> AEM's high-level content search API using key-value predicate maps. Simpler than JCR-SQL2, with built-in pagination (`p.limit`, `p.offset`), testable via `/bin/querybuilder.json` in browser, and automatic Oak index usage.

**Q62. What does `p.limit=-1` in Query Builder mean?**
> No result limit — returns ALL matching nodes. Dangerous for large repositories (can OOM). Always set explicit limits (`p.limit=20`) and paginate. Use `-1` only in batch background jobs that process results incrementally.

**Q63. What is a traversal query in Oak?**
> A query that doesn't use any Oak index — Oak scans ALL nodes matching the path. Extremely slow on large repos. Detected by "Traversal query" warnings in error.log. Fix by creating a Lucene index for the queried property.

**Q64. What is the immutable architecture in AEM Cloud?**
> `/apps`, `/libs`, `/oak:index` are READ-ONLY at runtime. No Package Manager in production. No CRXDE editing of `/apps`. All code must be deployed via Cloud Manager pipeline. Mutable paths: `/content`, `/conf`, `/var`, `/home`.

**Q65. What is the difference between a Content Fragment and an Experience Fragment?**
> CF = pure structured data, no presentation, delivered via GraphQL/REST. XF = designed visual block (HTML with components), embedded in pages or exported to Adobe Target. CF = headless content. XF = reusable UI block.

**Q66. What is Sling Model Exporter?**
> Adds `@Exporter(name="jackson", extensions="model")` to a Sling Model, making it accessible as JSON at `<resource>.model.json`. Combined with `ComponentExporter` interface, enables AEM SPA Editor where React/Angular consumes component data as JSON.

**Q67. What is a persisted GraphQL query?**
> A pre-approved, server-stored GraphQL query accessible via HTTP GET. More secure (no arbitrary queries) and cacheable by CDN (GET requests can be cached, POST cannot). Required for production headless delivery in AEM Cloud.

---

## 🏆 SECTION 7: Senior-Level Questions (8 Questions)

**Q68. What is the service user pattern and why is it essential?**
> Create a system user with minimum required JCR permissions (via repoinit). Map it via `ServiceUserMapperImpl.amended`. Get the resolver via `resolverFactory.getServiceResourceResolver(Map.of(SUBSERVICE, "name"))`. Use `try-with-resources`. This replaces the deprecated `loginAdministrative()` — enforces principle of least privilege.

**Q69. A component works in Author but not on Publish. What could be wrong?**
> 1) Component needs a service user with permissions — Publish may not have the right user setup. 2) Component calls an OSGi service that has Author-only config. 3) Content not replicated to Publish. 4) Dispatcher is caching an old version. 5) Different OSGi run mode configs (`config.publish/`) missing a required configuration.

**Q70. How do you prevent memory leaks in AEM?**
> Always use `try-with-resources` for ResourceResolver and Session. In OSGi services: close HTTP clients in `@Deactivate`, unregister JCR event listeners, clear caches. Never store mutable state in static fields without eviction. Always unschedule scheduler jobs in `@Deactivate`.

**Q71. What is the difference between `resourceResolver.commit()` and `session.save()`?**
> Both commit pending changes. `resourceResolver.commit()` is the Sling API — works at the Sling abstraction level. `session.save()` is the JCR API — lower level. In Sling context, prefer `resourceResolver.commit()`. Never call both — one will throw. Either can be used, but not both.

**Q72. How does MSM (Multi-Site Manager) inheritance work?**
> A Blueprint defines source content. Live Copies inherit from the Blueprint via Rollout Config. Each property has inheritance lock/unlock state. Locked properties sync from Blueprint on rollout. Unlocked properties are local overrides (not overwritten by rollout). Detached live copies have no Blueprint relationship.

**Q73. How do you design a component that calls an external API?**
> 1) Create an OSGi service (`@Component`) that handles the HTTP call with timeout, error handling, and caching (Guava Cache with TTL). 2) Inject the service into the Sling Model via `@OSGiService`. 3) In `@PostConstruct`, call the service only when `WCMMode.DISABLED` (Publish) — skip in Author. 4) Handle null/error response gracefully. 5) Never put API keys in code — use OSGi secret variables.

**Q74. What SonarQube quality gate thresholds does Cloud Manager enforce?**
> Key metrics: Code coverage (unit tests, threshold configurable — typically 50%+), 0 Critical/Blocker code issues, security vulnerabilities (0 critical), reliability rating (typically A or B). Failure blocks the pipeline from advancing to the next stage.

**Q75. What would you check if a Cloud Manager pipeline fails?**
> 1) Build failure → check Maven build output log for compile errors or missing dependencies. 2) Unit test failure → check JUnit test report in Cloud Manager. 3) Quality gate failure → check SonarQube report link in Cloud Manager. 4) Deploy failure → check `error.log` for the target environment in Cloud Manager log download. 5) AEM health check failure → check `/system/console/components` on the environment for failed OSGi components.

---

## ⚡ POWER PHRASES — Say These in Interviews

```
"I always use try-with-resources for ResourceResolvers —
 an unclosed resolver silently leaks until it causes OOM."

"DefaultInjectionStrategy.OPTIONAL is my default —
 a null field degrades gracefully, a ModelAdaptionException crashes the page for all users."

"For service users, I follow principle of least privilege —
 each subservice name maps to a system user with ONLY the paths it needs."

"In AEM Cloud, the immutable architecture means I can't use Package Manager in production —
 everything goes through the Cloud Manager pipeline."

"I prefer resource-type servlet registration over path-based —
 it inherits the content node's ACLs and doesn't need explicit Dispatcher allow rules."

"When extending Core Components, I use delegation —
 @Self @Via(type = ResourceSuperType.class) gives me the parent model,
 and I only override what I need. Never copy core component code."

"For Cache-Control headers, I coordinate between AEM's component TTL,
 Dispatcher statfileslevel, and CDN TTL — all three layers matter."

"In my @Deactivate, I always unschedule jobs, close HTTP clients, clear caches,
 and unregister JCR observation listeners — failing this causes memory leaks
 that accumulate until the JVM crashes."
```

---

## 📊 Scoring Your Readiness

| Score | Interpretation |
|-------|---------------|
| **60+/75** | 🟢 Ready for senior AEM roles (4 YOE) |
| **45–60/75** | 🟡 Ready for mid-level AEM roles (2-3 YOE) |
| **30–45/75** | 🟠 Need 1–2 more weeks of practice |
| **< 30/75** | 🔴 Revisit foundational modules (Days 1–15) |

**Focus on these if score < 45:**
- Sling Models (Section 3) — most asked
- OSGi Services (Section 4) — most differentiating
- HTL context escaping (Q50) — security knowledge
- Service users (Q68) — always asked at senior level
