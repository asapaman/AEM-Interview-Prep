# ACL — Access Control Lists in AEM
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What Are ACLs in AEM?

**Access Control Lists (ACLs)** define who can do what in AEM's JCR repository. Every node in the JCR has a set of permissions that control which users or groups can read, write, modify, or delete it.

AEM's access control model is based on the **JCR 2.0 Specification** (JSR-283) and implemented by **Apache Jackrabbit Oak**.

### The Three Layers of AEM Security

```
Layer 1: Authentication  → WHO is the user? (login, SSO, SAML)
Layer 2: Authorization   → WHAT can they access? (ACLs — this guide!)
Layer 3: Dispatcher      → Which URLs can reach AEM? (HTTP-level block)
```

---

## 👤 Principals — Users and Groups

In AEM's JCR, permissions are granted to **principals** (users and groups).

```
/home/
  ├── users/
  │     ├── admin                    ← Built-in admin
  │     ├── anonymous                ← Unauthenticated public users
  │     └── system/                  ← System users (service users)
  │           ├── my-site-reader
  │           └── workflow-service
  └── groups/
        ├── administrators           ← Full access
        ├── content-authors          ← Content editing
        ├── dam-users                ← DAM access
        ├── workflow-users           ← Workflow access
        └── mysite/
              ├── mysite-editors     ← Custom group for editors
              └── mysite-readers     ← Custom group for readers
```

---

## 🔑 JCR Privileges (Permissions)

These are the atomic permissions that ACL entries grant or deny:

| Privilege | What It Allows |
|-----------|---------------|
| `jcr:read` | Read the node and its properties |
| `jcr:write` | Aggregate: addChildNodes + removeNode + setProperties + removeChildNodes |
| `jcr:addChildNodes` | Create child nodes |
| `jcr:removeNode` | Delete a node |
| `jcr:setProperties` | Set/change property values |
| `jcr:removeChildNodes` | Delete child nodes |
| `jcr:readAccessControl` | Read the ACL of a node |
| `jcr:modifyAccessControl` | Modify the ACL of a node |
| `crx:replicate` | Activate/replicate content (Publish it) |
| `rep:write` | Aggregate: jcr:write + addChildNodes + removeChildNodes |

### Aggregate Privileges

```
rep:write = jcr:addChildNodes + jcr:removeNode + jcr:setProperties + jcr:removeChildNodes

jcr:all   = ALL privileges (avoid giving this except to admins)
```

---

## 📐 ACL Structure in JCR

Every node can have an ACL stored in a special `rep:policy` child node:

```
/content/mysite/restricted-section/
  └── rep:policy (rep:ACL)          ← Access Control List for this node
        ├── allow0 (rep:GrantACE)   ← Grant entry
        │     ├── rep:principalName = "mysite-editors"
        │     └── rep:privileges    = ["jcr:read", "jcr:write", "crx:replicate"]
        │
        ├── allow1 (rep:GrantACE)   ← Another grant
        │     ├── rep:principalName = "mysite-readers"
        │     └── rep:privileges    = ["jcr:read"]
        │
        └── deny0 (rep:DenyACE)     ← Deny entry
              ├── rep:principalName = "everyone"
              └── rep:privileges    = ["jcr:all"]
```

---

## 📋 Setting ACLs — Methods

### Method 1: CRXDE Lite (Development Only)

```
CRXDE → Navigate to node → "Access Control" tab
→ + (Add) → Enter principal, select allow/deny, check privileges
→ Save
```

### Method 2: repoinit Scripts (Recommended for AEM Cloud)

**`repoinit`** is AEM's declarative language for creating users, groups, and setting permissions as part of your code deployment. This is the **cloud-ready, code-as-config approach**.

```
# /apps/mysite/osgiconfig/config/org.apache.sling.jcr.repoinit.RepositoryInitializer.cfg.json
{
    "scripts": [
        "create service user my-site-reader",
        "create service user my-site-writer",
        "create group mysite-editors with path /home/groups/mysite",
        "create group mysite-readers with path /home/groups/mysite",

        "set ACL for my-site-reader",
        "    allow jcr:read on /content/mysite",
        "    allow jcr:read on /conf/mysite",
        "end",

        "set ACL for my-site-writer",
        "    allow jcr:read on /content/mysite",
        "    allow rep:write on /content/mysite/data",
        "    allow crx:replicate on /content/mysite",
        "end",

        "set ACL for mysite-editors",
        "    allow jcr:read on /content/mysite",
        "    allow rep:write on /content/mysite",
        "    allow crx:replicate on /content/mysite",
        "    allow jcr:read on /conf/mysite",
        "end",

        "set ACL for mysite-readers",
        "    allow jcr:read on /content/mysite",
        "end"
    ]
}
```

