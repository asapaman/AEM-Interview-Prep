# AEM 30-Day Interview Preparation Guide
**Target:** 2–4 YOE AEM Developers | **Versions:** AEM 6.5 & AEM as a Cloud Service

---

> 📌 This guide is designed to take you from **intermediate to senior-level** AEM knowledge. Even if you're a fresher, following this guide will give you the complete foundation for AEM development. Each file contains: **What is it? → Why it matters → How it works → Code examples → Interview Q&A → Best Practices → Hands-on Tasks**.

---

## 📂 File Index

| Day | File | Topics | Difficulty |
|-----|------|--------|-----------|
| 1 | [Day01_AEM_Architecture.md](Day01_AEM_Architecture.md) | Architecture layers, JCR, Sling, OSGi, Author/Publish/Dispatcher, request flow, run modes | 🟡 Medium |
| 2 | [Day02_Component_Development.md](Day02_Component_Development.md) | Component structure, dialog fields, sling:resourceType, sling:resourceSuperType, HTL basics | 🟡 Medium |
| 3 | [Day03_Complex_Components.md](Day03_Complex_Components.md) | Nested multifields, composite dialogs, API integration, CTA link handling, performance | 🔴 Advanced |
| 4 | [Day04_Editable_Templates.md](Day04_Editable_Templates.md) | Static vs Editable templates, Structure/Initial/Policies layers, Template Types | 🟡 Medium |
| 5 | [Day05_Policies_Component_Restrictions.md](Day05_Policies_Component_Restrictions.md) | Content Policies, Style System, allowed components, policy vs design_dialog | 🟡 Medium |
| 6–7 | [Day06_07_Client_Libraries.md](Day06_07_Client_Libraries.md) | Clientlibs, allowProxy, categories, css.txt/js.txt, embed vs dependencies, debugging | 🟢 Foundational |
| 8–11 | [Day08_11_Sling_Models.md](Day08_11_Sling_Models.md) | All annotations, Resource vs Request adaptable, @PostConstruct, multifield mapping, debugging | 🔴 Advanced |
| 12–15 | [Day12_15_OSGi_Services.md](Day12_15_OSGi_Services.md) | @Component, @Activate/@Modified/@Deactivate, @Reference, OSGi configs, memory leaks | 🔴 Advanced |
| 16–19 | [Day16_19_Servlets.md](Day16_19_Servlets.md) | SlingSafeMethodsServlet, path vs resource-type, GET/POST, CSRF, forward/redirect | 🟡 Medium |
| 20–21 | [Day20_21_HTL_Sightly.md](Day20_21_HTL_Sightly.md) | All data-sly-* statements, context-aware escaping, data-sly-template, real examples | 🟡 Medium |
| 22 | [Day22_Workflow_Scheduler.md](Day22_Workflow_Scheduler.md) | WorkflowProcess, OSGi Scheduler, cron syntax, memory leak prevention | 🔴 Advanced |
| 23 | [Day23_Dialog_Validation.md](Day23_Dialog_Validation.md) | Granite required fields, custom JS validation, field dependencies, show/hide | 🟡 Medium |
| 24–25 | [Day24_25_MSM_Translation.md](Day24_25_MSM_Translation.md) | Blueprint, Live Copy, rollout configs, Language Manager, Translation projects | 🔴 Advanced |
| 26 | [Day26_Permissions_Security.md](Day26_Permissions_Security.md) | Service users, ACLs, JCR privileges, ResourceResolverFactory, Dispatcher security | 🔴 Advanced |
| 27 | [Day27_AEM_Cloud_Service.md](Day27_AEM_Cloud_Service.md) | Immutable architecture, Cloud Manager, pipelines, env variables, project structure | 🔴 Advanced |
| 28–30 | [Day28_30_CF_XF_Exporter.md](Day28_30_CF_XF_Exporter.md) | Content Fragments, CF Model, Experience Fragments, Sling Model Exporter, GraphQL | 🔴 Advanced |
| Bonus | [Additional_Important_Questions.md](Additional_Important_Questions.md) | Dispatcher caching deep-dive, run modes, memory leaks, debug checklist, quick-fire Q&A | 🔴 Advanced |

---

## 🎯 Preparation Strategy

### Week 1 — Foundation (Days 1–7)
Focus on understanding the fundamentals. These are asked in EVERY AEM interview.

| Priority | Topics | Why |
|----------|--------|-----|
| ⭐⭐⭐ | AEM Architecture (Day 1) | Asked in every single AEM interview |
| ⭐⭐⭐ | Component Development (Day 2) | Core daily work |
| ⭐⭐⭐ | Client Libraries (Days 6–7) | Frequently asked, often misunderstood |
| ⭐⭐ | Editable Templates (Day 4) | Required for modern AEM |

### Week 2 — Java & Back-end (Days 8–19)
The most technically heavy section. Focus on Sling Models and OSGi.

| Priority | Topics | Why |
|----------|--------|-----|
| ⭐⭐⭐ | Sling Models (Days 8–11) | Deep-dive by every interviewer |
| ⭐⭐⭐ | OSGi Services (Days 12–15) | Service architecture questions |
| ⭐⭐ | Servlets (Days 16–19) | API development pattern |

### Week 3 — HTL, Security & Advanced (Days 20–27)
Separates 2 YOE from 4 YOE candidates.

| Priority | Topics | Why |
|----------|--------|-----|
| ⭐⭐⭐ | HTL Escaping (Days 20–21) | Security + practical knowledge |
| ⭐⭐⭐ | Permissions & Service Users (Day 26) | Senior-level must-know |
| ⭐⭐⭐ | AEM Cloud (Day 27) | Increasingly required |
| ⭐⭐ | Workflows & Schedulers (Day 22) | Common in large projects |

