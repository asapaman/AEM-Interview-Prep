# Day 24–25 — Multi Site Manager (MSM) & Translation
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What is MSM (Multi Site Manager)?

**MSM** is AEM's mechanism for managing multiple related websites that share content, structure, and/or components — while allowing controlled variation per site.

### Real-World Use Cases

1. **Multi-regional sites:** `mysite.com/us`, `mysite.com/uk`, `mysite.com/in` — same layout, different regional content
2. **Brand variants:** Main brand + sub-brand sharing common components but with different styling
3. **Franchise model:** Corporate template pushed to regional franchise sites; each region can add local content

### Core Concepts

```
BLUEPRINT (Master)           LIVE COPY
/content/mysite/us/     →   /content/mysite/uk/
  (Source of truth)            (Inherits from Blueprint)
```

| Term | Meaning |
|------|---------|
| **Blueprint** | The master/source site. All changes originate here. |
| **Live Copy** | A site that inherits from the Blueprint. Can inherit everything or selectively override. |
| **Rollout** | Pushing changes FROM Blueprint TO Live Copies |
| **Rollout Config** | Rules that define WHAT is rolled out and WHEN |
| **Inheritance** | Live Copy content linked to Blueprint — shows a lock icon in Author |
| **Broken Inheritance** | Author explicitly overrides a piece of content, breaking the link |

---

## 🔗 Blueprint → Live Copy Relationship

```
Blueprint: /content/mysite/us/home
  ├── jcr:content/
  │     ├── jcr:title = "Welcome to My Site USA"
  │     └── root/
  │           ├── hero/
  │           │     ├── jcr:title = "Discover America"
  │           │     └── sling:resourceType = "mysite/components/content/hero"
  │           └── text/
  │                 └── text = "We serve customers in the US..."

Live Copy: /content/mysite/uk/home
  ├── jcr:content/
  │     ├── cq:LiveSyncConfig  ← Link to blueprint (auto-created by MSM)
  │     ├── jcr:title = "Welcome to My Site UK" ← OVERRIDDEN by UK author
  │     └── root/
  │           ├── hero/
  │           │     ├── jcr:title = "Discover the UK"       ← UK OVERRIDE
  │           │     └── sling:resourceType = "mysite/components/content/hero"  ← INHERITED
  │           └── text/
  │                 └── text = "We serve customers in the UK..."  ← UK OVERRIDE
```

---

## ⚙️ Rollout Configurations

A **Rollout Config** defines what gets synchronized and when. AEM has several built-in rollout configs:

| Rollout Config | Behavior |
|----------------|---------|
| `Standard rollout config` | Pushes all content changes from Blueprint to Live Copies |
| `Push on Modify` | Rolls out whenever Blueprint content is modified |
| `Rollout page on activation` | Rolls out when Blueprint page is published/activated |
| `Page move rollout config` | Handles page moves in Blueprint |
| `Exclude paragraph` | Rolls out everything EXCEPT the main paragraph (authors manage their own) |

### Creating Custom Rollout Config
```java
// Extend LiveRelationshipManager to create custom rollout behavior
// Available at: Tools → Sites → Rollout Configurations
```

---

## 🌐 Setting Up MSM — Step by Step

### Step 1: Configure the Blueprint
1. Go to `Tools → Sites → Blueprints`
2. Click "Create" → Select the source page/section → Name it "My Site US Blueprint"
3. The page becomes the "master" for all live copies

### Step 2: Create a Live Copy
1. Right-click the Blueprint root page in Sites console
2. Select "Create → Live Copy"
3. Specify the target path: `/content/mysite/uk`
4. Select rollout configuration: "Standard rollout config"
5. AEM copies the content structure and creates `cq:LiveSyncConfig` on each page

### Step 3: Rollout Changes
1. Author makes changes on Blueprint (US site)
2. In Sites console → right-click Blueprint page → "Rollout"
3. Select which Live Copies to push to
4. Changes propagate — respecting any broken inheritances

---

## 🔓 Inheritance — Locking & Unlocking

