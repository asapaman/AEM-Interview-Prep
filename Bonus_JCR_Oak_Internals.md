# JCR & Oak Internals — AEM's Repository
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What is the JCR?

The **JCR (Java Content Repository)** is the standard Java API (JSR-283) for a hierarchical, tree-structured content store. AEM uses Apache Jackrabbit Oak as its JCR implementation.

Think of it as a **filesystem meets a database**:
- Hierarchical like a filesystem (paths, parent/child nodes)
- Typed like a database (property types, node type constraints)
- Versioned, queryable, and access-controlled

---

## 🌳 Core JCR Concepts

### Node

The fundamental data unit. Every node has:
- A **name** (e.g., `jcr:content`)
- A **primary type** (e.g., `cq:Page`)
- **Properties** (key-value pairs)
- **Child nodes**

```
/content/mysite/en/home          ← Node (name: "home")
  ├── jcr:primaryType = "cq:Page"    ← Property
  └── jcr:content/                    ← Child node (name: "jcr:content")
        ├── jcr:primaryType = "cq:PageContent"
        ├── jcr:title = "Home"
        ├── sling:resourceType = "mysite/components/page/homepage"
        └── root/                     ← Child node
```

### Property Types

| JCR Type | Java Type | Use Case |
|----------|-----------|---------|
| `STRING` | `String` | Text, paths, enumerations |
| `BOOLEAN` | `boolean` | Flags, toggles |
| `LONG` | `long` | Integers, counts, timestamps |
| `DOUBLE` | `double` | Decimal numbers |
| `DECIMAL` | `BigDecimal` | Currency, precise decimals |
| `DATE` | `Calendar` | Timestamps, scheduled dates |
| `BINARY` | `Binary`/`InputStream` | File content, images |
| `PATH` | `String` | JCR paths (not resolved) |
| `REFERENCE` | `String` (UUID) | Hard reference to a node |
| `WEAKREFERENCE` | `String` (UUID) | Soft reference (doesn't prevent delete) |
| `NAME` | `String` | JCR qualified names |
| `URI` | `String` | URI values |

### Namespaces

JCR uses XML-style namespaces to avoid naming conflicts:

| Prefix | Namespace URI | Used For |
|--------|--------------|---------|
| `jcr:` | `http://www.jcp.org/jcr/1.0` | Core JCR properties |
| `nt:` | `http://www.jcp.org/jcr/nt/1.0` | Node types |
| `mix:` | `http://www.jcp.org/jcr/mix/1.0` | Mixin types |
| `sling:` | `http://sling.apache.org/jcr/sling/1.0` | Sling framework |
| `cq:` | `http://www.day.com/jcr/cq/1.0` | AEM/CQ specific |
| `dam:` | `http://www.day.com/dam/1.0` | DAM assets |
| `rep:` | `internal` | Jackrabbit/Oak internal |
| `oak:` | `http://jackrabbit.apache.org/oak/ns/1.0` | Oak-specific |
| `granite:` | `http://www.adobe.com/jcr/granite/1.0` | Granite UI |

---

## 📋 Common Node Types

### AEM Page Structure

```
cq:Page
  └── jcr:content (cq:PageContent)
        ├── jcr:title
        ├── jcr:description
        ├── sling:resourceType
        ├── cq:template
        ├── cq:lastModified
        ├── cq:lastModifiedBy
        ├── cq:tags
        └── root/ (parsys or responsivegrid)
```

### DAM Asset Structure

```
dam:Asset
  └── jcr:content (dam:AssetContent)
        ├── metadata/ (nt:unstructured)
        └── renditions/ (nt:folder)
              └── original (nt:file)
                    └── jcr:content (nt:resource)
                          └── jcr:data [BINARY]
```

### Key Node Type Reference

| Node Type | `jcr:primaryType` | Description |
|-----------|------------------|-------------|
| `cq:Page` | `cq:Page` | AEM page |
| `cq:PageContent` | `cq:PageContent` | Page's content node |
| `cq:Component` | `cq:Component` | Component definition |
| `dam:Asset` | `dam:Asset` | DAM asset |
| `dam:AssetContent` | `dam:AssetContent` | Asset's content |
| `nt:unstructured` | `nt:unstructured` | Generic untyped node |
| `nt:file` | `nt:file` | File node |
| `nt:resource` | `nt:resource` | Binary resource |
| `nt:folder` | `nt:folder` | Folder node |
| `sling:Folder` | `sling:Folder` | Sling folder (sorted) |
| `sling:OrderedFolder` | `sling:OrderedFolder` | Order-preserving folder |
| `oak:QueryIndexDefinition` | `oak:QueryIndexDefinition` | Oak query index |

---

## 🏗️ Apache Jackrabbit Oak Architecture

Oak is AEM's JCR implementation. Understanding its storage model is essential for performance tuning.

```
Oak Architecture:

JCR API (javax.jcr.*)
     │
     ▼
Oak Core (NodeStore, CommitHook, IndexEditor)
     │
     ├── TarMK (Segment Store)   ← AEM 6.5 default for Author & Publish
     │     └── Storage: tar files on local filesystem
     │
     ├── MongoMK (Document Store) ← AEM 6.5 clustered Author
     │     └── Storage: MongoDB
     │
     └── Azure Blob Store         ← AEM Cloud Service
           └── Storage: Azure Blob Storage (Adobe managed)
```

### TarMK vs MongoMK

| Feature | TarMK (Segment Store) | MongoMK (Document Store) |
|---------|----------------------|------------------------|
| **Use case** | Single-node (Publish, standalone Author) | Clustered Author (multiple Author nodes) |
| **Storage** | Local `.tar` files | MongoDB |
| **Performance** | Faster reads | Horizontal scaling |
| **Compaction** | `oak-run.jar` revision cleanup | Automatic in MongoDB |
| **Backup** | File-based backup | mongodump |
| **AEM Cloud** | Not used | Not used (Azure Blob) |

---

## 🔍 Oak Index Types

Oak uses indexes to make queries fast. Without an index, every query does a **full repository traversal** (extremely slow).

| Index Type | Use Case | Notes |
|-----------|---------|-------|
| **Lucene** (full-text) | Full-text search, property queries | Most commonly used |
| **Property** | Simple property equality checks | Fast, lightweight |
| **Counter** | Counting nodes efficiently | Special use |
| **Reference** | `REFERENCE` property traversal | For reference lookups |
| **nodetype** | Node type queries | Built-in |
| **Solr** | External Solr integration | Rarely used now |

### Custom Lucene Index Definition

```xml
<!-- /oak:index/mysite-pages -->
<mysite-pages
    jcr:primaryType="oak:QueryIndexDefinition"
    type="lucene"
    async="async"
    compatVersion="{Long}2">

  <indexRules jcr:primaryType="nt:unstructured">
    <cq:Page jcr:primaryType="nt:unstructured">
      <properties jcr:primaryType="nt:unstructured">

        <!-- Index jcr:content/pageType for fast filtering -->
        <pageType jcr:primaryType="nt:unstructured"
                  name="jcr:content/pageType"
                  propertyIndex="{Boolean}true"
                  nodeScopeIndex="{Boolean}false"/>

        <!-- Index jcr:content/cq:tags for tag queries -->
        <tags jcr:primaryType="nt:unstructured"
              name="jcr:content/cq:tags"
              propertyIndex="{Boolean}true"/>

        <!-- Index jcr:content/jcr:title for full-text search -->
        <title jcr:primaryType="nt:unstructured"
               name="jcr:content/jcr:title"
               analyzed="{Boolean}true"   ← enables full-text
               nodeScopeIndex="{Boolean}true"/>

      </properties>
    </cq:Page>
  </indexRules>

</mysite-pages>
```

---

## 🔑 JCR Sessions — Working with the Repository

```java
import javax.jcr.*;

// Getting a Session from ResourceResolver
Session session = resourceResolver.adaptTo(Session.class);
// This session has the permissions of the current user/service user

// Reading a node
Node node = session.getNode("/content/mysite/en/home/jcr:content");
String title = node.getProperty("jcr:title").getString();
boolean exists = session.nodeExists("/content/mysite/en/home");

// Reading all properties of a node
PropertyIterator props = node.getProperties();
while (props.hasNext()) {
    Property prop = props.nextProperty();
    String name = prop.getName();
    if (!prop.isMultiple()) {
        String value = prop.getString();  // Works for most string types
    } else {
        Value[] values = prop.getValues();
    }
}

// Writing (requires commit/save)
Node contentNode = session.getNode("/content/mysite/en/home/jcr:content");
contentNode.setProperty("brandName", "My Site 2024");
session.save();  // ALWAYS save — changes are pending until save()

// Creating a new node
Node parent = session.getNode("/content/mysite/en/home/jcr:content");
Node newNode = parent.addNode("myNewNode", "nt:unstructured");
newNode.setProperty("title", "New Node");
session.save();

// Removing a node
session.getNode("/content/mysite/en/old-page").remove();
session.save();
```

---

## 🔄 JCR Versioning

AEM pages support versioning — every publish action creates a version.

```java
import javax.jcr.version.VersionManager;
import javax.jcr.version.VersionHistory;
import javax.jcr.version.Version;
import javax.jcr.version.VersionIterator;

Session session = resourceResolver.adaptTo(Session.class);
VersionManager versionManager = session.getWorkspace().getVersionManager();

String nodePath = "/content/mysite/en/home/jcr:content";

// Create a checkpoint (version)
Version version = versionManager.checkpoint(nodePath);
LOG.info("Created version: {}", version.getName());

// Get all versions
VersionHistory history = versionManager.getVersionHistory(nodePath);
VersionIterator versions = history.getAllVersions();
while (versions.hasNext()) {
    Version v = versions.nextVersion();
    LOG.info("Version: {} created at {}",
        v.getName(), v.getCreated().getTime());
}

// Restore a previous version
versionManager.restore(nodePath, "1.0", false);
session.save();
```

---

## 🔧 Oak Tools — Maintenance

```bash
# oak-run.jar — Oak's command-line utility for offline operations

# Check repository health
java -jar oak-run.jar check /path/to/segmentstore

# Compact the repository (reclaim disk space from deleted content)
# AEM 6.5 — Revision Cleanup (online, preferred)
# Tools → Operations → Maintenance → Revision Cleanup

# Offline compaction (AEM must be stopped)
java -jar oak-run.jar compact /path/to/segmentstore

# Recover corrupt repository
java -jar oak-run.jar recovery /path/to/segmentstore

# Generate index definition from content
java -jar oak-run.jar index --index-definitions /path/to/segmentstore
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is the difference between `nt:unstructured` and `cq:Page`?**

> **Answer:** `nt:unstructured` is the most flexible node type — it accepts any properties and any child nodes without type constraints. It's used for component data, configuration nodes, and any untyped content. `cq:Page` is a specific node type with defined structure — it must have a `jcr:content` child node of type `cq:PageContent`. AEM's Page Manager, replication, and authoring tools recognize `cq:Page` specifically. Use `cq:Page` for authored pages and `nt:unstructured` for configuration or data nodes.

**Q2. What is TarMK and when would you use MongoMK?**

> **Answer:** TarMK (Segment Node Store) stores content in `.tar` files on the local filesystem — it's fast, simple, and the default for Publish instances and standalone Author. MongoMK (Document Node Store) stores content in MongoDB — it enables multiple Author nodes to share the same repository (clustering), which TarMK can't do. Use MongoMK only when you need a clustered Author setup (multiple Author instances). In AEM Cloud, both are replaced by Adobe-managed Azure Blob Storage.

**Q3. Why do Oak queries sometimes cause repository traversal warnings?**

> **Answer:** If a query's predicate (filter) doesn't match any available Oak index, Oak falls back to scanning ALL nodes in the repository (traversal). This is extremely slow on large repositories and generates warning logs like "Traversal query: SELECT ... LIMIT ...". Fix by creating a `oak:QueryIndexDefinition` node that indexes the queried property. Tools like `/libs/granite/operations/content/diagnosys/queryPerformance.html` show slow queries.

**Q4. What is the difference between `REFERENCE` and `WEAKREFERENCE` property types?**

> **Answer:** Both store references to other nodes by UUID. `REFERENCE` is a hard reference — JCR prevents you from deleting the referenced node while a REFERENCE pointing to it exists (referential integrity). `WEAKREFERENCE` is a soft reference — the referenced node CAN be deleted even if WEAKREFERENCEs exist (they become dangling). Use REFERENCE for required relationships, WEAKREFERENCE for optional/informational links.

**Q5. What does `session.save()` do and what happens if you don't call it?**

> **Answer:** `session.save()` commits pending changes (new nodes, modified properties, deleted nodes) from the in-memory workspace to the persistent store (TarMK/MongoMK). Until `save()` is called, changes exist only in the session's in-memory workspace — they're NOT visible to other sessions and will be LOST if the session ends or the JVM crashes. In Sling Models and servlets, always call `session.save()` (or `resourceResolver.commit()`) after mutations.

---

## ✅ Best Practices

1. **Always call `session.save()` after mutations** — uncommitted changes are lost on session close
2. **Prefer `resourceResolver.commit()`** over `session.save()` in Sling code — works with Sling's resource API
3. **Create Oak indexes for frequently queried properties** — without them, queries traverse the entire repository
4. **Use `nt:unstructured` for component data** — don't define custom primary types unless absolutely necessary
5. **Run Revision Cleanup** regularly in AEM 6.5 — prevents segment store from growing unbounded
6. **Use `WEAKREFERENCE` for cross-content relationships** — hard `REFERENCE` makes content hard to delete
7. **Understand `async` indexes** — Lucene indexes are updated asynchronously (small delay after write)

---

## 🛠️ Hands-on Practice

1. In CRXDE, create a node at `/content/test/mynode` of type `nt:unstructured` with 5 properties of different types
2. Read that node programmatically in a Sling Model using both the JCR Session API and the Sling ValueMap API — compare the approaches
3. Write a Query Builder query that uses a property filter — then use `/libs/granite/operations/content/diagnosys/queryPerformance.html` to check if it uses an index
4. Create a custom Oak index definition for a custom property you added in step 1
5. (AEM 6.5) Find the Revision Cleanup task under Tools → Operations → Maintenance and run it
