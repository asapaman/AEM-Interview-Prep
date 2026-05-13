# Day 1 — AEM Architecture
**Target:** 2–4 YOE | **Versions Covered:** AEM 6.5 & AEM as a Cloud Service (AEMaaCS)

---

## 🌟 What is AEM? (Start Here if You're New)

**Adobe Experience Manager (AEM)** is an enterprise-level **Content Management System (CMS)** built by Adobe. It allows organizations to create, manage, and deliver digital content (websites, mobile apps, emails, etc.) across multiple channels.

Think of AEM as a very powerful, highly customizable WordPress — but built for large enterprises like banks, airlines, and global brands.

### Why AEM over other CMS platforms?
- Handles **millions of pages** across multiple languages and regions
- Deep integration with Adobe Marketing Cloud (Analytics, Target, Campaign)
- Supports **headless delivery** (Content Fragments + GraphQL)
- Enterprise-grade security and workflow management
- Supports complex **multi-site management** (MSM)

---

## 🏗️ AEM Technology Stack

AEM is NOT built from scratch — it's assembled from several powerful open-source frameworks:

| Layer | Technology | What it does |
|-------|-----------|-------------|
| **Content Repository** | Apache Jackrabbit Oak (JCR) | Stores ALL content as a tree of nodes (like a file system but in a database) |
| **Resource Resolution** | Apache Sling | Maps HTTP URLs to JCR content nodes |
| **Service Container** | Apache Felix (OSGi) | Manages Java bundles — start, stop, update without restarting the JVM |
| **Persistence (6.5)** | TarMK / MongoMK | Actually stores the data on disk |
| **Persistence (Cloud)** | Azure Blob Storage + Oak Segment | Cloud-native storage |
| **Cache Layer** | Dispatcher (Apache httpd module) | Sits in front of AEM, caches rendered HTML |
| **Template Engine** | HTL (HTML Template Language) | Server-side rendering language (replaces JSP) |

### 🔑 Key Insight
> AEM = **JCR** (content storage) + **Sling** (URL routing) + **OSGi** (service management) + **Oak** (JCR implementation) + **HTL** (templating) + **Dispatcher** (caching)

---

## 🌐 AEM Instances: Author vs Publish vs Dispatcher

### The Big Picture
```
                          ┌─────────────────────────────────────────────┐
                          │              PRODUCTION SETUP               │
                          │                                             │
  [Content Editors]       │   [CDN]  →  [Dispatcher]  →  [Publish]     │
        ↓                 │                  ↑                          │
  [Author Instance] ──────│──────── Replication ─────────────────────   │
  (Internal only)         │                                             │
                          └─────────────────────────────────────────────┘
```

### Author Instance
- **Port:** 4502 (default)
- **Who uses it:** Content editors, developers, template authors
- **What happens here:**
  - Authors create and edit pages
  - Workflows (review/approval) happen here
  - Content is saved as **drafts** until published
  - Access is restricted — NOT public-facing
- **AEM 6.5:** Single author instance (can be clustered with MongoMK)
- **AEM Cloud:** Multiple author pods (high availability built-in)

### Publish Instance
- **Port:** 4503 (default)
- **Who uses it:** End users (website visitors)
- **What happens here:**
  - Serves live, published content
  - Read-heavy workload
  - Multiple instances for load balancing
- **AEM 6.5:** Manually configured publish farm
- **AEM Cloud:** Auto-scales based on traffic (Kubernetes pods)

### Dispatcher
- **What it is:** An Apache httpd module (NOT a separate Java process)
- **Key functions:**
  1. **Cache:** Stores rendered HTML files on disk. Serves them without hitting AEM.
  2. **Load Balancer:** Distributes requests across multiple Publish instances
  3. **Security:** Blocks requests to sensitive AEM paths (`/system/console`, `/crx/de`, `/libs/granite/core/content/login.html`)

---

## 🔄 Request Flow — Step by Step (VERY Important for Interviews)

