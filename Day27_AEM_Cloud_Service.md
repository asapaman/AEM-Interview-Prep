# Day 27 — AEM Cloud Service
**Difficulty:** Hard | **4 YOE Focus**

---

## 📖 Topic Explanation

AEM as a Cloud Service (AEMaaCS) is the SaaS version of AEM, running on Adobe's managed cloud infrastructure (Adobe I/O Runtime + Kubernetes). It fundamentally changes how AEM is deployed and maintained.

---

## ☁️ Cloud vs On-Premises

| Aspect | AEM On-Premise / Managed Services | AEM as a Cloud Service |
|--------|----------------------------------|----------------------|
| Infrastructure | Customer-managed / Adobe-managed VMs | Adobe-managed Kubernetes |
| AEM Updates | Quarterly patch cycles | Continuous (auto-updated) |
| Scaling | Manual / pre-configured | Automatic horizontal scaling |
| Author HA | Single author | Multiple author pods |
| Codebase deployment | Package Manager | Cloud Manager CI/CD pipeline |
| Mutable Content | JCR mutable | Content immutable (code) |
| Service Pack | Manual | Automatic |

---

## 🏗️ AEMaaCS Architecture

```
[Cloud Manager]
      ↓ CI/CD Pipeline
[Build] → [Author Service] ←→ [Publish Service] → [CDN (Fastly)]
              ↓                      ↓
         [Asset Compute]        [Dispatcher]
              ↓
         [Adobe Asset Link]
```

### Author Service
- **Highly available** (multiple author pods)
- Content created/edited here
- Not directly accessible to end users

### Publish Service
- **Auto-scaling** based on traffic
- Content served to end users
- Stateless — no session affinity required

### CDN (Fastly)
- Adobe-managed CDN
- Caches content globally
- Supports custom rules and redirects

---

## 🔑 Key AEMaaCS Concepts

### Immutable Architecture
In AEMaaCS, **code is immutable** — you can't install packages via Package Manager in production. All code deployments go through **Cloud Manager pipeline**. Content (JCR data) remains mutable.

### Sling Feature Flags
```java
// Check if running on Cloud Service
if (Boolean.getBoolean("aem.cloud")) {
    // Cloud-specific logic
}
```

### Forbidden APIs in AEMaaCS
The following APIs/patterns are blocked in Cloud Service:
- `SlingRepository.loginAdministrative()` → Use service users
- `ResourceResolverFactory.getAdministrativeResourceResolver()` → Use service users
- Sync replication → Use async
- Custom Lucene index definitions in wrong location
- Static templates (partially) → Use Editable Templates

---

## 🔧 Cloud Manager

Cloud Manager is Adobe's CI/CD platform for AEMaaCS:

### Pipeline Types
| Pipeline | Purpose |
|----------|---------|
| **Full Stack** | Deploy both frontend and backend code |
| **Frontend** | Deploy only frontend (JS/CSS) |
| **Config** | Deploy OSGi configs, Dispatcher configs |

### Pipeline Stages
```
Code Quality → Build → Unit Tests → Security Scan → Deploy → Functional Tests → Performance Tests
```

### Quality Gates
- Code coverage minimum (configurable)
- SonarQube quality rules
- AEM-specific Dispatcher rules validation
- API compatibility checks (AEM Analyzer Maven plugin)

---

## 📦 Project Structure for AEMaaCS

```
mysite/
  ├── core/              ← Java code (OSGi bundle)
  ├── ui.apps/           ← Component scripts, dialogs, clientlibs (/apps content)
  ├── ui.config/         ← OSGi configs, Dispatcher configs
  ├── ui.content/        ← Initial JCR content (/content, /conf)
  ├── ui.frontend/       ← Webpack/frontend build
  ├── ui.tests/          ← Functional tests (Selenium/Cypress)
  └── all/               ← All-in-one deployment package
```

---

## 🔍 AEM Analyzer Maven Plugin

Validates code against AEMaaCS compatibility rules BEFORE deployment:
```xml
<plugin>
    <groupId>com.adobe.aem</groupId>
    <artifactId>aemanalyser-maven-plugin</artifactId>
    <version>1.x.x</version>
    <executions>
        <execution>
            <id>aem-analyser</id>
            <goals><goal>project-analyse</goal></goals>
        </execution>
    </executions>
</plugin>
```

Catches issues like: deprecated API usage, incorrect repo paths, forbidden Sling configurations.

---

## ❓ Interview Questions & Answers

**Q1. Why is AEM Cloud Service beneficial over On-Premise AEM?**
> 1. **No maintenance overhead**: Adobe handles updates, patches, infrastructure
> 2. **Auto-scaling**: Publish tier scales automatically with traffic
> 3. **Always up-to-date**: Continuous updates — no quarterly service pack cycles
> 4. **Built-in CI/CD**: Cloud Manager provides standardized deployment pipeline with quality gates
> 5. **Cloud-native features**: Asset Compute, CDN integration, Sling Job processing

**Q2. What is the significance of "immutable architecture" in AEMaaCS?**
> Code deployed to AEMaaCS cannot be changed via Package Manager or CRXDE in production. All code changes must go through Cloud Manager CI/CD pipeline. This ensures consistency, auditability, and prevents unauthorized changes. Content (JCR data) remains mutable.

**Q3. What is Cloud Manager and what does its pipeline include?**
> Cloud Manager is Adobe's CI/CD orchestration platform for AEMaaCS. A full-stack pipeline includes: code quality analysis (SonarQube), Maven build, unit tests, AEM analyzer checks, deployment to dev/stage/prod, functional tests, and performance tests. Multiple pipeline types exist for full-stack, frontend, and config-only deployments.

**Q4. What changes are needed when migrating code from AEM 6.5 to AEMaaCS?**
> 1. Remove all deprecated API usages (admin session, etc.)
> 2. Replace static with editable templates
> 3. Update OSGi config paths to `/apps/.../osgiconfig/`
> 4. Fix all Sling repository structure violations (AEM Analyzer)
> 5. Update index definitions to OAK-compatible format
> 6. Move Dispatcher configs to `ui.config/` module
> 7. Run AEM Best Practices Analyzer tool

**Q5. What is RDE (Rapid Development Environment) in AEMaaCS?**
> RDE is a special AEMaaCS environment for rapid iterative development. Unlike regular dev environments, you can deploy individual bundles, content packages, and Dispatcher configs directly via CLI without running the full pipeline. Enables fast feedback loop during development.

---

## ✅ Best Practices

- Always run the **AEM Analyzer** Maven plugin locally before pushing to Cloud Manager
- Use **environment variables** in Cloud Manager for secrets (never hardcode)
- Use **Frontend Pipeline** for CSS/JS changes — avoid full-stack pipeline for frontend-only changes
- Keep OSGi configs in run-mode folders (`config.author`, `config.publish`, `config.prod`)
- Write **functional tests** for critical user journeys — they run automatically in pipeline
- Use **Sling Job** API instead of custom schedulers for distributed processing
- Monitor via **New Relic** (integrated with AEMaaCS) for performance insights

---

## 🛠️ Hands-on Task

1. Study the Cloud Manager pipeline stages in Adobe's documentation
2. Run AEM Best Practices Analyzer on an existing AEM 6.5 project
3. Fix 2-3 AEM Analyzer warnings in a sample project
