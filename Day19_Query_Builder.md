# Query Builder — AEM's Content Search API
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What is Query Builder?

The **AEM Query Builder** is a Java API and REST interface that provides a simple, AEM-specific way to search for content in the JCR repository. It abstracts away the complexity of writing raw JCR-SQL2 or XPath queries.

### Why Use Query Builder Over Raw JCR Queries?

| Concern | Raw JCR-SQL2 | Query Builder |
|---------|-------------|--------------|
| **Syntax** | Complex SQL-like syntax | Simple key-value predicates |
| **Pagination** | Manual LIMIT/OFFSET | Built-in `p.limit` / `p.offset` |
| **Facets** | Complex to implement | Built-in faceting |
| **Hits API** | `NodeIterator` — basic | `Hit` objects with excerpts, paths |
| **URL-based testing** | No | ✅ Test directly via browser URL |
| **Performance** | You manage index selection | Automatically uses Oak indexes |

---

## 🔑 Query Builder — Core Concepts

### Predicates

A Query Builder query is a set of **predicates** (key=value filters). Each predicate narrows the search.

```
type=cq:Page                     ← "I want pages"
path=/content/mysite             ← "under this path"
property=jcr:content/pageType   ← "where this property"
property.value=article           ← "equals this value"
p.limit=10                       ← "give me 10 results"
```

### Predicate Groups

Group multiple conditions with `N_group.N_predicate` notation:

```
# OR condition: pages tagged with "summer" OR "campaign"
1_group.p.or=true
1_group.1_tagid=mysite:campaigns/summer
1_group.2_tagid=mysite:campaigns/campaign
```

---

## 🌐 Testing Queries in Browser (REST API)

This is the fastest way to test queries — no code needed:

```
http://localhost:4502/bin/querybuilder.json?
    type=cq:Page&
    path=/content/mysite&
    property=jcr:content/jcr:title&
    property.value=Home&
    p.limit=10&
    p.offset=0

# Response:
{
    "success": true,
    "results": 1,
    "total": 1,
    "more": false,
    "offset": 0,
    "hits": [
        {
            "path": "/content/mysite/en/home",
            "title": "Home",
            "excerpt": "..."
        }
    ]
}
```

---

## 💻 Query Builder in Java — Complete Examples

### Setup

```java
import com.day.cq.search.PredicateGroup;
import com.day.cq.search.Query;
import com.day.cq.search.QueryBuilder;
import com.day.cq.search.result.Hit;
import com.day.cq.search.result.SearchResult;

import java.util.*;
```

### Example 1: Find All Pages of a Type

```java
@Reference
private QueryBuilder queryBuilder;

@SlingObject
private ResourceResolver resourceResolver;

public List<String> findPagesByType(String pageType) {
    Session session = resourceResolver.adaptTo(Session.class);
    List<String> paths = new ArrayList<>();

    Map<String, String> params = new HashMap<>();
    params.put("type",           "cq:Page");            // Only pages
    params.put("path",           "/content/mysite");    // Under this path
    params.put("property",       "jcr:content/pageType");
    params.put("property.value", pageType);
    params.put("p.limit",        "-1");                 // All results (no limit)
    params.put("p.offset",       "0");

    Query query = queryBuilder.createQuery(PredicateGroup.create(params), session);
    SearchResult result = query.getResult();

    LOG.info("Query found {} total matches", result.getTotalMatches());

    for (Hit hit : result.getHits()) {
        try {
            paths.add(hit.getPath());
        } catch (RepositoryException e) {
            LOG.error("Error reading hit path", e);
        }
    }

    // IMPORTANT: Close the result to release the underlying JCR iterator
    result.getIterator();  // Ensure iterator is exhausted (needed for cleanup)

    return paths;
}
```

### Example 2: Full-Text Search with Pagination

```java
public SearchResult searchContent(String searchText, int limit, int offset) {
    Session session = resourceResolver.adaptTo(Session.class);

    Map<String, String> params = new HashMap<>();
    params.put("type",          "cq:Page");
    params.put("path",          "/content/mysite");
    params.put("fulltext",      searchText);       // Full-text search
    params.put("fulltext.relPath", "jcr:content"); // Search within jcr:content

    // Pagination
    params.put("p.limit",       String.valueOf(limit));
    params.put("p.offset",      String.valueOf(offset));

    // Sorting
    params.put("orderby",       "jcr:content/cq:lastModified");
    params.put("orderby.sort",  "desc");

    // Include total count (slightly more expensive)
    params.put("p.guessTotal",  "true");

    Query query = queryBuilder.createQuery(PredicateGroup.create(params), session);
    return query.getResult();
}
```