### Method 3: Programmatic via Jackrabbit API

```java
import org.apache.jackrabbit.api.security.JackrabbitAccessControlList;
import org.apache.jackrabbit.api.security.user.UserManager;
import org.apache.jackrabbit.commons.jackrabbit.authorization.AccessControlUtils;

import javax.jcr.*;
import javax.jcr.security.*;

// In a Sling Model or OSGi service (with appropriate service user)
public void setReadPermission(ResourceResolver resolver, String principalName, String path) {
    Session session = resolver.adaptTo(Session.class);
    if (session == null) return;

    try {
        AccessControlManager acm = session.getAccessControlManager();

        // Get or create the ACL for the node
        JackrabbitAccessControlList acl =
            AccessControlUtils.getAccessControlList(session, path);

        // Find the principal (user/group)
        Principal principal = AccessControlUtils.getPrincipal(session, principalName);

        // Add grant entry
        Privilege[] readPrivilege = new Privilege[] {
            acm.privilegeFromName(Privilege.JCR_READ)
        };
        acl.addEntry(principal, readPrivilege, true /* isAllow */);

        // Apply the ACL
        acm.setPolicy(path, acl);

        // Commit
        session.save();

        LOG.info("Granted jcr:read to '{}' on '{}'", principalName, path);

    } catch (RepositoryException e) {
        LOG.error("Failed to set ACL on '{}' for '{}'", path, principalName, e);
    }
}
```

### Method 4: Content Package (ui.content)

```xml
<!-- /apps/mysite/ui.content/jcr_root/content/mysite/restricted/.content.xml -->
<jcr:root xmlns:rep="internal"
          xmlns:jcr="http://www.jcp.org/jcr/1.0"
          jcr:primaryType="cq:Page">
  <rep:policy jcr:primaryType="rep:ACL">
    <allow0 jcr:primaryType="rep:GrantACE"
            rep:principalName="mysite-editors"
            rep:privileges="{Name}[jcr:read,rep:write,crx:replicate]"/>
    <allow1 jcr:primaryType="rep:GrantACE"
            rep:principalName="mysite-readers"
            rep:privileges="{Name}[jcr:read]"/>
    <deny0  jcr:primaryType="rep:DenyACE"
            rep:principalName="everyone"
            rep:privileges="{Name}[jcr:all]"/>
  </rep:policy>
</jcr:root>
```

---

## 🔐 Service Users — The Right Way to Do Programmatic JCR Access

### The Old Way (NEVER Use in Modern AEM)

```java
// ❌ DEPRECATED — Admin session bypasses all ACLs
Session adminSession = repository.loginAdministrative(null);
// This bypasses all ACL checks — security risk!
```

### The Right Way — Service User

```java
// ✅ Proper: Use service user with minimal permissions

// Step 1: Create service user via repoinit
// "create service user my-content-reader"
// "set ACL for my-content-reader"
// "    allow jcr:read on /content/mysite"
// "end"

// Step 2: Map service user to your bundle via OSGi config
// ServiceUserMapper service mapping (see below)

// Step 3: Use in code
@Reference
private ResourceResolverFactory resolverFactory;

public void readContent() {
    Map<String, Object> params = Collections.singletonMap(
        ResourceResolverFactory.SUBSERVICE, "my-content-reader"  // subservice name
    );
    try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(params)) {
        Resource content = resolver.getResource("/content/mysite/en/home");
        // Works only because my-content-reader has jcr:read on /content/mysite
    } catch (LoginException e) {
        LOG.error("Service user login failed", e);
    }
}
```

### Service User Mapping Configuration

```json
// /apps/mysite/osgiconfig/config/org.apache.sling.serviceusermapping.impl.ServiceUserMapperImpl.amended-mysite.cfg.json
{
    "user.mapping": [
        "com.mysite.core:my-content-reader=[my-content-reader]",
        "com.mysite.core:workflow-service=[my-workflow-user]"
    ]
}
```

**Format:** `bundle-symbolic-name:subservice-name=[system-user-name]`

---

## 🏛️ ACL Evaluation — How Oak Resolves Permissions

```
User makes request to /content/mysite/restricted/page

Oak checks ACLs from MOST SPECIFIC to LEAST SPECIFIC path:
/content/mysite/restricted/page/rep:policy   → Check here first
/content/mysite/restricted/rep:policy        → Then here
/content/mysite/rep:policy                   → Then here
/content/rep:policy                          → Then here
/rep:policy                                  → Root (last resort)

Rules:
1. DENY beats ALLOW at the same path level
2. More specific path overrides parent path
3. Anonymous user inherits "everyone" group ACLs
4. Users inherit ALL their groups' ACLs (merged)
```

---

