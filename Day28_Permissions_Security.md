# Day 26 — Permissions, Users & Security
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 Why Security Matters in AEM

AEM stores critical business content and often has integrations with backend systems, payment APIs, and customer data. Poorly configured security can lead to:
- Unauthorized content publishing
- Data breaches (accessing private assets)
- Elevation of privilege (regular users getting admin access)
- Memory leaks (admin session held open in code)

---

## 👤 Types of AEM Users

### 1. Regular (Human) Users
```
/home/users/mysite/
  ├── john-editor          ← Content editor
  ├── sarah-admin          ← Site admin
  └── bob-author           ← Content author
```

### 2. System Users (Service Users) — Critical Concept!
```
/home/users/system/
  ├── mysite-content-reader   ← For reading /content/mysite
  ├── mysite-dam-writer       ← For writing to /content/dam/mysite
  └── mysite-form-processor   ← For saving form submissions
```

**System users:**
- Have `jcr:primaryType = rep:SystemUser`
- NO password — cannot log into AEM UI
- Used by OSGi services, schedulers, and workflow steps
- Have MINIMAL permissions (principle of least privilege)
- Created under `/home/users/system/` to clearly distinguish from human users

### 3. Built-in Groups
| Group | Purpose |
|-------|---------|
| `administrators` | Full access to everything |
| `content-authors` | Create/edit pages under `/content` |
| `dam-users` | Create/manage assets in DAM |
| `workflow-users` | Participate in workflows |
| `everyone` | All authenticated users |

---

## 🔐 JCR Access Control Lists (ACLs)

### What is an ACL?
An ACL is a list of access rules attached to a JCR node. Each rule (ACE = Access Control Entry) says:
- **Who** (user or group) has/doesn't have access
- **What** permissions (read, write, replicate, etc.)
- **Where** (the node and optionally its children)

### ACL in CRXDE
```
/content/mysite/    [cq:Page]
  └── rep:policy    [rep:ACL]         ← ACL node
        ├── allow0  [rep:GrantACE]    ← Grant entry
        │     ├── rep:principalName = "mysite-editors"
        │     └── rep:privileges = [jcr:read, jcr:write, crx:replicate]
        ├── allow1  [rep:GrantACE]
        │     ├── rep:principalName = "content-authors"
        │     └── rep:privileges = [jcr:read, jcr:modifyProperties]
        └── deny0   [rep:DenyACE]     ← Deny entry
              ├── rep:principalName = "everyone"
              └── rep:privileges = [jcr:write]
```

### Common JCR Privileges
| Privilege | What it allows |
|-----------|---------------|
| `jcr:read` | Read nodes and their properties |
| `jcr:write` | Aggregate: add, modify, remove nodes and properties |
| `jcr:modifyProperties` | Modify existing properties |
| `jcr:addChildNodes` | Create child nodes |
| `jcr:removeNode` | Delete nodes |
| `jcr:removeChildNodes` | Remove children |
| `crx:replicate` | Activate/publish content to Publish |
| `jcr:lockManagement` | Lock/unlock nodes |
| `jcr:modifyAccessControl` | Change ACLs (very powerful — restrict carefully) |
| `jcr:all` | All privileges (administrator-level) |

---

## 🤖 Service Users — Deep Dive

This is one of the **most important topics for 4 YOE interviews**.

### Why Service Users?
Before AEM 6.2, code used admin sessions:
```java
// ❌ NEVER DO THIS — security risk, deprecated in AEM 6.2+
Session adminSession = repository.loginAdministrative(null);
ResourceResolver adminResolver = resolverFactory.getAdministrativeResourceResolver(null);
```

With admin sessions:
- Code has FULL access to everything — if code has a bug, it can delete/modify anything
- No audit trail — all changes attributed to "admin"
- No principle of least privilege

With service users:
- Only specific paths are accessible
- Audit trail shows the service user's actions
- Follows principle of least privilege

### Creating a Service User (AEM 6.5)

**Option 1: Via CRXDE (for development)**
```
1. Navigate to /home/users/system/mysite/ (create if doesn't exist)
2. Create node: mysite-content-reader
3. Node type: rep:SystemUser
4. Properties:
   - jcr:primaryType = rep:SystemUser
   - rep:principalName = mysite-content-reader
```

**Option 2: Via `repoinit` Script (Recommended for production)**
```
# ui.config/src/main/content/jcr_root/apps/mysite/osgiconfig/config/
# org.apache.sling.jcr.repoinit.RepositoryInitializer.cfg.json

{
    "scripts": [
        "create service user mysite-content-reader\n
         set ACL for mysite-content-reader\n
           allow jcr:read on /content/mysite\n
           allow jcr:read on /content/dam/mysite\n
         end"
    ]
}
```

