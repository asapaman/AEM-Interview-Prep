# 🚀 AEM 30-Day Interview Preparation Guide
### For 4 Years of Experience (4 YOE) AEM Developer

> Each file contains: Topic Explanation • Subtopics • Code Examples • Interview Q&A • Best Practices • Hands-on Tasks

---

## 📚 File Index

| Day(s) | Topic | Difficulty | File |
|--------|-------|-----------|------|
| 1 | AEM Architecture | Medium | [Day01_AEM_Architecture.md](./Day01_AEM_Architecture.md) |
| 2 | Component Development | Medium | [Day02_Component_Development.md](./Day02_Component_Development.md) |
| 3 | Complex Components | Hard | [Day03_Complex_Components.md](./Day03_Complex_Components.md) |
| 4 | Editable Templates | Hard | [Day04_Editable_Templates.md](./Day04_Editable_Templates.md) |
| 5 | Policies & Component Restrictions | Hard | [Day05_Policies_Component_Restrictions.md](./Day05_Policies_Component_Restrictions.md) |
| 6–7 | Client Libraries (Clientlibs) | Medium | [Day06_07_Client_Libraries.md](./Day06_07_Client_Libraries.md) |
| 8–11 | Sling Models (Complete Guide) | Medium–Hard | [Day08_11_Sling_Models.md](./Day08_11_Sling_Models.md) |
| 12–15 | OSGi & Services | Hard | [Day12_15_OSGi_Services.md](./Day12_15_OSGi_Services.md) |
| 16–19 | Servlets | Hard | [Day16_19_Servlets.md](./Day16_19_Servlets.md) |
| 20–21 | HTL / Sightly | Medium–Hard | [Day20_21_HTL_Sightly.md](./Day20_21_HTL_Sightly.md) |
| 22 | Workflow & Scheduler | Hard | [Day22_Workflow_Scheduler.md](./Day22_Workflow_Scheduler.md) |
| 23 | Dialog Validation | Medium | [Day23_Dialog_Validation.md](./Day23_Dialog_Validation.md) |
| 24–25 | MSM & Translation | Hard | [Day24_25_MSM_Translation.md](./Day24_25_MSM_Translation.md) |
| 26 | Permissions, Users & Security | Medium | [Day26_Permissions_Security.md](./Day26_Permissions_Security.md) |
| 27 | AEM Cloud Service | Hard | [Day27_AEM_Cloud_Service.md](./Day27_AEM_Cloud_Service.md) |
| 28–30 | Content Fragments, XF & Exporter | Medium–Hard | [Day28_30_CF_XF_Exporter.md](./Day28_30_CF_XF_Exporter.md) |
| Bonus | Additional Important Questions | — | [Additional_Important_Questions.md](./Additional_Important_Questions.md) |

---

## 🎯 Key Topics Covered Per File

### Day 1 — AEM Architecture
- Author vs Publish vs Dispatcher
- JCR, Sling, OSGi, Oak
- Request flow end-to-end
- CDN, TarMK vs MongoMK, Run modes

### Day 2 — Component Development
- sling:resourceType, sling:resourceSuperType
- Component structure, cq:dialog, HTL integration
- Core Components inheritance

### Day 3 — Complex Components
- Nested multifield (dialog + Sling Model)
- CTA/Link handling (internal vs external)
- API integration in components

### Day 4 — Editable Templates
- Structure vs Initial Content vs Policies
- Template creation, enabling, allowed templates
- Template path configuration

### Day 5 — Policies & Component Restrictions
- Page Policy vs Content Policy
- Layout Container component restrictions
- Style System

### Day 6–7 — Client Libraries
- allowProxy, js.txt, css.txt
- `embed` vs `dependencies`
- Including clientlibs in HTL
- Author-only vs Publish clientlibs

### Day 8–11 — Sling Models
- All annotations: `@ValueMapValue`, `@Inject`, `@ChildResource`, `@OSGiService`, `@Self`, `@Via`
- `Resource.class` vs `SlingHttpServletRequest.class`
- `@PostConstruct` usage
- Nested multifield mapping

### Day 12–15 — OSGi & Services
- `@Component`, `@Activate`, `@Deactivate`, `@Modified`, `@Reference`
- OSGi Component vs Service
- `@ObjectClassDefinition` (configurable services)
- Inter-service communication
- Environment-specific OSGi configs

### Day 16–19 — Servlets
- `SlingSafeMethodsServlet` vs `SlingAllMethodsServlet`
- Path-based vs Resource-type servlet
- GET, POST, PUT, DELETE handling
- CSRF protection, Forward vs Redirect
- Sling POST Servlet

### Day 20–21 — HTL / Sightly
- All `data-sly-*` statements with examples
- Context-aware escaping (uri, html, scriptString)
- `data-sly-template` + `data-sly-call` (reusable snippets)
- `data-sly-include` vs `data-sly-resource`

### Day 22 — Workflow & Scheduler
- Custom Workflow Process Step implementation
- Workflow Launcher configuration
- OSGi Scheduler with cron expressions
- Memory leak prevention in schedulers

### Day 23 — Dialog Validation
- Built-in Granite validation (required, maxlength)
- Custom JavaScript validators (`$.validator.register`)
- `cq.authoring.dialog` clientlib category

### Day 24–25 — MSM & Translation
- Blueprint, Live Copy, Rollout
- MSM + Translation combined pattern
- Language Copy structure
- Breaking inheritance

### Day 26 — Permissions, Users & Security
- Service users (mandatory for 4 YOE!)
- `ResourceResolverFactory` usage
- JCR ACLs, `rep:GrantACE`, `rep:DenyACE`
- Principle of least privilege

### Day 27 — AEM Cloud Service
- Cloud vs On-Premise comparison
- Immutable architecture
- Cloud Manager pipelines
- AEM Analyzer Maven plugin
- RDE (Rapid Development Environment)

### Day 28–30 — Content Fragments, XF & Exporter
- CF vs XF comparison
- Reading CF in Sling Model (ContentFragment API)
- CF Variations
- Sling Model Exporter (`@Exporter`, `.model.json`)
- GraphQL for CF (AEMaaCS)

### Additional Important Questions
- Static vs Editable Templates
- Dispatcher caching deep dive
- AEM Run modes
- cq:dialog vs cq:design_dialog
- Debugging Sling Models (checklist)
- ResourceResolverFactory patterns
- Memory leak prevention (5 rules)
- Forward vs Redirect
- Service user mapping
- Replication queue
- Quick-fire Q&A reference table

---

## ⚡ Interview Day Quick Reference

### Phrases that impress at 4 YOE:
- *"I always use `DefaultInjectionStrategy.OPTIONAL` to avoid null injection failures..."*
- *"For service users, I follow principle of least privilege and configure via `ServiceUserMapperImpl`..."*
- *"In AEMaaCS, the immutable architecture means we can't deploy via Package Manager in prod..."*
- *"I prefer Resource-type servlet registration over path-based for better security..."*
- *"For complex components, I separate data fetching (Service) from modeling (Sling Model) from rendering (HTL)..."*

### Topics most frequently asked at 4 YOE:
1. Sling Models (annotations, adaptables)
2. OSGi Services (lifecycle, configuration)
3. Component development (complex, nested multifield)
4. Servlets (path vs resource-type)
5. AEM Architecture (request flow)
6. Service Users & Security
7. Editable Templates vs Static Templates
8. AEM Cloud Service differences