## 📊 User Administration Tools

### Security Console

```
/useradmin                    → User Manager (AEM 6.5)
/libs/granite/security/content/groupeditor.html → Groups UI
Tools → Security → Users      → Modern user management
Tools → Security → Groups     → Group management
```

### Oak User Management via JMX

```
/system/console/jmx
→ org.apache.jackrabbit.oak:type=Security
→ Use to force ACL reindex if permissions seem cached incorrectly
```

---

## 🔄 Closed User Groups (CUG)

**CUG** restricts read access to a content tree to a specific set of groups. Used for:
- Member-only sections
- Premium content
- Geo-restricted content

```xml
<!-- Apply CUG to a node (via rep:CugPolicy) -->
<!-- Only users/groups listed in "principalNames" can read this node tree -->
<jcr:root xmlns:rep="internal"
          jcr:primaryType="cq:Page">
  <rep:cugPolicy jcr:primaryType="rep:CugPolicy"
                 rep:principalNames="[mysite-premium-members,administrators]"/>
</jcr:root>
```

```
AEM Author:
Closed User Group (CUG) tab in Page Properties:
→ Check "Enable CUG"
→ Add groups: mysite-premium-members, administrators
→ This restricts READ access on the Publish tier to those groups
→ Anonymous users get 401/redirect to login page
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is the difference between ACL and CUG in AEM?**

> **Answer:**
> - **ACL** (Access Control List): Fine-grained, per-node permission rules controlling any JCR privilege (read, write, replicate) for any principal (user/group). Applied to both Author and Publish.
> - **CUG** (Closed User Group): Specifically restricts READ access on the Publish tier to a defined set of groups. Simpler than ACLs — just "only these groups can see this section." Used for member-only content.

**Q2. Why should you use service users instead of admin sessions?**

> **Answer:** `loginAdministrative()` bypasses all ACL checks — any code using it can read/write anything in the repository, creating a serious security risk. Service users have specific, minimal permissions defined via repoinit scripts (principle of least privilege). If a service user's credentials are compromised or the code has a bug, the damage is limited to that user's permitted paths. Additionally, `loginAdministrative()` is deprecated and disabled by default in AEM 6.5+.

**Q3. What is repoinit and why is it preferred in AEM Cloud?**

> **Answer:** `repoinit` is AEM's declarative language for defining repository initialization — creating users, groups, service users, and ACLs as part of your code (OSGi config). It's preferred in AEM Cloud because the immutable architecture means you can't manually create users in CRXDE on production. Repoinit scripts run at startup before the repository is accessible to the application, ensuring the correct users and permissions exist before any code tries to use them. It's version-controlled, repeatable, and environment-agnostic.

**Q4. What does `jcr:write` include?**

> **Answer:** `jcr:write` is an aggregate privilege that includes: `jcr:addChildNodes` (create child nodes), `jcr:removeNode` (delete this node), `jcr:setProperties` (set/change properties), and `jcr:removeChildNodes` (delete child nodes). It does NOT include `crx:replicate` — you need that separately if the service user needs to publish content.

**Q5. How does ACL inheritance work in Oak (where does it check first)?**

> **Answer:** Oak evaluates ACLs from the most specific path upward. For a resource at `/content/mysite/en/home`, it checks `/content/mysite/en/home` first, then `/content/mysite/en`, then `/content/mysite`, then `/content`, then `/`. The most specific applicable ACL wins. DENY at the same level beats ALLOW. A DENY at a parent level can be overridden by an explicit ALLOW at a more specific path.

---

## ✅ Best Practices

1. **Always use service users** — never `loginAdministrative()`
2. **Define permissions via repoinit** — code-as-config, version-controlled
3. **Principle of least privilege** — grant only the exact paths and privileges needed
4. **Use groups for permissions** — assign users to groups, grant permissions to groups (not individual users)
5. **Separate service users by function** — one for reading, one for writing — don't share
6. **Test service user ACLs in dev** — use Oak's effective permissions tool to verify
7. **Never grant `jcr:all` to service users** — only to admin (and even that sparingly)
8. **Monitor `/system/console/slinglog`** for `LoginException` — indicates service user misconfiguration

---

## 🛠️ Hands-on Practice

1. **Create a service user** via repoinit with `jcr:read` only on `/content/mysite`
2. **Set up ServiceUserMapper** mapping the subservice name to your system user
3. **Write a Sling Model** that reads content using the service user (not admin session)
4. **Test ACL isolation:** Try to access `/conf/mysite` with the reader service user — verify it gets null (no permission)
5. **Set a CUG** on a page: `Page Properties → Permissions → Closed User Group → add group`
6. **Verify ACLs in CRXDE:** Navigate to node → Access Control tab → see all effective ACEs