### Setting ACLs for Service User
```
# In repoinit script:
set ACL for mysite-form-processor
    allow jcr:read, jcr:addChildNodes, jcr:modifyProperties on /content/userdata/forms
    allow jcr:read on /content/mysite
    deny  jcr:write on /content/mysite  # Can't modify site content
end
```

### Mapping Service User to OSGi Bundle

```json
// File: org.apache.sling.serviceusermapping.impl.ServiceUserMapperImpl.amended-mysite.cfg.json
{
    "user.mapping": [
        "com.mysite.core:content-reader=mysite-content-reader",
        "com.mysite.core:dam-writer=mysite-dam-writer",
        "com.mysite.core:form-processor=mysite-form-processor"
    ]
}
```

**Pattern:** `bundle-symbolic-name:subservice-name=system-user-id`

### Using Service User in Code

```java
package com.mysite.core.services.impl;

import org.apache.sling.api.resource.*;
import org.osgi.service.component.annotations.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import javax.jcr.RepositoryException;
import java.util.*;

@Component(service = ContentReaderService.class)
public class ContentReaderServiceImpl implements ContentReaderService {

    private static final Logger LOG = LoggerFactory.getLogger(ContentReaderServiceImpl.class);

    // Subservice name matches the ServiceUserMapper config
    private static final String SUBSERVICE_NAME = "content-reader";

    @Reference
    private ResourceResolverFactory resolverFactory;

    @Override
    public String getPageTitle(String pagePath) {
        // Build params map with subservice name
        Map<String, Object> params = new HashMap<>();
        params.put(ResourceResolverFactory.SUBSERVICE, SUBSERVICE_NAME);

        // ALWAYS use try-with-resources!
        try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(params)) {
            Resource pageResource = resolver.getResource(pagePath);
            if (pageResource == null) {
                LOG.warn("Page not found: {}", pagePath);
                return null;
            }

            // Navigate to jcr:content for page properties
            Resource jcrContent = pageResource.getChild("jcr:content");
            if (jcrContent == null) return null;

            return jcrContent.getValueMap().get("jcr:title", String.class);

        } catch (LoginException e) {
            LOG.error("Failed to get ResourceResolver for subservice '{}'. "
                + "Check service user mapping and ACLs.", SUBSERVICE_NAME, e);
            return null;
        }
        // ResourceResolver is auto-closed by try-with-resources
    }

    @Override
    public void saveFormData(String formPath, Map<String, String> formData) {
        // Use a DIFFERENT subservice with WRITE permissions
        Map<String, Object> params = new HashMap<>();
        params.put(ResourceResolverFactory.SUBSERVICE, "form-processor");

        try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(params)) {
            Resource formRoot = resolver.getResource(formPath);
            if (formRoot == null) {
                LOG.error("Form storage path not found: {}", formPath);
                return;
            }

            // Create new submission node
            String submissionId = "submission-" + System.currentTimeMillis();
            Resource newNode = resolver.create(formRoot, submissionId,
                Collections.singletonMap("jcr:primaryType", "nt:unstructured"));

            // Set properties
            ModifiableValueMap props = newNode.adaptTo(ModifiableValueMap.class);
            if (props != null) {
                props.putAll(formData);
                props.put("submittedAt", Calendar.getInstance());
            }

            // COMMIT the changes!
            resolver.commit();
            LOG.info("Saved form submission: {}/{}", formPath, submissionId);

        } catch (LoginException e) {
            LOG.error("Service user login failed for form-processor", e);
        } catch (PersistenceException e) {
            LOG.error("Failed to save form data to JCR", e);
        }
    }
}
```

---

## 🔑 ResourceResolverFactory — Complete Reference

```java
@Reference
private ResourceResolverFactory resolverFactory;

// Method 1: Service User (for services, schedulers, workflows)
// ALWAYS prefer this
Map<String, Object> params = Collections.singletonMap(
    ResourceResolverFactory.SUBSERVICE, "my-subservice"
);
try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(params)) {
    // Use resolver
}

// Method 2: Current user's session (for servlets/request processing)
// Use request.getResourceResolver() instead — it's already the user's session
// Don't create a new resolver from the request session

// Method 3: ❌ NEVER USE — deprecated and insecure
// resolverFactory.getAdministrativeResourceResolver(null);

// Checking if resolver is still valid
ResourceResolver resolver = ...; // obtained
if (resolver != null && resolver.isLive()) {
    // Safe to use
    resolver.close();  // Or use try-with-resources
}
```

---

## 🔒 Dispatcher Security Configuration

The Dispatcher is the first line of defense. Block dangerous paths:

