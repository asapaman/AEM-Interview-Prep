# ACS AEM Commons — Complete Guide
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What is ACS AEM Commons?

**ACS AEM Commons** (Adobe Consulting Services AEM Commons) is the most popular open-source library for AEM, created and maintained by Adobe's own consulting team. It provides:

- **Ready-to-use utilities** — components, OSGi services, and tools that solve common AEM problems
- **Development helpers** — error handler, MCP tools, Groovy console
- **Asset management tools** — bulk import, rendition utilities
- **Content authoring tools** — components for forms, lists, reporting

> Think of ACS AEM Commons as the "AEM Swiss Army knife" — a collection of battle-tested solutions that you'd otherwise need to write from scratch.

### Installation

```bash
# Download from: https://github.com/Adobe-Consulting-Services/acs-aem-commons/releases
# Install via Package Manager: /crx/packmgr

# Maven dependency (add to ui.apps pom.xml):
<dependency>
    <groupId>com.adobe.acs</groupId>
    <artifactId>acs-aem-commons-bundle</artifactId>
    <version>6.x.x</version>
    <scope>provided</scope>  <!-- ACS is installed separately as a package -->
</dependency>
```

---

## 🧰 Key ACS AEM Commons Features

### 1. EnsureOakIndex — Index Management

Ensures custom Oak indexes are created or updated automatically on startup — without manual CRXDE work.

```xml
<!-- /apps/mysite/oak:index/acs-ensure-oak-index-definition/
                             mysite-page-type/.content.xml -->
<jcr:root
    xmlns:oak="http://jackrabbit.apache.org/oak/ns/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    jcr:primaryType="nt:unstructured"
    delete="{Boolean}false"
    ignore="{Boolean}false">

  <!-- The actual Oak index definition to ensure -->
  <oakIndexDefinition
      jcr:primaryType="oak:QueryIndexDefinition"
      type="lucene"
      async="async">
    <indexRules jcr:primaryType="nt:unstructured">
      <cq:Page jcr:primaryType="nt:unstructured">
        <properties jcr:primaryType="nt:unstructured">
          <pageType jcr:primaryType="nt:unstructured"
                   name="jcr:content/pageType"
                   propertyIndex="{Boolean}true"/>
        </properties>
      </cq:Page>
    </indexRules>
  </oakIndexDefinition>
</jcr:root>
```

---

### 2. Generic Lists — Dropdown Data Management

**Problem:** You have a dropdown in a dialog that needs items managed by content editors (not developers).

**ACS Solution:** Store dropdown options as a JCR list that editors can update via AEM UI.

```
/etc/acs-commons/lists/
  └── product-categories/
        └── jcr:content/
              └── list/
                    ├── item0/ → text="Electronics", value="electronics"
                    ├── item1/ → text="Clothing", value="clothing"
                    └── item2/ → text="Books", value="books"
```

```xml
<!-- Dialog: Use Generic List as datasource -->
<categories jcr:primaryType="nt:unstructured"
            sling:resourceType="granite/ui/components/coral/foundation/form/select"
            fieldLabel="Product Category"
            name="./category">
  <datasource jcr:primaryType="nt:unstructured"
              sling:resourceType="acs-commons/components/utilities/genericlist/datasource"
              path="/etc/acs-commons/lists/product-categories"/>
</categories>
```

```java
// In Sling Model: read the selected value
@ValueMapValue
private String category;  // Reads the author's selected value (e.g., "electronics")
```

---

### 3. Throttled Task Runner / MCP (Manage Controlled Processes)

**Problem:** Running a bulk operation (migrate 50,000 pages) in a single thread blocks AEM and risks timeouts.

**ACS Solution:** MCP (Manage Controlled Processes) framework runs operations with controlled threading and progress reporting.

