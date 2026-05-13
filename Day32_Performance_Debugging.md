# Performance & Debugging in AEM
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 Why Performance Debugging Matters

Performance issues in AEM manifest as slow page loads, OOM errors, unresponsive Author UI, and publish failures. Understanding how to diagnose these systematically separates junior developers from senior ones.

---

## 🖥️ Felix Web Console — The Control Center

The **Apache Felix Web Console** at `/system/console` is your primary diagnostic tool. It's available only on Author (and Publish if unrestricted — BLOCK at Dispatcher!).

### Key Felix Console Tabs

| Path | Purpose |
|------|---------|
| `/system/console/components` | View all OSGi components, their status (active/failed) |
| `/system/console/bundles` | View all OSGi bundles, start/stop/install |
| `/system/console/configMgr` | OSGi configuration (read/write) |
| `/system/console/services` | All registered OSGi services |
| `/system/console/jmx` | JMX MBeans for monitoring |
| `/system/console/slinglog` | Log file viewer and configuration |
| `/system/console/requests` | Recent request log |
| `/system/console/status-slingjmodelregistry` | All registered Sling Models |
| `/system/console/status-Configurations` | All OSGi configs (effective) |
| `/system/console/productinfo` | AEM version, bundle versions |

### Diagnosing OSGi Component Failures

```
/system/console/components
→ Filter by "failed" status
→ Click on a failed component to see WHY it failed:
  - "Unsatisfied @Reference" → missing dependency (another bundle failed)
  - "Configuration not bound" → missing OSGi config file
  - "Exception in @Activate" → initialization error

Common failure cascade:
MyServiceImpl fails (missing @Reference SomeOtherService)
→ AnotherServiceImpl fails (missing @Reference MyService)
→ MyServlet fails (missing @Reference AnotherService)
```

---

## 📊 AEM Performance Tools

### 1. Query Performance Tool

```
/libs/granite/operations/content/diagnosys/queryPerformance.html
```

Shows all recent JCR queries sorted by execution time. Use this to find:
- Slow queries (> 100ms = problematic)
- Queries causing traversal (no index)
- Queries with large result sets

### 2. Request Analyzer

```
/system/console/requests
```

Shows recent HTTP requests with timing, URL, response code, and resource resolution.

### 3. System Overview

```
/libs/granite/operations/content/systemoverview.html
```

Shows memory usage, thread count, replication queue state, and system health.

### 4. Log Tailing

```
/system/console/slinglog

→ Add custom log level for your package:
  Logger: com.mysite.core
  Level: DEBUG
  Log File: logs/mysite.log

→ Tail in terminal:
  tail -f crx-quickstart/logs/error.log | grep "com.mysite"
```

---

## 🐛 Debugging Sling Models — Step by Step

When a component shows empty/incorrect output:

```
STEP 1: Check component is rendering at all
─────────────────────────────────────────────
View page source — is component HTML present?
If empty: Check if data-sly-test caused it to be skipped

STEP 2: Check model is adapting
─────────────────────────────────────────────
Add to HTL temporarily:
<p>Adapted: ${model != null}</p>
<p>Title: ${model.title}</p>

STEP 3: Check @Model registration
─────────────────────────────────────────────
/system/console/status-slingjmodelregistry
→ Search for your model class
→ Is it registered? What resourceType/adaptable does it list?

STEP 4: Check injection
─────────────────────────────────────────────
Add logging in @PostConstruct:
LOG.debug("title={}, ctaLink={}", title, ctaLink);

Check logs at /system/console/slinglog

STEP 5: Check resource properties
─────────────────────────────────────────────
CRXDE → Navigate to the component's JCR node
→ Verify the property names match @ValueMapValue field names exactly
```

---

## 🔍 Slow Query Debugging

```
SYMPTOM: Slow page renders, "Traversal query" warnings in error.log

STEP 1: Find the slow query
─────────────────────────────────────────────
/libs/granite/operations/content/diagnosys/queryPerformance.html
→ Sort by "Execution time"
→ Look for queries taking > 100ms or causing traversal

Error.log pattern:
"Traversal query: [SELECT * FROM [cq:Page] ...]  Limit: 100000"

STEP 2: Analyze the query execution plan
─────────────────────────────────────────────
Add &explain=true to QueryBuilder URL:
/bin/querybuilder.json?type=cq:Page&property=jcr:content/customProp&property.value=test&explain=true

OR use CRXDE → Query tab → type "JCR-SQL2" → run EXPLAIN

STEP 3: Check existing indexes
─────────────────────────────────────────────
/libs/granite/operations/content/diagnosys/queryPerformance.html
→ Click "Index Rules" to see all indexes

OR in CRXDE: browse /oak:index/ to see all index definitions

STEP 4: Create an index for the missing property
─────────────────────────────────────────────
Create an oak:QueryIndexDefinition node under /oak:index/
(see JCR_Oak_Internals.md for full example)

STEP 5: Reindex
─────────────────────────────────────────────
Set reindex=true on the index definition
AEM will async-reindex the repository (check /system/console/jmx → IndexStats)
```

