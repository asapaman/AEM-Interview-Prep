# Day 1 — AEM Architecture
**Difficulty:** Medium | **4 YOE Focus**

---

## 📖 Topic Explanation

AEM (Adobe Experience Manager) is a Java-based CMS built on top of several open-source technologies:

| Layer | Technology | Role |
|-------|-----------|------|
| Content Repository | Apache Jackrabbit Oak (JCR) | Stores all content as nodes/properties |
| Resource Resolution | Apache Sling | Maps URLs to JCR content |
| Service Container | Apache Felix (OSGi) | Manages Java bundles/services |
| Persistence | MongoDB / TarMK / DocumentMK | Actual data storage |
| Cache Layer | Dispatcher (Apache httpd module) | Caches HTML pages close to user |

### Request Flow (Critical to Know!)
```
Browser → CDN → Dispatcher → AEM Publish → Sling → JCR → Sling Model → HTL → HTML Response
```

**Detailed Steps:**
1. User hits URL → CDN checks its cache
2. Cache miss → CDN forwards to **Dispatcher**
3. Dispatcher checks `.cache` folder (file-based cache)
4. Cache miss → Dispatcher forwards to **AEM Publish** instance
5. Sling **ResourceResolver** resolves URL to JCR node
6. Sling selects correct **Servlet/Script** (HTL/JSP)
7. **Sling Model** is instantiated and adapts from Resource/Request
8. HTL renders HTML using model's data
9. Response travels back → cached at Dispatcher → cached at CDN

---

## 🏗️ Author vs Publish vs Dispatcher

```
[Author Instance]          [Publish Instance(s)]       [Dispatcher]
  - Content creation   →    - Content delivery      →    - Caching layer
  - Drafts/workflows        - Read-heavy                  - Load balancer
  - Port 4502               - Port 4503                   - Apache httpd mod
  - /author path            - /content path
```

**Key Differences:**
- Author: Where editors create/edit pages. Always password-protected.
- Publish: What end users see. Can be multiple instances for scaling.
- Dispatcher: Apache httpd with AEM module. Caches, load-balances, blocks author paths.

---

## 🔑 OSGi / Felix

- OSGi = Open Services Gateway initiative — a **modular Java runtime**
- AEM deploys code as **OSGi Bundles** (JAR files with special manifest)
- Bundles can be **started/stopped without restarting JVM**
- Services are registered/injected via `@Component`, `@Service`, `@Reference`

---

## 🌳 JCR & Oak

- JCR = **Java Content Repository** (JSR-170/283 spec)
- Content is stored as a **tree of nodes**, each with **properties**
- Oak is Apache's JCR implementation used by AEM 6.x+
- Common node types: `cq:Page`, `cq:PageContent`, `nt:unstructured`, `dam:Asset`

**Sample JCR Structure:**
```
/content/mysite/en/homepage   [cq:Page]
  └── jcr:content             [cq:PageContent]
        ├── jcr:title = "Home"
        ├── sling:resourceType = "mysite/components/page"
        └── root              [nt:unstructured]
              └── hero        [nt:unstructured]
                    └── jcr:title = "Welcome"
```

---

## ❓ Interview Questions & Answers

**Q1. Explain AEM architecture end-to-end.**
> Start from the layers: JCR for storage, Sling for URL resolution, OSGi for services, Oak for persistence. Explain Author→Publish replication, then request flow through Dispatcher→Publish→Sling→HTL. Mention CDN at the edge.

**Q2. What is the role of the Dispatcher?**
> Dispatcher is an Apache httpd module that acts as a **reverse proxy and cache**. It caches rendered HTML pages to avoid hitting AEM for every request. It also provides load balancing across multiple publish instances and security (blocking access to `/system`, `/bin`, author paths).

**Q3. What is Sling URL decomposition?**
> Sling decomposes a URL like `/content/site/page.model.json` into:
> - **Resource path**: `/content/site/page`
> - **Selector**: `model`
> - **Extension**: `json`
> Sling uses this to find the right script (e.g., `model.json.jsp` or Servlet registered for selector+extension).

**Q4. How does Author-Publish replication work?**
> When content is activated on Author, AEM creates a **Replication Agent** that sends the content package to the Publish instance(s) via HTTP POST. Reverse replication sends user-generated content from Publish back to Author.

**Q5. What is the difference between TarMK and MongoMK?**
> - **TarMK**: File-based storage (TAR files). Simple, fast for single-publish. Default for Author.
> - **MongoMK**: MongoDB-based. Supports clustering (multiple AEM instances sharing same repo). Used for Publish farms.

**Q6. What are AEM run modes?**
> Run modes configure AEM behavior for specific environments: `author`, `publish`, `dev`, `stage`, `prod`. They control which OSGi configs are active. Set via `-Dsling.run.modes=publish,prod` JVM arg or folder naming (`config.publish`, `config.prod`).

---

## ✅ Best Practices

- Always use **Dispatcher** in front of Publish — never expose Publish directly
- Use **CDN** (Fastly, Akamai, CloudFront) in front of Dispatcher
- Keep **Author and Publish on separate JVMs** — never run both on same instance in prod
- Use run modes (`config.author`, `config.publish`) for environment-specific OSGi configs
- Avoid storing large binaries in JCR — use **AEM Assets + Binary Store** (FileDataStore/S3DataStore)
- Use **Oak indexes** properly to avoid full traversal queries

---

## 🛠️ Hands-on Task

Draw this architecture diagram from memory and explain each component's role:
```
[CDN] → [Dispatcher] → [AEM Publish 1]  ←replication← [AEM Author]
                    ↘ [AEM Publish 2]
```
