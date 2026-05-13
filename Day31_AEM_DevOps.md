# AEM DevOps — Cloud Manager & Deployment
**Target:** 2–4 YOE | **Versions:** AEM Cloud Service

---

## 🌟 What is Cloud Manager?

**Cloud Manager** is Adobe's CI/CD platform for AEM as a Cloud Service. Every code change, configuration update, and content deployment goes through Cloud Manager. It provides:

- **Pipelines** — Automated build, test, and deploy workflows
- **Environments** — Dev, Stage, Production management
- **Quality Gates** — Code quality checks that block bad deployments
- **RDE** — Rapid Development Environment for fast developer iteration
- **Log access** — Download and tail AEM logs from any environment

---

## 🔁 Cloud Manager Pipeline Types

### Full Stack Pipeline
Builds and deploys everything: Java code, content packages, OSGi configs, Dispatcher configs, frontend code.

```
Code Push to Git
    │
    ▼
Cloud Manager Full Stack Pipeline:
  1. Build (mvn clean package)
  2. Unit Tests (JUnit + AEM Mocks)
  3. Code Quality Scan (SonarQube)
  4. Security Testing
  5. Deploy to Dev/Stage
  6. Performance Testing (Load tests against Stage)
  7. Deploy to Production (after approval)
```

### Web Tier Pipeline (Dispatcher Only)
Deploys ONLY Dispatcher/CDN configuration changes — no Java recompile needed.

```
Dispatcher config change in Git
    │
    ▼
Cloud Manager Web Tier Pipeline:
  1. Validate Dispatcher config syntax
  2. Deploy Dispatcher rules to CDN
  3. Live in ~5 minutes (vs 30+ min for Full Stack)
```

### Config Pipeline
Deploys ONLY OSGi configuration files (`*.cfg.json`) — no code changes.

```
OSGi config change in Git
    │
    ▼
Cloud Manager Config Pipeline:
  1. Validate config format
  2. Deploy configs to target environment
```

### Frontend Pipeline
Builds and deploys ONLY frontend assets (CSS, JS from ui.frontend module).

---

## 🏗️ Environment Types

| Environment | Purpose | Features |
|-------------|---------|---------|
| **RDE** (Rapid Dev Env) | Developer sandbox | Instant deploy, no pipeline |
| **Development** | Team development | Full pipeline, mutable content |
| **Stage** | Pre-production testing | Performance testing, content mirror |
| **Production** | Live site | Requires approval, auto-scaled |

---

## ⚡ RDE — Rapid Development Environment

**RDE (Rapid Development Environment)** lets developers deploy code changes directly WITHOUT running the full Cloud Manager pipeline — dramatically speeding up the dev-test cycle.

### RDE Setup

```bash
# Install Cloud Manager CLI
npm install -g @adobe/aio-cli
aio plugins:install @adobe/aio-cli-plugin-cloudmanager

# Login
aio login

# Configure RDE context
aio cloudmanager:set-context \
    --programId <your-program-id> \
    --environmentId <rde-environment-id>
```

### RDE Deploy Commands

```bash
# Deploy an OSGi bundle
aio aem:rde:deploy mysite.core-1.0.0-SNAPSHOT.jar

# Deploy a content package (ui.apps)
aio aem:rde:deploy mysite.ui.apps-1.0.0-SNAPSHOT.zip

# Deploy Dispatcher config
aio aem:rde:deploy mysite.dispatcher.zip --type dispatcher-config

# Deploy ALL at once (all module packages)
aio aem:rde:deploy mysite.all-1.0.0-SNAPSHOT.zip

# Check RDE status
aio aem:rde:status

# View recent deploy history
aio aem:rde:history

# Reset RDE to baseline (clean slate)
aio aem:rde:reset
```

### RDE Workflow for Fast Development

```
1. Make Java code change locally
2. mvn clean install -pl core (build only core, ~30 sec)
3. aio aem:rde:deploy core/target/mysite.core-*.jar  (~10 sec)
4. Test change on RDE URL
5. Iterate until satisfied
6. git push → trigger full pipeline for proper testing
```

**Speed comparison:**
- Full Stack Pipeline: 30–60 minutes
- RDE deploy: ~1 minute ← use this during development!

---

## 🔐 Environment Variables & Secrets

In AEM Cloud, you must NEVER hardcode credentials or environment-specific values in your code. Use Cloud Manager environment variables.

### Types of Variables

| Type | Visibility | Use Case |
|------|-----------|---------|
| **Variable** | Visible in logs | Non-sensitive config (URLs, timeout values) |
| **Secret** | Hidden in logs | API keys, passwords, tokens |