---

## 🧠 Memory Leak Detection

Memory leaks in AEM are commonly caused by:

### Cause 1: Unclosed ResourceResolvers

```java
// ❌ MEMORY LEAK — resolver never closed if exception thrown
ResourceResolver resolver = resolverFactory.getServiceResourceResolver(params);
Resource r = resolver.getResource("/content/mysite");
// Exception thrown here → resolver never closed!
doSomething(r);
resolver.close();

// ✅ CORRECT — try-with-resources ensures close even on exception
try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(params)) {
    Resource r = resolver.getResource("/content/mysite");
    doSomething(r);
}  // resolver.close() called automatically, even if exception thrown

// Detection:
// /system/console/jmx → ResourceResolverFactory → count of open resolvers
// Should be close to 0 when server is idle
```

### Cause 2: Unclosed JCR Sessions

```java
// ❌ Leak
Session session = repository.loginAdministrative(null);
// Exception thrown → session never logged out!
session.logout();

// ✅ Correct
Session session = null;
try {
    session = repository.loginService("my-subservice", null);
    // use session
} finally {
    if (session != null && session.isLive()) {
        session.logout();
    }
}
```

### Cause 3: JCR Observation Session Not Closed

```java
// In @Deactivate — MUST logout the observation session
@Deactivate
protected void deactivate() {
    if (observationSession != null && observationSession.isLive()) {
        try {
            observationSession.getWorkspace()
                              .getObservationManager()
                              .removeEventListener(eventListener);
        } catch (RepositoryException e) { LOG.warn("Error removing listener", e); }
        observationSession.logout();
    }
}
```

### Cause 4: Static Caches That Never Expire

```java
// ❌ Unbounded cache — grows forever
private static final Map<String, Object> cache = new HashMap<>();  // Never cleared!

// ✅ Bounded cache with TTL (using Guava or Caffeine)
private final Cache<String, Object> cache = CacheBuilder.newBuilder()
    .maximumSize(1000)              // Max 1000 entries
    .expireAfterWrite(10, TimeUnit.MINUTES)  // TTL: 10 minutes
    .build();

// ✅ Always clear in @Deactivate
@Deactivate
protected void deactivate() {
    cache.invalidateAll();
}
```

---

## 📈 JVM Memory Analysis

### Reading Heap Dumps

```bash
# Trigger a heap dump (AEM 6.5)
# 1. Via JMX:
/system/console/jmx → java.lang:type=Memory → Operations → dumpHeap

# 2. Via command line (while AEM is running):
jmap -dump:format=b,file=/tmp/aem-heap.hprof <AEM-PID>

# 3. On OutOfMemoryError (configure in start.sh):
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/opt/aem/heap-dumps/

# Analyze heap dump:
# Eclipse Memory Analyzer (MAT): https://www.eclipse.org/mat/
# Open heap dump → Run "Leak Suspects" report
# Look for: large retention trees, resource resolvers, sessions, caches
```

### Thread Dump Analysis

```bash
# Generate thread dump (identify deadlocks, blocked threads)
kill -3 <AEM-PID>       # Sends SIGQUIT — dumps threads to console output
jstack <AEM-PID>        # More detailed thread dump
jstack -l <AEM-PID>     # Include lock info

# AEM also has thread dumps at:
/system/console/requests → "Recent Requests" → look for long-running requests

# Common thread states:
# RUNNABLE  → Executing (OK if brief)
# BLOCKED   → Waiting for a lock (potential deadlock)
# WAITING   → Waiting for notify (typically idle threads in pools)
# TIMED_WAITING → sleep/wait with timeout (usually OK)

# Red flags:
# Many threads BLOCKED on the same lock → deadlock/contention
# Sling request threads stuck > 30s → slow query or external API call
```

---

## 📋 AEM Error Log — Common Patterns

```bash
# Common error patterns and their meanings:

# 1. Session/ResourceResolver leak
"Could not cleanly logout ResourceResolver: ..."
→ A ResourceResolver was not closed → find where it's opened without close()

# 2. Index traversal
"Traversal query (query without index): SELECT * FROM..."
→ Create an Oak index for the queried property

# 3. Service user not found
"Failed to login service: java.security.LoginException: Unable to get User"
→ Service user not created, or ServiceUserMapper mapping is wrong

# 4. Bundle wiring failure
"Uses constraint violation: ..."
→ Two bundles exporting the same package with different versions → check pom.xml deps

# 5. Replication failure
"Replication failed for path: /content/... Cannot connect to..."
→ AEM Publish is down, or Dispatcher is blocking the replication

# 6. Workflow engine backlog
"Workflow engine is backed up with N waiting items"
→ Workflow steps are slow or stuck → check via /libs/cq/workflow/admin/console

# 7. Memory pressure
"GC overhead limit exceeded" or "OutOfMemoryError: Java heap space"
→ Heap dump needed → look for cache leaks, unclosed resolvers
```

