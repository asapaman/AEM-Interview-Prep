# Groovy Console in AEM
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service (ACS AEM Tools)

---

## 🌟 What is the Groovy Console?

The **Groovy Console** is a tool (provided by the ACS AEM Tools or CQ Groovy Console open-source projects) that lets you execute **Apache Groovy** scripts directly against AEM's JCR repository in real-time — without writing, building, and deploying Java code.

> ⚠️ **CRITICAL:** The Groovy Console has direct, unrestricted access to AEM as an admin. **NEVER install it on production** without strict access control. One wrong script can delete content, break user accounts, or corrupt the repository.

### What Groovy Console Is Used For

1. **Content migration** — Move/rename/restructure existing content nodes
2. **Bulk updates** — Update a property across thousands of pages
3. **Data fixing** — Fix corrupt or incorrect data introduced by a bug
4. **Analysis** — Query and report on JCR content
5. **Testing** — Quickly test a service or API without redeploying
6. **Tag management** — Create, merge, or reorganize tags

### Installation

```bash
# Via ACS AEM Tools (includes many useful tools including Groovy-like scripts)
# Download: https://github.com/Adobe-Consulting-Services/acs-aem-commons/releases

# OR: CQ Groovy Console (dedicated Groovy scripting)
# https://github.com/orbinson/aem-groovy-console
# Install package via Package Manager
# Access at: /groovyconsole
```

---

## 🔑 Groovy Console Built-in Bindings

When you write a Groovy script in the console, these objects are available by default:

| Variable | Type | Description |
|----------|------|-------------|
| `session` | `javax.jcr.Session` | JCR Session (admin privileges) |
| `resourceResolver` | `ResourceResolver` | Sling ResourceResolver |
| `pageManager` | `PageManager` | WCM Page Manager |
| `queryManager` | `QueryManager` | JCR Query Manager |
| `slingRepository` | `SlingRepository` | Sling Repository service |
| `request` | `SlingHttpServletRequest` | Current HTTP Request |
| `response` | `SlingHttpServletResponse` | Current HTTP Response |
| `log` | `Logger` | SLF4J Logger |
| `out` | `PrintStream` | Script output (shown in console) |

---

## 💻 Groovy Script Examples — From Basic to Advanced

### Example 1: Read a Page's Properties

```groovy
// Read properties of a page
def page = pageManager.getPage('/content/mysite/en/home')

if (page) {
    println "Title: ${page.getTitle()}"
    println "Path: ${page.getPath()}"
    println "Last Modified: ${page.getLastModified()?.getTime()}"
    println "Template: ${page.getTemplate()?.getPath()}"

    def props = page.getProperties()
    println "jcr:title: ${props.get('jcr:title', String)}"
    println "sling:vanityPath: ${props.get('sling:vanityPath', String)}"
} else {
    println "Page not found!"
}
```

### Example 2: Bulk Update a Property Across All Pages

```groovy
// Update "brand" property on all pages under /content/mysite
def contentRoot = resourceResolver.getResource('/content/mysite')
int count = 0
int errors = 0

def updatePages
updatePages = { Resource resource ->
    // Check if it's a page's jcr:content node
    if (resource.resourceType == 'cq:PageContent') {
        def valueMap = resource.adaptTo(org.apache.sling.api.resource.ModifiableValueMap)
        if (valueMap != null) {
            valueMap.put('brandName', 'My Site 2024')  // Add/update property
            count++
        }
    }

    // Recurse into children
    resource.listChildren().each { child ->
        try {
            updatePages(child)
        } catch (Exception e) {
            println "Error on ${child.path}: ${e.message}"
            errors++
        }
    }
}

updatePages(contentRoot)

// IMPORTANT: Commit the changes!
resourceResolver.commit()

println "✅ Updated: ${count} nodes"
println "❌ Errors:  ${errors} nodes"
```

### Example 3: Find and Delete Orphaned Nodes