### Setting Variables in Cloud Manager

```
Cloud Manager → Environment → ... → Environment Variables
→ + Add Variable
→ Name: EXTERNAL_API_KEY
→ Value: sk_live_xxxx
→ Type: Secret
→ Service: Author + Publish (or just one)
→ Save
```

### Reading Variables in OSGi Config

```json
// /apps/mysite/osgiconfig/config/com.mysite.core.services.ApiServiceImpl.cfg.json
{
    "apiBaseUrl":  "$[env:EXTERNAL_API_URL;default=https://api.mysite.com]",
    "apiKey":      "$[secret:EXTERNAL_API_KEY]",
    "timeout":     "$[env:API_TIMEOUT;type=Integer;default=5000]"
}
```

```
Syntax:
$[env:VARIABLE_NAME]                    ← Environment variable (visible)
$[secret:SECRET_NAME]                   ← Secret (hidden in logs)
$[env:VAR;default=fallback-value]       ← With default value
$[env:VAR;type=Integer;default=5000]    ← With type coercion
```

### Reading Variables in Java

```java
// Variables are injected into OSGi configs, then into your service via @Activate
@Activate
protected void activate(Config config) {
    this.apiKey = config.apiKey();     // Reads the secret at activation time
    this.apiUrl = config.apiBaseUrl(); // Reads the variable at activation time
}

@ObjectClassDefinition
public @interface Config {
    String apiBaseUrl() default "https://api.mysite.com";
    String apiKey() default "";  // Will be overridden by secret at runtime
    int timeout() default 5000;
}
```

---

## 📊 Quality Gates — SonarQube

Cloud Manager runs SonarQube static code analysis on every pipeline run. Failures BLOCK deployment.

### Key Quality Metrics

| Metric | Description | Typical Threshold |
|--------|-------------|------------------|
| **Code Coverage** | % of code covered by unit tests | ≥ 50% (configurable) |
| **Critical Issues** | Code bugs that will cause problems | 0 |
| **Major Issues** | Code smells, bad practices | Defined threshold |
| **Security Issues** | Security vulnerabilities (OWASP) | 0 Critical |
| **Reliability** | Bug density | ≥ A rating |
| **Maintainability** | Technical debt | ≤ 5 days |

### Common SonarQube Issues in AEM

```java
// ❌ SONAR ISSUE: String concatenation in loop (performance)
for (String item : items) {
    result = result + item;  // SQ: Use StringBuilder
}
// ✅ Fix:
StringBuilder sb = new StringBuilder();
for (String item : items) {
    sb.append(item);
}
String result = sb.toString();

// ❌ SONAR ISSUE: Empty catch block
try { ... }
catch (Exception e) {}  // SQ: Empty catch — at least log it!
// ✅ Fix:
catch (Exception e) { LOG.error("Failed to ...", e); }

// ❌ SONAR ISSUE: Null pointer risk
String value = map.get("key").toString();  // SQ: NPE if get() returns null
// ✅ Fix:
Object obj = map.get("key");
String value = obj != null ? obj.toString() : null;

// ❌ SONAR ISSUE: Resource leak
ResourceResolver resolver = resolverFactory.getServiceResourceResolver(params);
// ... (no close!)
// ✅ Fix: try-with-resources
```

---

## 📋 Log Access & Management

### Viewing Logs in Cloud Manager

```
Cloud Manager → Environments → Select Environment
→ ... → Download Logs
→ Select service (Author / Publish / Dispatcher)
→ Download: error.log, request.log, access.log

OR: Use the Cloud Manager CLI
aio cloudmanager:download-log <environment-id> author aem error.log
```

### Log Forwarding to Splunk/Datadog

```
Cloud Manager → Environments → Select Environment
→ Settings → Log Forwarding
→ Configure: Splunk / Azure EventHub / Datadog / HTTPS endpoint
→ All logs streamed in real-time
```

### Custom Log Levels in AEM Cloud

```json
// OSGi config: org.apache.sling.commons.log.LogManager.factory.config-mysite.cfg.json
{
    "org.apache.sling.commons.log.level": "DEBUG",
    "org.apache.sling.commons.log.names": [
        "com.mysite.core",
        "com.mysite.core.services",
        "com.mysite.core.models"
    ],
    "org.apache.sling.commons.log.file": "logs/mysite.log"
}
```

---

## 🔄 Cloud Manager API — Automation

Cloud Manager has a full REST API for automation:

