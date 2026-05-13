# Day 26 — Permissions, Users & Security
**Difficulty:** Medium | **4 YOE Focus**

---

## 📖 Topic Explanation

AEM uses JCR-based Access Control Lists (ACLs) for fine-grained permissions. Users and groups are stored in the JCR under `/home/users/` and `/home/groups/`.

---

## 👤 Users, Groups & Roles

### Out-of-the-box Groups
| Group | Access |
|-------|--------|
| `administrators` | Full system access |
| `content-authors` | Create/edit content pages |
| `dam-users` | Create/manage DAM assets |
| `workflow-users` | Participate in workflows |
| `replication-receivers` | Used by replication agents |

### User Types
- **Regular Users**: Human users who log in to AEM
- **Service Users**: Technical accounts for system processes (schedulers, services, replication). NO password, used programmatically.
- **System Users**: Similar to service users, created at bundle installation

---

## 🔐 ACL Structure (JCR Permissions)

Permissions are set as `rep:policy` nodes:
```xml
<!-- /content/mysite/.content.xml -->
<jcr:root>
  <rep:policy jcr:primaryType="rep:ACL">
    <!-- Allow editors to read and write /content/mysite -->
    <allow0 jcr:primaryType="rep:GrantACE"
            rep:principalName="mysite-editors"
            rep:privileges="{Name}[jcr:read,jcr:write,crx:replicate]"/>
    <!-- Deny everyone else write access -->
    <deny0 jcr:primaryType="rep:DenyACE"
           rep:principalName="everyone"
           rep:privileges="{Name}[jcr:write]"/>
  </rep:policy>
</jcr:root>
```

### Common Privileges
| Privilege | Description |
|-----------|------------|
| `jcr:read` | Read nodes and properties |
| `jcr:write` | Create, modify, delete nodes/properties |
| `jcr:modifyProperties` | Modify properties only |
| `jcr:addChildNodes` | Add child nodes |
| `jcr:removeNode` | Delete nodes |
| `crx:replicate` | Activate/replicate content |
| `jcr:modifyAccessControl` | Change permissions |

---

## 🤖 Service Users (Critical for 4 YOE!)

Service users allow OSGi services/schedulers to access JCR without admin credentials.

### Step 1: Create Service User
```
CRXDE → /home/users/system/mysite/mysite-service-user
jcr:primaryType = rep:SystemUser
```
Or via `repoinit` script.

### Step 2: Map Service User to Bundle
```xml
<!-- OSGi Config: org.apache.sling.serviceusermapping.impl.ServiceUserMapperImpl.amended-mysite.cfg.json -->
{
    "user.mapping": [
        "com.mysite.core:my-subservice-name=mysite-service-user"
    ]
}
```

### Step 3: Use in Code
```java
@Reference
private ResourceResolverFactory resolverFactory;

private void doSecureWork() {
    Map<String, Object> params = new HashMap<>();
    params.put(ResourceResolverFactory.SUBSERVICE, "my-subservice-name");

    try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(params)) {
        Resource resource = resolver.getResource("/content/mysite/data");
        // Do work
    } catch (LoginException e) {
        LOG.error("Service user login failed", e);
    }
}
```

---

## 🔑 What is ResourceResolverFactory?

`ResourceResolverFactory` is an OSGi service that creates `ResourceResolver` instances. It's the gateway to JCR access:

| Method | Use Case |
|--------|---------|
| `getResourceResolver(authInfo)` | User-authenticated resolver (from request) |
| `getServiceResourceResolver(params)` | Service user resolver (for background tasks) |
| `getAdministrativeResourceResolver()` | ⚠️ DEPRECATED — admin resolver (avoid!) |

---

## ❓ Interview Questions & Answers

**Q1. What are User Roles, Groups and Permissions in AEM?**
> - **Users**: Individuals who interact with AEM (human or system)
> - **Groups**: Collections of users. Permissions are assigned to groups, not individual users (best practice)
> - **Permissions**: JCR ACLs defining what actions (read, write, replicate) are allowed/denied on specific paths

**Q2. What is a service user and why do we need them?**
> Service users are system-level users (no password) used by OSGi services for JCR access. We need them because using `admin` credentials in code is a security risk (OWASP) and is deprecated since AEM 6.2. Service users have minimal, specific permissions (principle of least privilege).

**Q3. How do you configure a service user mapping?**
> 1. Create a system user in JCR
> 2. Assign appropriate ACLs to the user
> 3. Create an OSGi config `ServiceUserMapper.amended` with the bundle+subservice → user mapping
> 4. In code, use `ResourceResolverFactory.getServiceResourceResolver(Map.of(SUBSERVICE, "name"))`

**Q4. What is the principle of least privilege in AEM?**
> Service users and user groups should only have the MINIMUM permissions required for their function. E.g., a scheduler that reads assets should only have `jcr:read` on `/content/dam`, not write access to all of `/content`.

**Q5. What is `ResourceResolverFactory` and when should you use it?**
> It's the OSGi service for creating `ResourceResolver` instances. Use `getServiceResourceResolver()` in schedulers, workflow steps, and any background process. NEVER use `getAdministrativeResourceResolver()` (deprecated, insecure).

**Q6. How do you grant access to only specific paths for authors?**
> In CRXDE or via UI (Security tab): Set `rep:GrantACE` with `jcr:read, jcr:write` on `/content/mysite/specific-path`. Use `rep:DenyACE` to explicitly deny access to sub-paths if needed.

**Q7. What is `rep:SystemUser`?**
> A special JCR user type (`rep:SystemUser`) for system/service accounts. Cannot be used for UI login, has no password. Created under `/home/users/system/`. Always use this for service user accounts.

---

## ✅ Best Practices

- **Never** use admin credentials in code — always use service users
- Assign permissions to **groups**, not individual users
- Use the **principle of least privilege** — minimal permissions needed
- Always close `ResourceResolver` in a `finally` block or try-with-resources
- Use `rep:SystemUser` for service accounts, not regular `rep:User`
- Store service user configurations in version-controlled OSGi config files
- Regularly audit user permissions via `crx/de/index.jsp#/home/users`

---

## 🛠️ Hands-on Task

1. Create a service user `mysite-read-service` with `jcr:read` on `/content/mysite`
2. Create the OSGi service user mapping config
3. Write a Scheduler that uses this service user to read pages and log their titles