```groovy
// Find all draft pages that were never published and are older than 90 days
import java.util.Calendar

def draftPagesRoot = pageManager.getPage('/content/mysite')
def ninetyDaysAgo = Calendar.getInstance()
ninetyDaysAgo.add(Calendar.DAY_OF_YEAR, -90)

def orphanedPaths = []

def checkPage
checkPage = { page ->
    def props = page.getProperties()
    def lastModified = props.get('cq:lastModified', Calendar)

    boolean isDraft = !props.containsKey('cq:lastPublished')  // Never published
    boolean isOld = lastModified != null && lastModified.before(ninetyDaysAgo)

    if (isDraft && isOld) {
        orphanedPaths << page.getPath()
        println "Orphaned: ${page.getPath()} (last modified: ${lastModified?.getTime()})"
    }

    // Check children
    page.listChildren().each { child ->
        checkPage(child)
    }
}

checkPage(draftPagesRoot)

println "\nTotal orphaned pages: ${orphanedPaths.size()}"

// UNCOMMENT to actually delete (test with dry run first!):
// orphanedPaths.each { path ->
//     pageManager.delete(pageManager.getPage(path), false)
// }
// resourceResolver.commit()
// println "Deleted ${orphanedPaths.size()} pages"
```

### Example 4: Query Builder via Groovy

```groovy
// Find all pages using a specific template
import com.day.cq.search.QueryBuilder
import com.day.cq.search.PredicateGroup

def queryBuilder = sling.getService(QueryBuilder)
def searchMap = [
    'type'      : 'cq:Page',
    'path'      : '/content/mysite',
    'property'  : 'jcr:content/cq:template',
    'property.value': '/conf/mysite/settings/wcm/templates/homepage',
    'p.limit'   : '-1',  // All results
    'p.offset'  : '0'
]

def query = queryBuilder.createQuery(PredicateGroup.create(searchMap), session)
def result = query.getResult()

println "Found ${result.getTotalMatches()} pages using homepage template:\n"
result.getHits().each { hit ->
    println "  ${hit.getPath()}"
}
```

### Example 5: Content Replication (Activate Pages)

```groovy
// Activate (publish) a list of pages
import com.day.cq.replication.ReplicationActionType
import com.day.cq.replication.Replicator

def replicator = sling.getService(Replicator)
def pagesToActivate = [
    '/content/mysite/en/home',
    '/content/mysite/en/about',
    '/content/mysite/en/products'
]

pagesToActivate.each { path ->
    try {
        replicator.replicate(session, ReplicationActionType.ACTIVATE, path)
        println "✅ Activated: ${path}"
    } catch (Exception e) {
        println "❌ Failed to activate ${path}: ${e.message}"
    }
}
```

### Example 6: Tag Operations

```groovy
// Create a new tag and apply it to all pages in a section
import com.day.cq.tagging.TagManager

def tagManager = resourceResolver.adaptTo(TagManager)

// Create tag
def newTag = tagManager.createTag(
    'mysite:campaigns/summer-2024',  // Tag ID
    'Summer 2024',                    // Tag title
    'Summer 2024 Campaign Tag',       // Description
    true                              // Auto-save
)
println "Created tag: ${newTag?.getPath()}"

// Apply tag to all pages under /content/mysite/en/campaign/summer-2024
def campaignPage = pageManager.getPage('/content/mysite/en/campaign/summer-2024')
if (campaignPage) {
    def children = campaignPage.listChildren()
    int taggedCount = 0

    children.each { Page child ->
        def resource = child.getContentResource()
        if (resource) {
            def mvm = resource.adaptTo(org.apache.sling.api.resource.ModifiableValueMap)
            if (mvm) {
                // Tags are stored as a String array
                def existingTags = mvm.get('cq:tags', new String[0])
                def tagList = new ArrayList<>(Arrays.asList(existingTags))
                if (!tagList.contains('mysite:campaigns/summer-2024')) {
                    tagList.add('mysite:campaigns/summer-2024')
                    mvm.put('cq:tags', tagList.toArray(new String[0]))
                    taggedCount++
                }
            }
        }
    }

    resourceResolver.commit()
    println "Tagged ${taggedCount} pages"
}
```

### Example 7: OSGi Service Usage

```groovy
// Invoke any OSGi service from Groovy console
import com.mysite.core.services.ProductSyncService

def productSyncService = sling.getService(ProductSyncService)
if (productSyncService) {
    def result = productSyncService.syncProducts()
    println "Sync result: ${result}"
} else {
    println "ProductSyncService not available — is the bundle active?"
}
```

---

## 🔍 Useful Groovy Console Patterns

### Dry Run Pattern (Always Do This First!)