### Example 3: Date Range Search

```java
public List<String> findRecentlyModified(int daysBack) {
    Session session = resourceResolver.adaptTo(Session.class);

    // Calculate date range
    Calendar fromDate = Calendar.getInstance();
    fromDate.add(Calendar.DAY_OF_YEAR, -daysBack);

    Map<String, String> params = new HashMap<>();
    params.put("type",                "cq:Page");
    params.put("path",                "/content/mysite");

    // Date range predicate
    params.put("daterange.property",  "jcr:content/cq:lastModified");
    params.put("daterange.lowerBound", ISO8601.format(fromDate));  // From X days ago
    params.put("daterange.upperBound", ISO8601.format(Calendar.getInstance()));  // To now

    params.put("p.limit",             "50");
    params.put("orderby",             "jcr:content/cq:lastModified");
    params.put("orderby.sort",        "desc");

    Query query = queryBuilder.createQuery(PredicateGroup.create(params), session);
    List<String> paths = new ArrayList<>();

    for (Hit hit : query.getResult().getHits()) {
        try {
            paths.add(hit.getPath());
        } catch (RepositoryException e) {
            LOG.error("Error", e);
        }
    }

    return paths;
}
```

### Example 4: Tag-Based Search

```java
public List<Resource> findByTags(String[] tagIds) {
    Session session = resourceResolver.adaptTo(Session.class);

    Map<String, String> params = new HashMap<>();
    params.put("type", "cq:Page");
    params.put("path", "/content/mysite");

    // Match ANY of these tags
    params.put("1_group.p.or", "true");
    for (int i = 0; i < tagIds.length; i++) {
        params.put("1_group." + (i + 1) + "_tagid",        tagIds[i]);
        params.put("1_group." + (i + 1) + "_tagid.property", "jcr:content/cq:tags");
    }

    // Example for tagIds = ["mysite:campaigns/summer", "mysite:campaigns/sale"]:
    // 1_group.p.or=true
    // 1_group.1_tagid=mysite:campaigns/summer
    // 1_group.1_tagid.property=jcr:content/cq:tags
    // 1_group.2_tagid=mysite:campaigns/sale
    // 1_group.2_tagid.property=jcr:content/cq:tags

    params.put("p.limit", "20");

    List<Resource> results = new ArrayList<>();
    for (Hit hit : queryBuilder.createQuery(PredicateGroup.create(params), session)
                               .getResult().getHits()) {
        try {
            Resource r = resourceResolver.getResource(hit.getPath());
            if (r != null) results.add(r);
        } catch (RepositoryException e) {
            LOG.error("Error", e);
        }
    }
    return results;
}
```

### Example 5: AND + OR Complex Query

```java
// Find pages that:
// (are tagged with "summer" OR "sale") AND (have pageType = "article") AND (under /content/mysite/en)
public List<String> complexSearch() {
    Session session = resourceResolver.adaptTo(Session.class);

    Map<String, String> params = new LinkedHashMap<>();

    // Base filters
    params.put("type",                     "cq:Page");
    params.put("path",                     "/content/mysite/en");
    params.put("property",                 "jcr:content/pageType");
    params.put("property.value",           "article");

    // OR group for tags
    params.put("1_group.p.or",             "true");
    params.put("1_group.1_tagid",          "mysite:campaigns/summer");
    params.put("1_group.1_tagid.property", "jcr:content/cq:tags");
    params.put("1_group.2_tagid",          "mysite:campaigns/sale");
    params.put("1_group.2_tagid.property", "jcr:content/cq:tags");

    params.put("orderby",                  "@jcr:content/jcr:title");
    params.put("orderby.sort",             "asc");
    params.put("p.limit",                  "20");

    List<String> paths = new ArrayList<>();
    for (Hit hit : queryBuilder.createQuery(PredicateGroup.create(params), session)
                               .getResult().getHits()) {
        try { paths.add(hit.getPath()); } catch (RepositoryException e) { /* log */ }
    }
    return paths;
}
```

---