```
User Browser
    │
    ▼
   CDN (Fastly / Akamai / CloudFront)
    │ Cache Hit? → Return cached response
    │ Cache Miss? → Forward to Dispatcher
    ▼
  Dispatcher (/var/cache/...)
    │ Cache Hit? → Return cached HTML file
    │ Cache Miss? → Forward to AEM Publish
    ▼
  AEM Publish (port 4503)
    │
    ▼
  Sling URL Decomposition
    │ /content/mysite/en/home.html
    │ → Resource Path: /content/mysite/en/home
    │ → Extension: html
    ▼
  ResourceResolver → Finds JCR Node
    │ /content/mysite/en/home [cq:Page]
    │   └── jcr:content [cq:PageContent]
    │         └── sling:resourceType = "mysite/components/page/homepage"
    ▼
  Sling Script Resolution
    │ Looks for: /apps/mysite/components/page/homepage/homepage.html
    ▼
  HTL Template Executes
    │ data-sly-use.model="com.mysite.HomepageModel"
    │ → Sling Model instantiated → @PostConstruct runs
    ▼
  HTML Response
    │ Cached by Dispatcher
    │ Cached by CDN
    ▼
  User sees the page! ✅
```

### Why is this important in interviews?
Interviewers ask: *"What happens when a user requests a page in AEM?"*
Walk through each step above with confidence.

---

## 🌳 JCR — Java Content Repository

### What is JCR?
JCR (JSR-283 specification) defines how content is stored. Think of it as a **hierarchical database** — content is organized as a **tree of nodes**, where each node has **properties**.

### Analogy
Imagine a file system:
- **Nodes** = Folders
- **Properties** = Files inside folders
- But each "folder" can have typed metadata (like a database record)

### JCR Node Structure for a Page
```
/content/                          ← Root of all site content
  └── mysite/                      [sling:OrderedFolder]
        └── en/                    [sling:OrderedFolder]
              └── home/            [cq:Page]          ← PAGE NODE
                    └── jcr:content [cq:PageContent]  ← PAGE PROPERTIES
                          ├── jcr:title = "Home Page"
                          ├── jcr:description = "Welcome to our site"
                          ├── sling:resourceType = "mysite/components/page/homepage"
                          ├── cq:template = "/conf/mysite/settings/wcm/templates/homepage"
                          └── root/                   [nt:unstructured]
                                └── hero/             [nt:unstructured] ← COMPONENT DATA
                                      ├── jcr:title = "Welcome!"
                                      └── sling:resourceType = "mysite/components/content/hero"
```

### Common Node Types
| Node Type | Used For |
|-----------|---------|
| `cq:Page` | AEM pages |
| `cq:PageContent` | Page properties (jcr:content node) |
| `nt:unstructured` | Component content nodes |
| `dam:Asset` | DAM assets (images, PDFs) |
| `dam:AssetContent` | Asset metadata |
| `cq:ClientLibraryFolder` | Clientlib folders |
| `sling:Folder` | Generic folders |
| `rep:User` | User accounts |
| `rep:Group` | User groups |

---

## ⚙️ Apache Sling — URL to Content Mapping

### What is Sling?
Sling is a **REST-based web framework** that maps HTTP URLs directly to JCR content. The URL IS the path to the content node.

### URL Decomposition
```
URL: /content/mysite/en/products/laptop.selector1.selector2.html/suffix/path

Breaking it down:
├── Resource Path: /content/mysite/en/products/laptop
├── Selectors:     selector1, selector2
├── Extension:     html
└── Suffix:        /suffix/path
```

### Real Examples
```
/content/mysite/home.html
→ Renders home page with HTML

/content/mysite/home.model.json
→ Returns home page data as JSON (Sling Model Exporter)

/content/dam/mysite/images/hero.img.800.600.png
→ Serves hero image resized to 800x600 (using selectors)
```

### Sling Script Resolution Order
When Sling receives a request for a resource with `sling:resourceType = "mysite/components/hero"`:
1. Look for `/apps/mysite/components/hero/hero.html` (exact match)
2. Look for `/apps/mysite/components/hero/GET.html`
3. Walk up `sling:resourceSuperType` chain
4. Fall back to default rendering

---

## 🔧 OSGi — Module System

### What is OSGi?
OSGi is a **dynamic module system for Java**. AEM's Java code is packaged as **OSGi Bundles** (JAR files with special metadata). These bundles can be:
- Started and stopped **without restarting the JVM**
- Versioned independently
- Wire services together via dependency injection

