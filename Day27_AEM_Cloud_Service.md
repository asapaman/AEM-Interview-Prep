# Day 27 — AEM as a Cloud Service (AEMaaCS)
**Target:** 2–4 YOE | Exclusive Cloud Focus

---

## 🌟 What is AEM as a Cloud Service?

**AEM as a Cloud Service (AEMaaCS)** is the cloud-native, SaaS version of AEM hosted entirely by Adobe on Microsoft Azure using Kubernetes. Unlike AEM 6.5 (which you install on your own servers or VMs), AEMaaCS is fully managed by Adobe.

### Why Move to Cloud Service?

| Concern | AEM 6.5 | AEM Cloud Service |
|---------|---------|-----------------|
| **Maintenance** | You manage AEM upgrades, patches, OS | Adobe manages everything |
| **Scaling** | Pre-provision servers (over/under-capacity) | Automatic elastic scaling |
| **High Availability** | Complex clustering setup | Built-in HA for Author/Publish |
| **Always Current** | Quarterly Service Pack (sometimes delayed) | Continuously updated (weekly) |
| **CDN** | Self-managed or 3rd party | Fastly CDN built in |
| **Security** | Your responsibility (OS, JVM, AEM patches) | Adobe-managed |
| **Cost Model** | License + infrastructure + ops team | Subscription (all-in) |

---

## 🏗️ AEM Cloud Architecture

```
                    ┌─────────────────────────────────┐
                    │         ADOBE MANAGED           │
                    │                                 │
Users               │  [Fastly CDN]                   │
  │                 │       ↓                         │
  └──────────────── │  [Dispatcher/CDN Edge]          │
                    │       ↓                         │
                    │  [Publish Tier]     [Author Tier]│
                    │  (Multiple pods,    (Multiple    │
                    │   auto-scaling)     pods, HA)    │
                    │       ↓                  ↓      │
                    │  [Azure Blob Storage - Shared]   │
                    │                                 │
                    └─────────────────────────────────┘
                              ↑
                    [Cloud Manager CI/CD Pipeline]
                    (Adobe-managed deployment)
                              ↑
                    [Your Git Repository]
                    (Customer-managed code)
```

---

## 🔑 Key Concept: Immutable Architecture

This is the **most important difference** to understand for interviews.

### What "Immutable" Means

In AEM Cloud, the repository is split into two zones:

```
IMMUTABLE (Read-Only after deployment)         MUTABLE (Writable at runtime)
─────────────────────────────────────          ──────────────────────────────
/apps/    ← Your components, OSGi configs      /content/    ← Authored pages
/libs/    ← Adobe's OOTB code                  /conf/       ← Templates, policies
/oak:index/  ← Search index definitions        /var/        ← Runtime data
/i18n/    ← Translations                       /home/       ← Users, groups
                                               /dam/        ← Digital assets
Code is deployed ONCE via Cloud Manager        Authors can change this at any time
and CANNOT be changed in production.
No Package Manager, No CRXDE writes to /apps
```

### What This Means in Practice