## 📋 All Query Builder Predicates — Complete Reference

### Core Predicates

| Predicate | Description | Example |
|-----------|-------------|---------|
| `type` | Node type filter | `type=cq:Page` |
| `path` | Search under path | `path=/content/mysite` |
| `path.exact` | Match exact path | `path.exact=true` |
| `path.self` | Include path itself | `path.self=true` |
| `fulltext` | Full-text search | `fulltext=search term` |
| `fulltext.relPath` | Where to search | `fulltext.relPath=jcr:content` |

### Property Predicates

| Predicate | Description | Example |
|-----------|-------------|---------|
| `property` | Property name | `property=jcr:content/jcr:title` |
| `property.value` | Exact value | `property.value=Homepage` |
| `property.operation` | Comparison | `property.operation=like` |
| `property.depth` | Property depth | `property.depth=1` |

```
property.operation values:
  equals       (default)
  unequals
  like         (SQL LIKE — use % wildcard)
  exists       (property must exist)
  not          (property must not exist)
  containsWords
```

### Date Range

| Predicate | Description | Example |
|-----------|-------------|---------|
| `daterange.property` | Date property | `daterange.property=jcr:content/cq:lastModified` |
| `daterange.lowerBound` | From date (ISO 8601) | `daterange.lowerBound=2024-01-01T00:00:00.000Z` |
| `daterange.upperBound` | To date | `daterange.upperBound=2024-12-31T23:59:59.999Z` |
| `relativedaterange.property` | Date property | `relativedaterange.property=jcr:content/cq:lastModified` |
| `relativedaterange.lowerBound` | Relative (e.g., -7d) | `relativedaterange.lowerBound=-7d` |

### Tag Predicates

| Predicate | Description | Example |
|-----------|-------------|---------|
| `tagid` | Tag ID | `tagid=mysite:campaigns/summer` |
| `tagid.property` | Where tags are stored | `tagid.property=jcr:content/cq:tags` |

### Ordering and Pagination

| Predicate | Description | Example |
|-----------|-------------|---------|
| `orderby` | Sort property | `orderby=jcr:content/jcr:title` |
| `orderby.sort` | Sort direction | `orderby.sort=asc` or `desc` |
| `orderby.case` | Case sensitivity | `orderby.case=ignore` |
| `p.limit` | Max results (-1 = all) | `p.limit=20` |
| `p.offset` | Skip N results | `p.offset=20` |
| `p.guessTotal` | Include total count | `p.guessTotal=true` |

### Node Name

| Predicate | Description | Example |
|-----------|-------------|---------|
| `nodename` | Node name filter | `nodename=home` |
| `nodename=*image*` | Wildcard name | Nodes containing "image" |

---

## 🏎️ Performance Best Practices

### Use Specific Predicates (Use Oak Indexes)

```java
// ❌ SLOW: fulltext search on large repositories without index
params.put("fulltext", "laptop");

// ✅ FAST: property-based search that uses a Lucene index
params.put("property",       "jcr:content/productType");
params.put("property.value", "laptop");
```

### Always Set `p.limit`

```java
// ❌ DANGEROUS: No limit — could return millions of nodes
params.put("p.limit", "-1");  // Use -1 ONLY when you absolutely need all results

// ✅ SAFE: Explicit limit with pagination
params.put("p.limit",  "20");
params.put("p.offset", "0");
```

### Don't Use Query Builder in Tight Loops

```java
// ❌ WRONG: Running a query for each page in a list (N+1 problem)
for (String pagePath : pageList) {
    Map<String, String> params = ...;
    // One query per page — terrible performance!
    queryBuilder.createQuery(...).getResult();
}

// ✅ RIGHT: One query to get all pages you need
params.put("path", "/content/mysite");
params.put("type", "cq:Page");
// Get all in one shot, then process the results
```

### Debug with `p.debugOut`

```
# Add to browser URL to get the generated JCR-SQL2 query:
/bin/querybuilder.json?...&p.debugOut=true

# Shows something like:
"debugOut": "SELECT ... FROM [cq:Page] WHERE ISDESCENDANTNODE('/content/mysite') AND [jcr:content/pageType] = 'article'"
```

---

## 🔍 JCR-SQL2 — When to Use Direct Queries

For advanced cases where Query Builder is too limiting:

```java
import javax.jcr.query.QueryManager;
import javax.jcr.query.QueryResult;

Session session = resourceResolver.adaptTo(Session.class);
QueryManager queryManager = session.getWorkspace().getQueryManager();

// JCR-SQL2 syntax
String sql = "SELECT * FROM [cq:Page] AS page " +
             "WHERE ISDESCENDANTNODE(page, '/content/mysite') " +
             "AND [jcr:content/pageType] = 'article' " +
             "ORDER BY [jcr:content/cq:lastModified] DESC";

javax.jcr.query.Query query = queryManager.createQuery(sql, javax.jcr.query.Query.JCR_SQL2);
query.setLimit(20);
query.setOffset(0);

QueryResult result = query.execute();
javax.jcr.NodeIterator nodes = result.getNodes();

while (nodes.hasNext()) {
    javax.jcr.Node node = nodes.nextNode();
    System.out.println(node.getPath());
}
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is Query Builder and why use it over raw JCR queries?**

> **Answer:** Query Builder is AEM's high-level search API that uses simple key-value predicate maps to query JCR content. It's preferred over raw JCR-SQL2/XPath because: it has simpler syntax, built-in pagination (`p.limit`/`p.offset`), a browser-testable REST interface (`/bin/querybuilder.json`), automatic use of Oak indexes, and richer `Hit` objects with path, excerpt, and metadata. For most AEM search use cases, Query Builder is the right tool.

**Q2. How do you implement pagination with Query Builder?**

> **Answer:** Use `p.limit` and `p.offset`:
> - `p.limit=20` — return 20 results
> - `p.offset=0` — start from the first result (page 1)
> - `p.offset=20` — skip first 20, start from result 21 (page 2)
> - `p.guessTotal=true` — include total match count (for displaying "Page 2 of 50")

**Q3. How do you create an OR condition in Query Builder?**

> **Answer:** Use numbered predicate groups with `p.or=true`:
> ```
> 1_group.p.or=true
> 1_group.1_property=jcr:content/type
> 1_group.1_property.value=article
> 1_group.2_property=jcr:content/type
> 1_group.2_property.value=blog
> ```
> This matches pages where type is "article" OR "blog". Multiple groups can be ANDed together.

**Q4. What is the performance impact of `p.limit=-1`?**

> **Answer:** `p.limit=-1` returns ALL matching results — no upper bound. For small datasets this is fine. For large repositories, this can: exhaust heap memory, timeout the request, create a massive JCR node iterator, and slow down the entire AEM instance. Always use explicit limits for production queries and implement pagination. Only use `-1` for background/scheduled jobs that process all results in batches.

**Q5. How do you debug a slow Query Builder query?**

> **Answer:**
> 1. Use `/bin/querybuilder.json?...&p.debugOut=true` to see the generated JCR-SQL2 query
> 2. Copy the generated SQL2 into Oak's Query Performance tool at `/libs/granite/operations/content/diagnosys/queryPerformance.html`
> 3. Check if a Lucene index covers the query's property predicates
> 4. If no index exists, create one in Oak index definitions under `/oak:index/`
> 5. Use `explain=true` in the query to see the execution plan

---

## ✅ Best Practices

1. **Always set `p.limit`** — never leave queries unbounded in production
2. **Use property predicates** over fulltext for structured data — uses indexes better
3. **Combine path + type** always — narrows the search space dramatically
4. **Test in browser first** — `/bin/querybuilder.json?...` before writing Java
5. **Use `p.guessTotal`** only when needed — adds computational cost
6. **Don't run queries in loops** — batch fetch everything, then process in memory
7. **Create custom Oak indexes** for frequently searched properties
8. **Use `daterange` predicates** for time-based queries — more efficient than property.operation=like on date strings

---

## 🛠️ Hands-on Practice

1. Test these queries in browser at `http://localhost:4502/bin/querybuilder.json`:
   - All pages under `/content/mysite` (type=cq:Page, p.limit=20)
   - Pages modified in the last 7 days (relativedaterange)
   - Pages with a specific tag
2. Write a Sling Model that uses QueryBuilder to find the 5 most recently modified pages
3. Write a search servlet that accepts a `q=` param and returns matching page paths as JSON
4. Create an OR query: pages with tag "summer" OR pages with tag "sale"
5. Test `p.debugOut=true` and read the generated JCR-SQL2 — understand what index it's using