### OSGi Console (Important for Debugging!)
Access at: `http://localhost:4502/system/console`

Key pages:
- `/system/console/bundles` → See all bundles, check if ACTIVE
- `/system/console/components` → See all OSGi components
- `/system/console/configMgr` → OSGi configuration manager
- `/system/console/slinglog` → AEM logs configuration

---

## ☁️ AEM 6.5 vs AEM as a Cloud Service

| Aspect | AEM 6.5 | AEM as a Cloud Service |
|--------|---------|----------------------|
| **Infrastructure** | On-premise or Managed Services (Adobe or customer VMs) | Adobe-managed Kubernetes on Azure |
| **Updates** | Quarterly Service Packs | Continuous (auto-updated weekly) |
| **Scaling** | Manual — pre-provision servers | Automatic horizontal scaling |
| **Author HA** | Single author (or clustered with MongoMK) | Multiple author pods (built-in HA) |
| **Deployment** | Package Manager (CRX/DE, `crx-quickstart`) | Cloud Manager CI/CD pipeline ONLY |
| **Code Changes** | Can install packages via Package Manager | Immutable — only via pipeline |
| **Storage** | TarMK (file-based) or MongoMK (MongoDB) | Azure Blob + Oak Segment Compacted |
| **Dispatcher** | Self-managed Apache httpd | Adobe-managed (Fastly CDN built-in) |
| **Repository Browser** | CRXDE Lite (`/crx/de`) | Limited — via Cloud Manager console |
| **Service Users** | `ServiceUserMapper` OSGi config | Same, but stricter enforcement |

### Key Difference — Immutable Architecture (Cloud)
In AEM 6.5, you can:
- Install packages via Package Manager
- Edit files in CRXDE directly in production (bad practice but possible)
- Stop/start individual bundles manually

In AEM Cloud Service:
- **ALL code** must go through Cloud Manager CI/CD pipeline
- NO direct CRXDE access in production
- Code layer is **read-only** after deployment
- Only JCR content (authored content) is mutable

---

## ❓ Interview Questions & Detailed Answers

**Q1. Explain AEM architecture end-to-end. What happens when a user requests a page?**

> **Answer:** AEM architecture has several key layers. At the top is the CDN which caches content globally. Behind it is the Dispatcher — an Apache httpd module that caches rendered HTML and load-balances across multiple Publish instances. The Publish instance receives requests that aren't cached and processes them through Sling's URL decomposition to find the JCR content node. Sling then selects the appropriate HTL script based on the `sling:resourceType` of the content node. The script uses a Sling Model (Java POJO) to fetch and process data, then renders HTML which gets cached at the Dispatcher and CDN for subsequent requests.

**Q2. What is the role of the Dispatcher?**

> **Answer:** The Dispatcher serves three main purposes:
> 1. **Caching:** It stores rendered HTML files on the filesystem. When a user requests a page, if a cached version exists, it serves it directly without touching AEM — massively reducing load.
> 2. **Load Balancing:** It distributes incoming requests across multiple AEM Publish instances.
> 3. **Security:** It blocks access to sensitive AEM paths like `/system/console`, `/crx/de`, `/bin/` (unless explicitly allowed), and author-only paths.

**Q3. What is the difference between Author and Publish instances?**

> **Answer:** Author is where content editors work — creating pages, running workflows, managing assets. It's internal-only (never public). Publish is what end-users see — it serves live, published content and is optimized for high read throughput. When an editor "activates" a page on Author, it gets **replicated** to Publish via a Replication Agent.

**Q4. What is Sling and why is it important?**

> **Answer:** Sling is a REST-based web framework that maps HTTP URLs directly to JCR content. It performs URL decomposition (resource path, selectors, extension, suffix) and uses the `sling:resourceType` property of the resolved JCR node to find the correct rendering script. Without Sling, we'd need explicit routing rules like a traditional web framework. With Sling, the URL itself tells you where the content lives in the JCR.

**Q5. What is the JCR and how is content stored in it?**

> **Answer:** JCR (Java Content Repository, JSR-283) is a hierarchical content store. Content is stored as a tree of nodes, each with typed properties. For example, a page `/content/mysite/home` is a `cq:Page` node with a `jcr:content` child node of type `cq:PageContent` that stores properties like `jcr:title`, `sling:resourceType`, and all component data as nested nodes. It's like an XML tree stored in a database.