---

## 🔧 AEM Debugging Checklist

```
COMPONENT NOT RENDERING:
[ ] Is the sling:resourceType correct in CRXDE?
[ ] Is the HTL file named correctly (component name + .html)?
[ ] Is the model class registered? (/system/console/status-slingjmodelregistry)
[ ] Check error.log for NullPointerException in @PostConstruct
[ ] Add LOG.debug statements to @PostConstruct to trace injection

PAGE LOADS SLOWLY:
[ ] Check /libs/granite/operations/content/diagnosys/queryPerformance.html
[ ] Look for traversal queries → create Oak indexes
[ ] Check if external API calls in @PostConstruct have timeouts
[ ] Check thread dumps for blocked threads
[ ] Check if Dispatcher cache is working (look for X-Cache headers)

MEMORY/OOM ERROR:
[ ] Check /system/console/jmx → Memory → HeapMemoryUsage
[ ] Check for unclosed ResourceResolvers (open resolver count > idle baseline)
[ ] Check for unbounded static caches
[ ] Generate heap dump and analyze with Eclipse MAT
[ ] Check for JCR Observation session leaks

OSGi COMPONENT FAILED:
[ ] /system/console/components → filter by failed
[ ] Check which @Reference is unsatisfied
[ ] Check if referenced bundle is active (/system/console/bundles)
[ ] Check for @Activate exceptions in error.log
[ ] Check if OSGi config file is deployed correctly
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. How do you diagnose a slow AEM page response?**

> **Answer:** Systematic approach: 1) Check if Dispatcher is caching the page (add `?wcmmode=disabled` to bypass AEM authoring mode). 2) If uncached, add timing logs in the Sling Model `@PostConstruct`. 3) Check `queryPerformance.html` for slow/traversal queries. 4) Check for external API calls without timeouts in components. 5) Thread dump to see if Sling request threads are blocked. 6) Check `/system/console/requests` for recent slow requests. The most common cause is a missing Oak index causing full repository traversal.

**Q2. How do you detect a ResourceResolver leak?**

> **Answer:** Check `/system/console/jmx → ResourceResolverFactory → OpenResolverCount`. When the server is idle, this should be close to 0. If it keeps growing, something is opening resolvers without closing them. To find the leak: temporarily set log level to DEBUG for `org.apache.sling.resourceresolver` — it logs resolver open/close. Look for resolvers opened but no corresponding close. Common cause: resolver opened in a service without `try-with-resources`.

**Q3. What does a "Traversal query" warning in the log mean?**

> **Answer:** It means a JCR query ran without using any Oak index — Oak had to scan ALL nodes matching the path to find results. This is catastrophic for performance on large repositories (can take seconds to minutes for a query that should take milliseconds). Fix: create an `oak:QueryIndexDefinition` that indexes the property being queried. Use `queryPerformance.html` to identify the slow query, then `explain=true` in QueryBuilder to see which index (if any) it's using.

**Q4. What causes "OutOfMemoryError: Java heap space" in AEM?**

> **Answer:** Common causes: 1) Unclosed ResourceResolvers/Sessions accumulating. 2) Unbounded caches (static Maps/Lists that grow without eviction). 3) Very large content trees loaded into memory (large query results). 4) JCR Observation sessions not cleaned up in `@Deactivate`. 5) Binary assets loaded into memory without streaming. Diagnose with `-XX:+HeapDumpOnOutOfMemoryError` and analyze the dump with Eclipse MAT to find the largest retention trees.

---

## ✅ Best Practices

1. **Always use `try-with-resources`** for ResourceResolver and Session
2. **Set timeouts on all external HTTP calls** — prevent hung threads
3. **Cache OSGi service results** — not the ResourceResolver itself
4. **Set `p.limit` on all queries** — unbounded queries can OOM
5. **Create Oak indexes** for all frequently queried properties
6. **Enable GC logging** in AEM start script for memory analysis: `-Xloggc:/opt/aem/logs/gc.log`
7. **Monitor open resolvers** in JMX during load testing — detect leaks early

---

## 🛠️ Hands-on Practice

1. Write a Sling Model with a deliberate ResourceResolver leak — observe the growing count in JMX — then fix it
2. Write a Query Builder query without an index — observe traversal warning in logs — create an index — verify query now uses it
3. Navigate to `/system/console/components` — find any failed components on your local AEM — diagnose why they failed
4. Configure a custom log level for `com.mysite.core` at DEBUG and watch your component's `@PostConstruct` logs in real-time
5. Generate a thread dump while AEM is under load — identify which threads are doing what