| AEM 6.5 (What you could do) | AEM Cloud (What you CAN'T do) |
|----|---|
| Install a content package with components via Package Manager | ❌ — must use Cloud Manager pipeline |
| Edit a component's HTL directly in CRXDE in production | ❌ — CRXDE is read-only for /apps |
| Upload an OSGi config via Felix Console | ❌ — configs must be in code |
| Manually start/stop OSGi bundles in production | ❌ — only via pipeline |

---

## 🚀 Cloud Manager — The Deployment Engine

**Cloud Manager** is Adobe's CI/CD platform. ALL code changes to AEM Cloud go through it.

### Pipeline Flow

```
Developer pushes code
        ↓
Git Repository (your GitHub/Azure DevOps/Bitbucket)
        ↓
Cloud Manager Full-Stack Pipeline
  Step 1: Code Build (Maven)
  Step 2: Unit Tests
  Step 3: Code Quality (SonarQube) — fails on critical issues
  Step 4: Deploy to DEV environment
  Step 5: Deploy to STAGE environment
  Step 6: [Optional] Performance Tests
  Step 7: Deploy to PRODUCTION
        ↓
Live in Production (typically 1-2 hours from commit)
```

### Pipeline Types

| Pipeline Type | Purpose |
|--------------|---------|
| **Full Stack Pipeline** | Deploys entire Maven project (all modules) |
| **Front-end Pipeline** | Deploys only frontend code (ui.frontend module) |
| **Web Tier Pipeline** | Deploys only Dispatcher/CDN configs |
| **Content Pipeline** | Deploys only content packages (ui.content) |

### Cloud Manager Quality Gates

Your code must pass these checks to deploy:

```
Code Quality Gate (SonarQube scan):
─────────────────────────────────────────
Reliability Rating: A (no bugs)
Security Rating:    A (no vulnerabilities)
Maintainability:    A (no code smells)
Coverage:          > 50% (unit test coverage)
Duplications:       < 5%

Common failures:
→ Null pointer that could cause NPE
→ Unused imports (code smells)
→ Hardcoded credentials (security)
→ Empty catch blocks (reliability)
→ Low unit test coverage
```

---

## 📁 AEM Cloud Project Structure (Maven Multi-Module)

```
mysite/                              ← Parent Maven project
  ├── pom.xml                        ← Parent POM (manages all modules)
  ├── core/                          ← Java code (OSGi bundle)
  │     ├── pom.xml
  │     └── src/main/java/com/mysite/core/
  │           ├── models/            ← Sling Models
  │           ├── services/          ← OSGi Services
  │           ├── servlets/          ← Sling Servlets
  │           └── workflows/         ← Workflow processes
  ├── ui.apps/                       ← /apps content (JCR structure)
  │     └── src/main/content/jcr_root/
  │           └── apps/mysite/
  │                 ├── components/
  │                 └── clientlibs/
  ├── ui.content/                    ← Mutable content (/content, /conf)
  │     └── src/main/content/jcr_root/
  │           ├── content/mysite/    ← Sample/initial content
  │           └── conf/mysite/       ← Templates, policies
  ├── ui.config/                     ← OSGi configurations
  │     └── src/main/content/jcr_root/
  │           └── apps/mysite/osgiconfig/
  │                 ├── config/           ← All run modes
  │                 ├── config.author/    ← Author only
  │                 ├── config.publish/   ← Publish only
  │                 └── config.prod/      ← Production only
  ├── ui.frontend/                   ← Frontend (Node/webpack)
  │     ├── webpack.config.js
  │     └── src/
  │           ├── styles/
  │           └── scripts/
  ├── ui.tests/                      ← Selenium/Cypress UI tests
  ├── it.tests/                      ← Integration tests
  └── dispatcher/                    ← Dispatcher/CDN configuration
        └── src/conf.dispatcher.d/
```

---

## ⚙️ OSGi Config in Cloud Service

In AEM Cloud, OSGi configs use **environment variable substitution**:

```json
// /apps/mysite/osgiconfig/config/com.mysite.core.services.impl.ApiServiceImpl.cfg.json
{
    "apiBaseUrl": "$[env:API_BASE_URL;default=https://api.mysite.com]",
    "apiKey": "$[secret:API_SECRET_KEY]",
    "timeout": 5000,
    "enabled": true
}
```

Environment variables are configured in Cloud Manager per environment:
- `API_BASE_URL` = `https://api.mysite.com` (non-secret)
- `API_SECRET_KEY` = `sk-abc123...` (secret — never stored in code!)

```
$[env:VAR_NAME]           → Non-sensitive env variable (visible in Cloud Manager)
$[env:VAR_NAME;default=x] → With fallback value
$[secret:SECRET_NAME]     → Secret variable (hidden, encrypted)
```

---

## 🔍 Repository Modernization — Oak Index

Search indexes in AEM Cloud must be defined in code (not via CRXDE/OOTB):

```xml
<!-- /apps/mysite/oak:index/mysite-products/oak:index/.content.xml -->
<jcr:root
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    xmlns:oak="http://jackrabbit.apache.org/oak/ns/1.0"

    jcr:primaryType="oak:QueryIndexDefinition"
    type="lucene"
    async="async"
    compatVersion="{Long}2">
  <indexRules jcr:primaryType="nt:unstructured">
    <nt:unstructured jcr:primaryType="nt:unstructured">
      <properties jcr:primaryType="nt:unstructured">
        <productId
            jcr:primaryType="nt:unstructured"
            name="productId"
            propertyIndex="{Boolean}true"
            analyzed="{Boolean}false"/>
        <jcr:primaryType
            jcr:primaryType="nt:unstructured"
            name="jcr:primaryType"
            propertyIndex="{Boolean}true"/>
      </properties>
    </nt:unstructured>
  </indexRules>
</jcr:root>
```

---

## 🛡️ AEM Cloud Development Best Practices (Critical!)

### 1. Package Design — Separate Mutable from Immutable

```xml
<!-- In ui.apps/pom.xml — contains ONLY /apps -->
<plugin>
    <groupId>org.apache.jackrabbit</groupId>
    <artifactId>filevault-package-maven-plugin</artifactId>
    <configuration>
        <packageType>application</packageType>  <!-- Immutable -->
    </configuration>
</plugin>

<!-- In ui.content/pom.xml — contains /content, /conf -->
<plugin>
    <configuration>
        <packageType>content</packageType>  <!-- Mutable -->
    </configuration>
</plugin>
```

### 2. Dynamic Media & Asset Compute

In Cloud Service, asset processing uses **Adobe Asset Compute** (serverless, on Firefly):
- Custom renditions are workers deployed to Adobe I/O Runtime
- NOT custom workflow steps as in AEM 6.5

### 3. Sling Context-Aware Configuration (CAConfig)

```java
// More powerful than OSGi config for per-site or per-path configuration
import org.apache.sling.caconfig.annotation.Configuration;
import org.apache.sling.caconfig.annotation.Property;

@Configuration(label = "My Site - Feature Config")
public @interface MySiteConfig {
    @Property(label = "Feature Enabled")
    boolean featureEnabled() default false;

    @Property(label = "API Endpoint")
    String apiEndpoint() default "https://api.mysite.com";
}
```

```java
@Model(adaptables = Resource.class, defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class FeatureModel {

    @SlingObject
    private Resource resource;

    @OSGiService
    private ConfigurationResolver configResolver;

    @PostConstruct
    protected void init() {
        // Get config scoped to current page's content tree
        MySiteConfig config = configResolver
            .get(resource)
            .as(MySiteConfig.class);

        boolean enabled = config.featureEnabled();
        String apiEndpoint = config.apiEndpoint();
    }
}
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is the most important architectural difference between AEM 6.5 and AEM Cloud Service?**

> **Answer:** The **immutable architecture**. In AEM Cloud, the code layer (`/apps`) is deployed ONCE via Cloud Manager and becomes read-only after deployment. There's no Package Manager in production, no CRXDE writes to `/apps`. Every code change must go through Cloud Manager CI/CD pipeline. This enforces discipline — no "just this once" manual changes in production. All mutable data (`/content`, `/conf`) remains editable by authors.

**Q2. What is Cloud Manager and why is it essential for AEM Cloud?**

> **Answer:** Cloud Manager is Adobe's CI/CD platform and the ONLY way to deploy code to AEM Cloud environments. It runs: Maven build → Unit tests → Code quality checks (SonarQube) → Deployment to dev/stage/prod. It also manages environment variables, secrets, Dispatcher configs, and CDN rules. In AEM 6.5, you could bypass CI/CD with Package Manager; in Cloud, every change must pass through Cloud Manager's gates — enforcing code quality.

**Q3. What are the Cloud Manager quality gates and what happens if code fails them?**

> **Answer:** Cloud Manager runs SonarQube analysis with thresholds: Reliability Rating A (no bugs), Security Rating A (no vulnerabilities), Maintainability A (no major code smells), unit test coverage > 50%. If code fails CRITICAL or BLOCKER severity issues, the pipeline fails and deployment is STOPPED. The developer must fix the issue and push again. This cannot be bypassed for production deployments.

**Q4. How do you handle environment-specific configuration in AEM Cloud?**

> **Answer:** Use run-mode config folders (`config.author/`, `config.publish/`, `config.prod/`) with JSON config files that use `$[env:VAR_NAME]` substitution for non-sensitive values and `$[secret:SECRET_NAME]` for credentials. The actual values are set in Cloud Manager's environment configuration UI — they're never stored in code. This means your code repo has no secrets, and different environments (dev/stage/prod) can have different API endpoints and credentials without code changes.

**Q5. What is the `ui.config` module used for in an AEM Cloud project?**

> **Answer:** The `ui.config` module contains OSGi configuration JSON files. It's a separate Maven module (from `ui.apps` and `ui.content`) because configs need to be deployed in a specific order and are considered "application configuration" — separate from component code and content. The config files go to `/apps/mysite/osgiconfig/config*/` within this module.

**Q6. What cannot be done in AEM Cloud that is possible in AEM 6.5?**

> **Answer:**
> - ❌ Install packages via Package Manager in production (no UI available)
> - ❌ Edit files in CRXDE `/apps/` — immutable after deployment
> - ❌ Upload OSGi configs via Felix Console — must be code
> - ❌ Start/stop individual bundles manually in production
> - ❌ Custom persistent installations (`install/` folders) in JCR
> - ❌ Direct JVM access (no SSH to production servers)
> - ❌ Custom Adobe Asset Compute workers in OSGi — must be Adobe I/O Runtime

---

## ✅ Best Practices for AEM Cloud Development

1. **Never use Package Manager** for production deploys — always Cloud Manager pipeline
2. **All configs in code** — use `$[secret:]` for credentials, never hardcode
3. **Separate ui.apps from ui.content** — never mix code and mutable content
4. **Test locally with AEM Cloud SDK** (Adobe provides a local quickstart JAR simulating Cloud behavior)
5. **Write unit tests** — Cloud Manager fails if coverage is below threshold
6. **Fix code quality issues early** — don't let technical debt accumulate (pipeline will block you)
7. **Use Oak indexes in code** — don't create indexes via CRXDE in Cloud
8. **Monitor Cloud Manager logs** — first place to check when deployment fails

---

## 🛠️ Hands-on Practice

1. **Download AEM Cloud SDK** from Adobe Software Distribution
2. **Explore the project structure:** Use `mvn -B archetype:generate -D archetypeGroupId=com.adobe.aem -D archetypeArtifactId=aem-project-archetype` to scaffold a project
3. **Identify the modules:** `core/`, `ui.apps/`, `ui.content/`, `ui.config/`, `ui.frontend/`
4. **Add an OSGi config** with env variable substitution in `ui.config/`
5. **Check Cloud Manager quality:** Run `mvn sonar:sonar` locally (or push to Cloud Manager dev env)
6. **Practice the rule:** Make a change → Cloud Manager pipeline → verify it works
