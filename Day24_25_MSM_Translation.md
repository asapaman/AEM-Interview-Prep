# Day 24 & 25 — MSM & Translation
**Difficulty:** Hard | **4 YOE Focus**

---

## 📖 MSM (Multi Site Manager)

MSM allows you to reuse and manage content across multiple websites — typically for multi-region/multi-language sites.

### Core MSM Concepts

| Concept | Description |
|---------|------------|
| **Blueprint** | The source/master site (e.g., English US) |
| **Live Copy** | A copy of blueprint content, linked to it |
| **Rollout** | Push changes from Blueprint to Live Copy |
| **Synchronization** | Keep Live Copy in sync with Blueprint |
| **Inheritance** | Live Copy inherits Blueprint content by default |
| **Break Inheritance** | Decouple specific component/page from Blueprint |

### MSM Relationship Flow
```
/content/mysite/us/en/     ← Blueprint (Source)
      ↓ rollout
/content/mysite/uk/en/     ← Live Copy (UK)
/content/mysite/au/en/     ← Live Copy (Australia)
/content/mysite/ca/en/     ← Live Copy (Canada)
```

---

## 🔧 Creating a Live Copy

**Via UI:** Sites Console → Select Blueprint page → Create → Live Copy
**Via CRXDE:** Create `/content/target/` with `cq:LiveRelationshipManager` mixin

### JCR Properties on Live Copy
```xml
<!-- On Live Copy page's jcr:content -->
<jcr:content
    cq:LiveCopyConfig="/content/blueprint"
    cq:isSyncDisabled="{Boolean}false">
  <cq:LiveSyncConfig
      jcr:primaryType="cq:LiveCopy"
      cq:master="/content/blueprint/page"
      cq:rolloutConfigs="[/libs/msm/wcm/rolloutconfigs/default]"/>
</jcr:content>
```

---

## 📤 Rollout Configurations

Define WHAT gets rolled out (which properties/nodes to sync):

| Config | Description |
|--------|------------|
| `default` | All content properties and child nodes |
| `contentUpdate` | Content properties only |
| `push` | Manual push only (no auto-sync) |
| Custom | Your own rollout config with specific exclusions |

### Custom Rollout Config via OSGi
```java
@Component(service = RolloutConfig.class)
@Designate(ocd = CustomRolloutConfig.class)
public class CustomRolloutConfigImpl implements RolloutConfig {
    // Define what to include/exclude in rollout
}
```

---

## 🌍 Translation in AEM

AEM integrates with Translation Management Systems (TMS) for multilingual content.

### Translation Workflow
```
Blueprint (en-US) → Translation Connector → TMS (e.g., Translations.com)
                                                ↓
                    Language Copy ← Translated content returned
```

### Language Copy Structure
```
/content/mysite/
  ├── us/
  │     └── en/       ← Language Master (Blueprint for translation)
  ├── de/
  │     └── de/       ← German translation
  ├── fr/
  │     └── fr/       ← French translation
  └── ja/
        └── ja/       ← Japanese translation
```

### Language Root Configuration
```xml
<!-- /content/mysite/us/en/jcr:content -->
<jcr:content
    jcr:language="en"
    jcr:mixinTypes="[mix:language]"/>
```

---

## 🔑 MSM + Translation Combined Pattern

Large enterprise sites combine both:
1. **MSM** manages structural copies between regions (US, UK, AU)
2. **Translation** manages language variants (en, de, fr) within a region

```
Blueprint: /content/mysite/en (English master)
    ↓ MSM rollout (structure)
Live Copies: /content/mysite/de, /content/mysite/fr
    ↓ Translation workflow
Translated: /content/mysite/de (German content filled in)
```

---

## ❓ Interview Questions & Answers

**Q1. What is MSM in AEM and what problem does it solve?**
> MSM (Multi Site Manager) solves the challenge of managing content across multiple sites/regions without duplicating effort. A Blueprint is the master source; Live Copies inherit its structure and content. When the Blueprint is updated, changes roll out to all Live Copies — reducing manual effort and ensuring consistency.

**Q2. What is the difference between Blueprint and Live Copy?**
> - **Blueprint**: The master/source site. Content authors make changes here.
> - **Live Copy**: A linked copy of a Blueprint. Inherits Blueprint content. Can have local overrides. Receives updates via rollout.

**Q3. What is rollout and how is it triggered?**
> Rollout is the process of pushing Blueprint changes to Live Copies. Triggered:
> - Manually: Sites Console → Select Blueprint → Rollout
> - Automatically: On page activation (if rollout config includes auto-trigger)
> - Programmatically: Via `RolloutManager.rollout()` API

**Q4. How do you break inheritance for a specific component?**
> In the Author UI, select the component in the Live Copy page → Click the "Cancel Inheritance" (chain link) icon. In JCR, this adds `cq:PropertyLiveSyncCancelled` or sets `cq:isSyncDisabled=true` on the component node, preventing further rollout for that component.

**Q5. What is the difference between Language Copy and Live Copy?**
> - **Language Copy**: A copy of content translated into another language. NOT automatically synced with source after initial copy (translations may diverge). Created via Sites Console → Language Copy.
> - **Live Copy**: A copy synced with its Blueprint via rollout. Structural inheritance maintained. Used for regional variants of the same language.

**Q6. How does translation work in AEM?**
> 1. Create a **Translation Integration Framework** (TIF) and connect to a TMS
> 2. Create a **Translation Project** from the Sites Console (select pages → Create Translation Project)
> 3. AEM extracts translatable content into Translation Objects (XLIFF)
> 4. Sends to TMS connector
> 5. Translated content returned → imported back to language copy pages
> 6. Review and approve → publish

**Q7. What is `jcr:language` property used for?**
> Marks a content node as a language root. The `mix:language` mixin enables this property. AEM uses it to identify the language of a page/asset and for language copy operations. Value should be BCP-47 language tag (e.g., `en`, `de`, `fr-FR`).

---

## ✅ Best Practices

- Always define a clear **Blueprint** site — never use a Live Copy as Blueprint
- Use **Rollout Configs** to exclude region-specific fields (don't overwrite local content)
- Add `cq:isSyncDisabled=true` carefully — it breaks the MSM relationship permanently for that node
- Use **Translation Projects** for organized translation workflows with review steps
- Keep language masters (`/content/<site>/en`) as the single source of truth
- Test rollout in **staging** before rolling out in production

---

## 🛠️ Hands-on Tasks

**Day 24**: Create a Blueprint page, create 2 Live Copies, rollout a change
**Day 25**: Create a Language Copy from English to German, simulate translation workflow