```java
import com.adobe.acs.commons.mcp.ProcessDefinition;
import com.adobe.acs.commons.mcp.ProcessInstance;
import com.adobe.acs.commons.mcp.form.*;

@Component(service = ProcessDefinition.class)
public class BulkPageUpdateProcess extends ProcessDefinition {

    @FormField(
        name = "Content Root",
        description = "Root path to process",
        hint = "/content/mysite",
        component = PathfieldComponent.class,
        required = true
    )
    private String contentRoot;

    @FormField(
        name = "New Brand Name",
        description = "Brand name to set",
        required = true
    )
    private String brandName;

    @FormField(
        name = "Dry Run",
        description = "Preview changes without saving",
        component = CheckboxComponent.class
    )
    private boolean dryRun = true;

    @Override
    public void init() throws RepositoryException {
        // Initialization before processing starts
    }

    @Override
    public void run(ProcessInstance instance, ResourceResolver resolver,
                    ActionManager manager) throws Exception {
        // Manager handles concurrency and progress tracking
        manager.withResolver(resolver, r -> {
            // Process content root — find all pages
            Resource root = r.getResource(contentRoot);
            if (root == null) throw new IllegalArgumentException("Root not found: " + contentRoot);

            processPages(root, r, manager, instance);
        });
    }

    private void processPages(Resource resource, ResourceResolver resolver,
                               ActionManager manager, ProcessInstance instance) {
        if ("cq:PageContent".equals(resource.getResourceType())) {
            manager.deferredWithResolver(r -> {
                // This runs on a background thread, managed by MCP
                ModifiableValueMap mvm = resource.adaptTo(ModifiableValueMap.class);
                if (mvm != null && !dryRun) {
                    mvm.put("brandName", brandName);
                    r.commit();
                }
                instance.getReportBuilder().addRow(
                    "Updated", resource.getPath(), brandName
                );
            });
        }

        for (Resource child : resource.getChildren()) {
            processPages(child, resolver, manager, instance);
        }
    }

    @Override
    public void storeReport(ProcessInstance instance, ResourceResolver rr)
            throws RepositoryException, PersistenceException {
        // Save report results to JCR
        instance.persistReport(rr);
    }
}
```

Access MCP at: `/mnt/overlay/acs-commons/content/manage-controlled-processes.html`

---

### 4. Error Page Handler

Provides custom error pages (404, 500, etc.) configured per content tree.

```xml
<!-- OSGi Config: Enable ACS Error Page Handler -->
<!-- /apps/mysite/osgiconfig/config/com.adobe.acs.commons.errorpagehandler.impl.ErrorPageHandlerImpl.cfg.json -->
{
    "enabled": true,
    "error-page.path": "/content/mysite/en/errors",
    "error-page.extension": "html",
    "error-codes": ["404", "500", "403"],
    "serve-authenticated-from-cache": false
}
```

```
/content/mysite/en/errors/
  ├── 404/         ← Custom 404 page
  │     └── jcr:content/ (cq:Page)
  ├── 500/         ← Custom 500 page
  └── 403/         ← Custom 403 page
```

---

### 5. Named Image Transform / Image Transform Service

Pre-define named image transformations (crop, resize, rotate) configured in OSGi.

```json
// Named Image Transforms OSGi config
{
    "named.transforms": [
        "thumbnail:{ width: 200, height: 200, mode: \"bound\", quality: 80 }",
        "hero:{ width: 1920, height: 600, mode: \"crop\", quality: 90 }",
        "card:{ width: 400, height: 300, mode: \"crop\", quality: 85 }"
    ]
}
```

```html
<!-- Use named transform in HTL to serve correctly sized image -->
<img src="${resource.path}/jcr:content/image.thumbnail.jpg @ context='uri'}"
     alt="${model.imageAlt}"/>
```

---

### 6. Redirect Map Manager

Manages URL redirects without requiring code deployments. Editors can upload CSV files with redirects.

```
/etc/acs-commons/redirect-maps/mysite-redirects/
  └── jcr:content/
        └── redirectMap.txt  ← CSV: old-path,new-path
```

```apache
# Generated .htaccess / Apache RewriteMap:
/old/products   /content/mysite/en/products.html
/old/about      /content/mysite/en/about-us.html
/old/contact    /content/mysite/en/contact.html
```

```xml
<!-- Enable in Dispatcher virtual host -->
RewriteMap acs-redirects txt:/etc/httpd/conf.dispatcher.d/redirects/mysite.txt
RewriteCond ${acs-redirects:%{REQUEST_URI}} !=""
RewriteRule .* ${acs-redirects:%{REQUEST_URI}} [R=301,L]
```

---

### 7. Shared Component Properties / Component Helper

**Shared Component Properties** store properties accessible by multiple components on the same page — avoiding repetitive configuration.

```java
// Reading shared page properties
import com.adobe.acs.commons.wcm.SharedComponentProperties;

@Model(adaptables = SlingHttpServletRequest.class, ...)
public class MyModel {

    @OSGiService
    private SharedComponentProperties sharedComponentProperties;

    @ScriptVariable
    private Page currentPage;

    @PostConstruct
    protected void init() {
        // Get properties shared across all components on this page
        ValueMap sharedProps = sharedComponentProperties.getSharedProperties(request, response);
        String globalBrand = sharedProps.get("brandName", String.class);
    }
}
```

---