**Q6. What is TarMK vs MongoMK in AEM 6.5?**

> **Answer:**
> - **TarMK (Tar Micro Kernel):** File-based storage using TAR segment files. Simple, performant for a single AEM instance. Default for Author and single-server Publish. Data stored locally on disk.
> - **MongoMK (MongoDB Micro Kernel):** Stores JCR data in MongoDB. Supports **clustering** — multiple AEM Author/Publish instances sharing the same repository. Required for active-active author clustering. More complex to manage.

**Q7. What is the difference between AEM 6.5 and AEM as a Cloud Service?**

> **Answer:** Key differences:
> - **Deployment:** 6.5 uses Package Manager; Cloud uses Cloud Manager pipeline (immutable architecture)
> - **Scaling:** 6.5 requires manual scaling; Cloud auto-scales
> - **Updates:** 6.5 has quarterly service packs; Cloud is continuously updated
> - **Storage:** 6.5 uses TarMK/MongoMK; Cloud uses Azure Blob Storage
> - **Author:** 6.5 has single author; Cloud has multiple author pods for HA

**Q8. What are AEM run modes and how do they work?**

> **Answer:** Run modes configure AEM instance behavior for specific environments. Set via JVM argument `-Dsling.run.modes=publish,prod`. OSGi configs are stored in run-mode specific folders:
> - `/apps/mysite/osgiconfig/config/` → All environments
> - `/apps/mysite/osgiconfig/config.author/` → Author only
> - `/apps/mysite/osgiconfig/config.publish/` → Publish only
> - `/apps/mysite/osgiconfig/config.prod/` → Production only
> AEM applies the most specific matching config. Common run modes: `author`, `publish`, `dev`, `stage`, `prod`.

**Q9. What is a Replication Agent in AEM?**

> **Answer:** A Replication Agent is an AEM service that sends content from Author to Publish (or other targets). It:
> 1. Packages the activated content as a binary
> 2. Sends it via HTTP POST to the Publish instance's `/bin/receive` servlet
> 3. The Publish servlet unpacks and stores it in the JCR
> You can monitor replication queues at `Tools → Deployment → Replication → Agents on Author`. If the queue is blocked, published content won't appear on Publish.

**Q10. What is the `/content` vs `/apps` vs `/conf` vs `/libs` structure in AEM?**

> **Answer:**
> - `/content` — All site content (pages, assets). Mutable in both 6.5 and Cloud.
> - `/apps` — Custom code: components, templates, clientlibs. READ-ONLY in Cloud (deployed via pipeline). In 6.5, modifiable via Package Manager.
> - `/libs` — Adobe's OOTB code. NEVER modify directly — overlay via `/apps` instead.
> - `/conf` — Configuration: editable templates, policies, cloud configs. Mutable.
> - `/etc` — Legacy configs (deprecated in 6.5+, migrated to `/conf`).
> - `/home` — Users and groups.
> - `/var` — Runtime data: workflows, eventing, replication logs.

---

## ✅ Best Practices

1. **Never modify `/libs` directly** — Always overlay to `/apps` (mirror the path)
2. **Always use CDN + Dispatcher** in production — Never expose Publish directly to the internet
3. **Separate Author and Publish** on different JVMs — NEVER run both on the same instance in production
4. **Use run modes** for environment-specific OSGi configurations
5. **Avoid large binaries in JCR** — Use AEM Assets with external binary storage (S3DataStore / Azure Blob)
6. **Monitor Dispatcher cache hit ratio** — Low hit ratio means poor performance
7. **AEM Cloud:** Never rely on CRXDE for production changes — everything through pipeline

---

## 🛠️ Hands-on Practice

1. **Draw** the full AEM architecture diagram from memory (Author → Publish → Dispatcher → CDN)
2. **Navigate** JCR in CRXDE Lite (`/crx/de`) — explore `/content/mysite/` node structure
3. **Check** the Sling URL decomposition: access `http://localhost:4502/content/mysite/page.model.json` and explain each part
4. **Check** replication agents: `Tools → Deployment → Replication`
5. **View** OSGi bundles: `http://localhost:4502/system/console/bundles` — make sure your custom bundle is Active