```apache
# dispatcher.any — Security rules

/filter {
    # Deny everything by default
    /0001 { /type "deny" /url "*" }

    # Allow GET requests to public content
    /0002 { /type "allow" /method "GET" /url "/content/mysite/*" }

    # Allow static resources
    /0003 { /type "allow" /method "GET" /url "/etc.clientlibs/*" }

    # Allow health check
    /0004 { /type "allow" /method "GET" /url "/bin/mysite/health" }

    # NEVER allow these paths:
    # /system/console/* — OSGi console
    # /crx/*           — Repository browser
    # /libs/granite/*  — Admin UIs
    # *.cfg, *.xml     — Config files
    # /content/dam/*.html — DAM assets as pages
}
```

---

## 📊 Permission Troubleshooting

```
Problem: Service user getting AccessDeniedException
→ Check 1: Does the system user exist?
  /crx/de → /home/users/system/mysite/mysite-content-reader

→ Check 2: Does the user have the right ACLs?
  /crx/de → navigate to target path → right-click → Access Control

→ Check 3: Is the ServiceUserMapper config correct?
  /system/console/configMgr → "Apache Sling Service User Mapper Service Amendment"
  Verify: com.mysite.core:content-reader=mysite-content-reader

→ Check 4: Is the bundle symbolic name correct?
  /system/console/bundles → find your bundle → check Bundle-SymbolicName in MANIFEST.MF

→ Check 5: Check the error log
  /system/console/slinglog → ERROR for LoginException with service user details
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is a service user and why do we need them?**

> **Answer:** A service user is a system-level JCR account (`rep:SystemUser`) used by OSGi services, schedulers, and workflow steps for JCR access. We need them because:
> 1. Admin sessions (`loginAdministrative()`) are deprecated since AEM 6.2 — they give unlimited access which violates the principle of least privilege
> 2. Service users have ONLY the permissions they need — a content reader service can only read, not write
> 3. Audit trail: JCR changes show the service user's name, not "admin"
> 4. Security: if a service is compromised, attacker only has that service user's limited permissions

**Q2. How do you set up a service user mapping?**

> **Answer:**
> 1. Create a system user in JCR: `/home/users/system/mysite/mysite-reader` with `jcr:primaryType=rep:SystemUser`
> 2. Grant ACLs: allow `jcr:read` on `/content/mysite` for this user
> 3. Create OSGi config `ServiceUserMapperImpl.amended-mysite.cfg.json` with `user.mapping` property: `["com.mysite.core:reader-service=mysite-reader"]`
> 4. In code: `Map.of(ResourceResolverFactory.SUBSERVICE, "reader-service")` → `resolverFactory.getServiceResourceResolver(params)`

**Q3. What is the principle of least privilege?**

> **Answer:** Each user, group, or service should have ONLY the minimum permissions required to do their job. A content author doesn't need to replicate content — remove that privilege. A form processing service only needs write access to `/content/forms` — not all of `/content`. This limits the blast radius if an account is compromised or has a bug.

**Q4. Why must you always close a ResourceResolver?**

> **Answer:** Each `ResourceResolver` holds an underlying JCR session with allocated memory and potentially database connections. If not closed, sessions accumulate and eventually cause:
> - Memory leaks (OutOfMemoryError)
> - Session limits exceeded (too many open sessions → new sessions fail)
> - JCR repository instability
> Always use try-with-resources: `try (ResourceResolver resolver = ...) { }` — it automatically calls `resolver.close()` even if an exception occurs.

**Q5. What is `rep:policy` in JCR?**

> **Answer:** `rep:policy` is a special child node of type `rep:ACL` (Access Control List) that stores the access control entries for that node. Each entry (`rep:GrantACE` for allow, `rep:DenyACE` for deny) contains `rep:principalName` (who) and `rep:privileges` (what). The ACL applies to that node and, by default, its descendants.

**Q6. How do you check a user's current permissions in CRXDE?**

> **Answer:** In CRXDE Lite: navigate to the node → right-click → Access Control → Current Effective Policy. This shows what the currently logged-in user or a specified user can do on that node. You can also check at `/system/console/jmx` → `com.adobe.granite:type=Repository` for repository-level info.

---

## ✅ Best Practices

1. **Always use service users** — never `getAdministrativeResourceResolver()`
2. **Minimal permissions** — grant only what's needed, on specific paths
3. **Use `repoinit` scripts** to create service users — version-controlled, repeatable
4. **Separate subservice names** for different permission levels (reader vs writer)
5. **Always use try-with-resources** for ResourceResolver — prevents leaks
6. **Lock down Dispatcher** — deny everything, allow only what's needed
7. **Never store credentials** in code — use OSGi config with `$[secret:]` in Cloud
8. **Regularly audit** user permissions — remove inactive users and overly broad permissions

---

## 🛠️ Hands-on Practice

1. Create a service user `mysite-reader` via CRXDE with `jcr:read` on `/content/mysite`
2. Configure `ServiceUserMapperImpl` OSGi config to map it to `com.mysite.core:content-reader`
3. Write a scheduler that uses this service user to list all pages under `/content/mysite/en`
4. Test what happens when you remove `jcr:read` from the service user — observe `LoginException`