In the Page Editor on a Live Copy page:
- **Locked component (blue padlock):** Inherits from Blueprint — author cannot edit
- **Unlocked component (open padlock):** Inheritance broken — author can freely edit

```
Author clicks padlock → Unlocks (breaks) inheritance for that component
                     → From now on, Blueprint rollouts WON'T overwrite this component
                     → Author can now edit freely
Author re-locks → Restores inheritance → Blueprint will overwrite on next rollout
```

---

## 🌍 AEM Translation Framework

### Translation Workflow Overview

```
AEM Author Creates/Updates Content
            ↓
Content sent to Translation Service
(machine translation or human translators)
            ↓
Translated content returned as XLIFF/XML
            ↓
AEM imports translations back
            ↓
Language copies created/updated under /content/mysite/{lang}/
```

### Language Tree Structure

```
/content/mysite/
  ├── en/                      ← Language master (source)
  │     ├── home/
  │     ├── about/
  │     └── products/
  ├── de/                      ← German translation
  │     ├── home/
  │     └── about/
  ├── fr/                      ← French translation
  └── ja/                      ← Japanese translation
```

### Creating a Language Copy

**Via AEM Sites console:**
1. Select pages to translate from `/content/mysite/en/`
2. `References → Language Copies → Create & Translate`
3. Select target languages (de, fr, ja)
4. Select Translation Method: "Machine Translation" or "Human Translation"
5. Select Translation Project: create new or add to existing
6. Submit → AEM creates translation jobs in `/content/projects/`

---

## 📦 Translation Project

AEM stores translation work in a **Translation Project** (`/content/projects/<project-name>/`):

```
/content/projects/mysite-de-translation/
  ├── dashboard/           ← Project UI config
  ├── jobs/
  │     └── translation-job-001/
  │           ├── payload → [references to en content]
  │           └── status = "InProgress"
  └── members/             ← Project team
```

**Translation Job States:**
```
CREATED → IN_PROGRESS → TRANSLATED → IN_REVIEW → APPROVED → PUBLISHED
```

---

## 🔧 Reading Locale in a Sling Model

```java
import com.day.cq.wcm.api.LanguageManager;
import java.util.Locale;

@Model(
    adaptables = SlingHttpServletRequest.class,
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class InternationalizedModel {

    @ScriptVariable
    private Page currentPage;

    @SlingObject
    private ResourceResolver resourceResolver;

    @OSGiService
    private LanguageManager languageManager;

    private Locale locale;
    private String localizedDateFormat;
    private String currencySymbol;

    @PostConstruct
    protected void init() {
        // Get the locale from the page's language root
        if (currentPage != null && languageManager != null) {
            Page languageRoot = languageManager.getLanguageRoot(currentPage.adaptTo(Resource.class));
            if (languageRoot != null) {
                locale = languageRoot.getLanguage();
            }
        }

        // Fallback locale
        if (locale == null) {
            locale = Locale.ENGLISH;
        }

        // Use locale for formatting
        localizedDateFormat = getDateFormatForLocale(locale);
        currencySymbol = getCurrencySymbol(locale);
    }

    private String getDateFormatForLocale(Locale locale) {
        String lang = locale.getLanguage();
        switch (lang) {
            case "de": return "dd.MM.yyyy";
            case "ja": return "yyyy年MM月dd日";
            case "en":
            default: return "MM/dd/yyyy";
        }
    }

    private String getCurrencySymbol(Locale locale) {
        return java.util.Currency.getInstance(locale).getSymbol(locale);
    }

    public Locale getLocale()               { return locale; }
    public String getLocalizedDateFormat()  { return localizedDateFormat; }
    public String getCurrencySymbol()       { return currencySymbol; }
    public String getLanguage()            { return locale.getLanguage(); }
}
```

---

## ☁️ MSM & Translation in AEM Cloud

| Aspect | AEM 6.5 | AEM Cloud Service |
|--------|---------|-----------------|
| **MSM** | Same functionality | Same, with improved UI |
| **Translation Connectors** | Configure via OSGi | Same |
| **Language Copies** | Same | Same |
| **Translation Memory** | Not built-in | Adobe Translation Memory (new) |
| **Auto-translation** | Google Translate via connector | Same, plus Adobe AI connectors |

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is MSM and when would you use it?**