```groovy
boolean DRY_RUN = true  // Set to false to actually make changes

def changes = []

// Find what would be changed
changes << "/content/mysite/en/home → would update title"
changes << "/content/mysite/en/about → would update title"

if (DRY_RUN) {
    println "=== DRY RUN — No changes made ==="
    changes.each { println "  Would: ${it}" }
    println "Total: ${changes.size()} changes pending"
} else {
    println "=== LIVE RUN — Making changes ==="
    // Actual modification code here
    resourceResolver.commit()
    println "Done: ${changes.size()} changes applied"
}
```

### Pagination for Large Datasets

```groovy
// Process large content sets in batches to avoid memory issues
int BATCH_SIZE = 100
int offset = 0
int processed = 0

while (true) {
    def query = queryBuilder.createQuery(PredicateGroup.create([
        'type'    : 'cq:Page',
        'path'    : '/content/mysite',
        'p.limit' : BATCH_SIZE.toString(),
        'p.offset': offset.toString()
    ]), session)

    def hits = query.getResult().getHits()
    if (hits.isEmpty()) break  // No more results

    hits.each { hit ->
        // Process each page
        processed++
    }

    offset += BATCH_SIZE
    session.save()  // Commit batch
    println "Processed ${processed} pages so far..."
}

println "Total processed: ${processed}"
```

---

## ☁️ Groovy Console in AEM Cloud

> ⚠️ **AEM Cloud Consideration:** In AEM Cloud, Groovy Console should **NOT be installed on production** (Adobe's security policies may prevent it). Use it on:
> - Local SDK (development)
> - Dev/Stage environments (with restricted access)

**Alternatives for AEM Cloud production data fixes:**
1. **Content Migration scripts** via `oak-run.jar` (offline)
2. **AEM Cloud Migration scripts** (Adobe-provided tooling)
3. **Sling Maintenance Tasks** for scheduled cleanup
4. **Custom OSGi Maintenance Tasks** deployed via Cloud Manager

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is the Groovy Console and when would you use it?**

> **Answer:** Groovy Console is a scripting tool (from ACS AEM Tools or CQ Groovy Console) that lets you execute Groovy scripts against AEM's JCR repository in real-time as an admin. Use it for: emergency data fixes in dev/staging, content migrations (move/rename nodes), bulk property updates, ad-hoc analysis, and testing OSGi services without redeployment. **Never install on production without strict access control.**

**Q2. What built-in variables does the Groovy Console provide?**

> **Answer:** `session` (JCR admin session), `resourceResolver` (Sling ResourceResolver), `pageManager` (WCM PageManager for page CRUD), `queryManager` (JCR query manager), `slingRepository`, `request`, `response`, `log` (SLF4J logger), and `out` (print output to console UI).

**Q3. Why is a dry-run pattern important in Groovy scripts?**

> **Answer:** Groovy scripts run with admin privileges and can modify or delete content across the entire repository. A dry-run first identifies all affected nodes and logs what WOULD be changed, without committing. This lets you review and validate the scope before execution. One incorrect GROQ/XPath query without a dry-run could inadvertently affect thousands of nodes with no undo. Always dry-run first, especially on any non-localhost environment.

**Q4. How do you call a custom OSGi service from Groovy Console?**

> **Answer:** Use `sling.getService(YourService.class)` where `YourService` is the service interface class. The console has access to the OSGi service registry via the `sling` object. This is useful for testing a service's behavior without writing test code: `def myService = sling.getService(com.mysite.core.services.MyService); println myService.doSomething()`.

---

## ✅ Best Practices

1. **Always dry-run first** — print affected paths before making changes
2. **Process in batches** — don't query thousands of nodes without pagination
3. **Commit in batches** — call `session.save()` every N iterations for large datasets
4. **Log everything** — use `println` generously for audit trail
5. **NEVER install on production** — or if you must, restrict access to admins only
6. **Test on local/dev first** — verify the script works before running on stage
7. **Keep scripts version-controlled** — save migration scripts in your project's `/scripts` folder

---

## 🛠️ Hands-on Practice

1. Install CQ Groovy Console package on local AEM instance
2. Write a script that lists all pages under `/content/mysite` with their last modified date
3. Write a bulk update script that adds a `siteName=mysite` property to all `jcr:content` nodes — DRY RUN first
4. Write a script that finds all assets in `/content/dam/mysite` larger than 5MB
5. Write a script that moves pages from `/content/mysite/old-path` to `/content/mysite/new-path` using `PageManager.move()`