### 8. On/Off Time — Content Scheduling

ACS provides a Sling Filter that shows/hides content based on configured on/off times — without workflow.

```
Component Properties:
  onTime  = 2024-06-01T08:00:00
  offTime = 2024-07-01T00:00:00
```

```java
// ACS handles this automatically if configured — the content is hidden outside the window
// Works with Sling onTime/offTime properties standard
```

---

### 9. ACS Tags — Tag Namespace Management

ACS provides a UI for managing tag namespaces and hierarchies more easily.

```
Tools → ACS Commons → Tags
→ Create namespace, create tags, merge tags, bulk tag assets
```

---

### 10. CSV Asset Importer

Bulk import assets and their metadata from a CSV file — critical for large content migrations.

```csv
# import-assets.csv format:
File,Title,Description,Tags
/assets/laptop.jpg,Laptop Pro 15,High-performance laptop,mysite:products/electronics
/assets/phone.jpg,SmartPhone X,Latest smartphone,mysite:products/electronics/mobile
```

```
Tools → ACS Commons → CSV Asset Importer
→ Upload CSV → Map columns to metadata fields
→ Run import → Creates/updates assets with metadata
```

---

## 🔧 Common OSGi Configurations for ACS AEM Commons

```json
// Throttled Task Runner configuration
// com.adobe.acs.commons.util.impl.WorkflowPackageManagerImpl.cfg.json
{
    "max.running.workflows": 1,
    "workflow.purge.max.age": 30,
    "min.query.save.threshold": 1000
}
```

```json
// AEM Environment Indicator (shows dev/stage banner in Author UI)
// com.adobe.acs.commons.environment.impl.EnvironmentIndicatorImpl.cfg.json
{
    "css-color": "#E74C3C",
    "title": "DEV ENVIRONMENT"
}
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is ACS AEM Commons and why would you use it in a project?**

> **Answer:** ACS AEM Commons is an open-source library maintained by Adobe's consulting team that provides reusable utilities, components, and tools for AEM development. Use it to avoid reinventing common solutions: Generic Lists for editor-managed dropdowns, Error Page Handler for custom error pages, MCP for safe bulk operations, Redirect Map Manager for business-managed redirects. It's well-tested, maintained, and widely used in enterprise AEM projects.

**Q2. What is a Generic List and how does it work?**

> **Answer:** A Generic List is an ACS-managed JCR structure under `/etc/acs-commons/lists/` that stores key-value pairs as JCR nodes. It's used as a datasource for dialog dropdowns — editors can add/remove/edit dropdown options via AEM's list editor UI without developer code changes. In the dialog XML, reference it via `sling:resourceType="acs-commons/components/utilities/genericlist/datasource"` with the list path. The selected value is stored like any other dialog property.

**Q3. What is MCP (Manage Controlled Processes)?**

> **Answer:** MCP is ACS's framework for running bulk/long-running AEM operations safely. It provides: a form-based UI for operators to configure and trigger the process, controlled threading to avoid overloading AEM, progress reporting and logging, and the ability to pause/stop operations. Use it instead of running raw Groovy scripts for bulk content migrations — it's safer, trackable, and auditable.

**Q4. How does the ACS Error Page Handler differ from the built-in 404 handling?**

> **Answer:** AEM's built-in error handling can serve a static error page, but it's global and not content-aware. ACS Error Page Handler resolves the appropriate error page based on the content hierarchy — different sections of the site can have different 404/500 pages. It supports pages at `/content/<site>/errors/<code>/` and renders them through AEM's full page rendering pipeline (with header, footer, analytics, etc.).

---

## ✅ Best Practices

1. **Use ACS for common problems** — don't reinvent Generic Lists, Error Pages, MCP
2. **Check ACS compatibility** with your AEM version before installing
3. **Only install features you use** — ACS has OSGi config toggles to disable unused features
4. **Use MCP for bulk operations** — not raw scripts on production
5. **Generic Lists for editor-managed dropdowns** — keeps development out of content decisions
6. **Redirect Map Manager** for URL redirect management — marketing teams can self-serve
7. **Use ACS in Cloud** — most features support AEM Cloud, but verify version compatibility

---

## 🛠️ Hands-on Practice

1. Install ACS AEM Commons on local AEM instance
2. Create a Generic List for "Page Category" with 5 values
3. Add a dropdown field to a component dialog that uses your Generic List as datasource
4. Configure the Error Page Handler to serve a custom `/content/mysite/en/errors/404` page for 404 errors
5. (Advanced) Write an MCP ProcessDefinition that lists all pages missing a meta description