```bash
# List all pipelines
curl -X GET \
    -H "Authorization: Bearer $ACCESS_TOKEN" \
    "https://cloudmanager.adobe.io/api/program/{programId}/pipelines"

# Trigger a pipeline
curl -X PUT \
    -H "Authorization: Bearer $ACCESS_TOKEN" \
    -H "x-gw-ims-org-id: $ORG_ID" \
    "https://cloudmanager.adobe.io/api/program/{programId}/pipeline/{pipelineId}/execution"

# Get pipeline execution status
curl -X GET \
    -H "Authorization: Bearer $ACCESS_TOKEN" \
    "https://cloudmanager.adobe.io/api/program/{programId}/pipeline/{pipelineId}/executions"
```

---

## 🌐 AEM Cloud Deployment Architecture

```
Developer Machine
  │  git push
  ▼
Git Repository (GitHub/GitLab/Azure DevOps)
  │  Webhook
  ▼
Cloud Manager Pipeline
  │  Build → Quality → Test → Deploy
  ▼
AEM Cloud Service Infrastructure (Adobe-managed)
  ├── Author instances (2+ replicas, auto-scaled)
  ├── Publish instances (2+ replicas, auto-scaled)
  └── Dispatcher/Fastly CDN
        └── User Traffic
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is an RDE and how does it improve developer velocity?**

> **Answer:** RDE (Rapid Development Environment) is an AEM Cloud instance where developers can deploy code changes directly using the Cloud Manager CLI (`aio aem:rde:deploy`) — bypassing the full Cloud Manager pipeline. A full pipeline takes 30–60 minutes; an RDE deploy takes under 2 minutes. This allows rapid iteration: code → build locally → deploy to RDE → test → repeat. When the feature is ready, push to Git and run the full pipeline for proper quality/security checks.

**Q2. What are Cloud Manager environment variables and how do you use them in AEM?**

> **Answer:** Cloud Manager environment variables (set in the Cloud Manager UI per environment) are injected into AEM's OSGi configuration system using the `$[env:VAR_NAME]` substitution syntax. Secrets use `$[secret:SECRET_NAME]`. This means your OSGi config files in Git don't contain actual credentials — they contain placeholders that Cloud Manager fills in at deploy time for each environment. Developers can't see secret values even via Cloud Manager UI.

**Q3. What is a Quality Gate in Cloud Manager and what happens if it fails?**

> **Answer:** Quality Gates are SonarQube-based code analysis checks that run on every full-stack pipeline: code coverage (unit tests), critical/major code issues, security vulnerabilities, and reliability rating. If a gate fails, the pipeline STOPS and does NOT deploy to the next environment. The team must fix the issues before the pipeline can proceed. Thresholds are configurable per program via Cloud Manager quality profile settings.

**Q4. What is the difference between a Full Stack Pipeline and a Web Tier Pipeline?**

> **Answer:** A Full Stack Pipeline builds and deploys everything: Java (core), content packages (ui.apps, ui.content), OSGi configs (ui.config), frontend (ui.frontend), and Dispatcher configs. Takes 30–60 minutes. A Web Tier Pipeline deploys ONLY Dispatcher/CDN configuration — no Java build needed. Takes ~5 minutes. Use Web Tier for pure Dispatcher changes (filter rules, cache configs, vhosts) to avoid the long full stack build cycle.

---

## ✅ Best Practices

1. **Use RDE for all development iteration** — never run full pipeline just to test a code change
2. **Never commit secrets** to Git — always use Cloud Manager secret variables
3. **Fix SonarQube issues before code review** — don't let quality gate failures become blockers at deploy time
4. **Use Config Pipeline** for OSGi-only config changes — faster than full stack
5. **Set up log forwarding** to a log management platform (Splunk/Datadog) for production
6. **Use environment-specific OSGi configs** (`config.dev/`, `config.stage/`) — don't hardcode environment differences in Java
7. **Pin Java version** in Cloud Manager pipeline settings to match your `pom.xml`

---

## 🛠️ Hands-on Practice

1. Install the Cloud Manager CLI and authenticate: `npm install -g @adobe/aio-cli`
2. If you have an AEM Cloud trial: Set up an RDE and deploy your core bundle via CLI
3. Set a Cloud Manager environment variable (non-secret) and read it via OSGi config substitution
4. Look at your project's `pom.xml` — find where the Jacoco code coverage plugin is configured
5. Run `mvn clean install` and check the SonarQube/Jacoco report in `core/target/site/jacoco/index.html`