> **Answer:** MSM (Multi Site Manager) is AEM's mechanism for managing multiple related sites that share content from a common source (Blueprint). It's used when you have a primary/master site and need regional/brand variants that inherit the structure and content but allow controlled local overrides. Use cases: US/UK/India regional sites sharing global content, corporate-to-franchise rollouts, brand-to-sub-brand relationships.

**Q2. What is the difference between a Blueprint and a Live Copy?**

> **Answer:** A Blueprint is the master/source — authors make canonical changes here. A Live Copy inherits from the Blueprint — it receives content via rollout but can override individual components. When inheritance is "locked" on a Live Copy component, Blueprint changes will overwrite it on rollout. When inheritance is "broken" (unlocked), the Live Copy component becomes independent and won't be overwritten.

**Q3. What is a Rollout and when does it happen?**

> **Answer:** A Rollout pushes content changes from a Blueprint to its Live Copies. It can be triggered manually (Sites console → right-click Blueprint page → Rollout), automatically on activation (if "Rollout page on activation" config is used), or programmatically via the `RolloutManager` API. During rollout, MSM respects broken inheritances — it only updates components where inheritance is still intact.

**Q4. What is a Language Copy vs a Live Copy?**

> **Answer:** Both create copies of content, but for different purposes:
> - **Live Copy:** For MULTI-SITE management — same language, different regional content. Blueprint → rollout push. Managed by MSM.
> - **Language Copy:** For TRANSLATION — same content, different language. Source language → translated to target language. Managed by AEM Translation Framework.
> They can be combined: a German site (`/content/mysite/de`) might be both a language copy (translated from EN) AND a live copy (inheriting structure from the master site).

**Q5. How would you programmatically check if a page is a Live Copy?**

> **Answer:**
```java
@Reference
private LiveRelationshipManager liveRelationshipManager;

public boolean isLiveCopy(Resource pageResource) {
    try {
        LiveRelationship relationship = liveRelationshipManager.getLiveRelationship(
            pageResource, false
        );
        return relationship != null;
    } catch (WCMException e) {
        LOG.error("Error checking live relationship", e);
        return false;
    }
}
```

**Q6. What is `LanguageManager` used for?**

> **Answer:** `LanguageManager` is an AEM OSGi service that finds the **language root** of a page — the ancestor page that defines the locale (e.g., the `/en` or `/de` folder). From the language root, you can get the `Locale` (language, country, region). This is how components determine which language to display dates, currencies, and locale-specific content in. You inject it with `@OSGiService` and call `languageManager.getLanguageRoot(resource)`.

---

## ✅ Best Practices

1. **Minimize broken inheritances** — the more overrides in Live Copies, the harder to maintain
2. **Use "Shallow rollout"** when you want structural updates but not content overwrites
3. **Structure language tree correctly** — `/content/<site>/en/`, `/content/<site>/de/` (ISO language codes)
4. **Use `LanguageManager`** in Sling Models to determine locale — don't parse page path strings
5. **Test rollout before production** — rollouts can overwrite author work if inheritances aren't managed
6. **Export MSM configs as packages** — blueprint configurations should be version-controlled

---

## 🛠️ Hands-on Practice

**Day 24 — MSM:**
1. Create a Blueprint at `/content/mysite/us/`
2. Create a Live Copy at `/content/mysite/uk/` with Standard rollout config
3. Modify Blueprint content and observe rollout behavior
4. Break inheritance on one component in the UK site and verify rollout doesn't overwrite it

**Day 25 — Translation:**
1. Create an English language root at `/content/mysite/en/`
2. Use Sites console to create a German language copy (`de`)
3. Create a Translation Project and send 2 pages to "translation" (use Mock translation service)
4. Approve translations and verify German pages are created
5. Write a Sling Model that reads `LanguageManager` to get the locale and format a date