### Week 4 — Headless & Synthesis (Days 28–30 + Bonus)

| Priority | Topics | Why |
|----------|--------|-----|
| ⭐⭐⭐ | Content Fragments + GraphQL (Days 28–30) | Hot topic — headless AEM |
| ⭐⭐⭐ | Additional Questions (Bonus) | Mock interview preparation |
| ⭐⭐ | MSM & Translation (Days 24–25) | Enterprise project knowledge |

---

## 🔥 Top 15 Most-Asked AEM Interview Questions

These come up in nearly every AEM interview. Know them cold.

1. **Explain AEM request flow end-to-end** → Day 1
2. **What is sling:resourceType and why is it important?** → Day 2
3. **What is the difference between Resource.class and SlingHttpServletRequest.class adaptable?** → Day 8
4. **Why use DefaultInjectionStrategy.OPTIONAL?** → Day 8
5. **What is a Service User and why not use admin session?** → Day 26
6. **What are the three layers of an Editable Template?** → Day 4
7. **What is allowProxy in clientlibs and why is it needed?** → Day 6
8. **What is the difference between @Activate, @Modified, and @Deactivate?** → Day 12
9. **What is context-aware escaping in HTL?** → Day 20
10. **How does Dispatcher caching work?** → Additional Questions
11. **What is the difference between AEM 6.5 and AEM Cloud Service?** → Day 27
12. **What is a Content Fragment vs Experience Fragment?** → Day 28
13. **How do you map a composite multifield to Java POJOs?** → Days 3, 8
14. **What happens when you lock/unlock inheritance in a Live Copy?** → Day 24
15. **What is the immutable architecture in AEM Cloud?** → Day 27

---

## 💡 Key Patterns to Memorize

### Service User Pattern
```java
Map<String, Object> params = Map.of(ResourceResolverFactory.SUBSERVICE, "my-subservice");
try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(params)) {
    // use resolver
} catch (LoginException e) { LOG.error("...", e); }
```

### Sling Model Pattern
```java
@Model(adaptables = Resource.class, defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class MyModel {
    @ValueMapValue private String title;
    @PostConstruct protected void init() { /* process data */ }
    public String getTitle() { return title; }
}
```

### OSGi Service Pattern
```java
@Component(service = MyService.class)
@Designate(ocd = MyConfig.class)
public class MyServiceImpl implements MyService {
    @Activate protected void activate(MyConfig config) { /* load config */ }
    @Modified protected void modified(MyConfig config) { activate(config); }
    @Deactivate protected void deactivate() { /* cleanup */ }
}
```

### HTL Pattern
```html
<sly data-sly-use.model="com.mysite.core.models.MyModel"/>
<div data-sly-test="${model.hasContent}">
    <h2>${model.title}</h2>
    <a href="${model.link @ context='uri'}">${model.ctaLabel}</a>
</div>
```

---

## ⚠️ Common Mistakes to Avoid in Interviews

| ❌ Wrong | ✅ Correct |
|---------|----------|
| `loginAdministrative()` | `getServiceResourceResolver()` with service user |
| `DefaultInjectionStrategy.REQUIRED` | `DefaultInjectionStrategy.OPTIONAL` |
| `@Inject` for everything | Specific: `@ValueMapValue`, `@OSGiService`, `@SlingObject` |
| Not closing ResourceResolver | `try (ResourceResolver resolver = ...)` |
| Storing ResourceResolver as field | Get-and-close per method |
| Modifying `/libs` directly | Overlay to `/apps` |
| JSP scriptlets | HTL + Sling Model |
| Path-based servlet for component API | Resource-type servlet |
| `context='unsafe'` in HTL | Never use unsafe |
| Skipping `@Deactivate` cleanup | Always cleanup in `@Deactivate` |

---

## 🏆 Power Phrases for Interviews

Use these to instantly sound like a senior developer:

> *"I always use `DefaultInjectionStrategy.OPTIONAL` — a null injection silently degrading is better than the entire page crashing for all users..."*

> *"For service users, I follow the principle of least privilege — each subservice name maps to a system user with ONLY the JCR paths it needs..."*

> *"In AEM Cloud, the immutable architecture means everything goes through Cloud Manager — there's no Package Manager in production, and CRXDE is read-only for /apps..."*

> *"I prefer resource-type servlet registration — it inherits the content node's ACLs and doesn't require explicit Dispatcher rules for /bin/ paths..."*

> *"In `@Deactivate`, I always unschedule jobs, close HTTP clients, and clear caches — failing to do this causes memory leaks that accumulate until OOM crashes the JVM..."*

> *"When extending Core Components, I use `@Self @Via(type = ResourceSuperType.class)` to delegate to the core model, inheriting all its behavior while only adding my custom fields..."*

---

## 📚 Additional Resources

| Resource | URL |
|----------|-----|
| AEM Documentation | https://experienceleague.adobe.com/docs/experience-manager.html |
| AEM Core Components | https://github.com/adobe/aem-core-wcm-components |
| AEM CIF Components | https://github.com/adobe/aem-cif-project-archetype |
| AEM Project Archetype | https://github.com/adobe/aem-project-archetype |
| AEM Dispatcher Tools | https://docs.adobe.com/content/help/en/experience-manager-learn/ams/dispatcher/overview.html |
| GROQ Syntax Reference | https://www.sanity.io/docs/groq |
| HTL Specification | https://github.com/adobe/htl-spec |

---

*Generated for a 4 YOE AEM Developer. AEM 6.5 + AEM as a Cloud Service focus. Covers architecture, development patterns, security, and headless delivery.*
