# Day 23 — Dialog Validation
**Difficulty:** Medium | **4 YOE Focus**

---

## 📖 Topic Explanation

Dialog validation ensures authors enter valid data in component dialogs. AEM supports both client-side (Coral UI/Granite) and server-side validation.

---

## 🎨 Client-Side Validation (Coral UI / Granite)

### Built-in Validation via Dialog XML
```xml
<!-- Required field -->
<title jcr:primaryType="nt:unstructured"
       sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
       fieldLabel="Title"
       name="./title"
       required="{Boolean}true"/>

<!-- Min/Max length -->
<description jcr:primaryType="nt:unstructured"
             sling:resourceType="granite/ui/components/coral/foundation/form/textarea"
             fieldLabel="Description"
             name="./description"
             maxlength="500"/>

<!-- Pattern validation (regex) -->
<email jcr:primaryType="nt:unstructured"
       sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
       fieldLabel="Email"
       name="./email"
       validation="foundation.jcr.name"
       required="{Boolean}true"/>

<!-- Number field with min/max -->
<count jcr:primaryType="nt:unstructured"
       sling:resourceType="granite/ui/components/coral/foundation/form/numberfield"
       fieldLabel="Count"
       name="./count"
       min="{Long}1"
       max="{Long}100"/>
```

---

## 🛠️ Custom JavaScript Validation

```javascript
// /apps/mysite/components/content/mycomponent/clientlibs/editor/js/validation.js
(function($, $document) {
    "use strict";

    // Register custom validator for a field
    $(document).on("foundation-validation-helper-show-error", function(e) {
        // Custom error handling
    });

    // Validator registered as a Foundation validator
    $.validator.register({
        selector: "[data-validation='mysite.url']",
        validate: function(el) {
            var value = el.val();
            if (!value) return; // Required check handled separately

            var urlPattern = /^(https?:\/\/)?([\da-z.-]+)\.([a-z.]{2,6})([/\w .-]*)*\/?$/;
            if (!urlPattern.test(value)) {
                return Granite.I18n.get("Please enter a valid URL");
            }
        }
    });

}(jQuery, jQuery(document)));
```

### Use custom validator in dialog:
```xml
<link jcr:primaryType="nt:unstructured"
      sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
      fieldLabel="URL"
      name="./link"
      validation="mysite.url"/>
```

---

## 🔧 Clientlib for Editor (Author-only)

```xml
<!-- clientlibs/editor/.content.xml -->
<jcr:root xmlns:cq="http://www.day.com/jcr/cq/1.0"
          jcr:primaryType="cq:ClientLibraryFolder"
          categories="[cq.authoring.dialog]"/>
```
> `cq.authoring.dialog` category is loaded ONLY in Author Edit mode dialogs.

---

## 📋 Server-Side Validation (POST-save)

For critical validation, do it in a Sling Model or Servlet:
```java
@PostConstruct
protected void init() {
    if (email != null && !email.matches("^[A-Za-z0-9+_.-]+@(.+)$")) {
        LOG.warn("Invalid email stored in component: {}", resource.getPath());
        email = null; // Graceful degradation
    }
}
```

---

## ❓ Interview Questions & Answers

**Q1. How do you make a dialog field required?**
> Add `required="{Boolean}true"` to the field definition in dialog XML. Coral UI automatically prevents dialog submission if the field is empty, showing a validation error.

**Q2. How do you write custom dialog validation?**
> 1. Create a clientlib with category `cq.authoring.dialog`
> 2. Register a custom validator using `$.validator.register()` (Foundation UI validator API)
> 3. Apply it to the field via `validation="your.custom.validator"` attribute in dialog XML

**Q3. What is the `validation` property in Granite UI fields?**
> It references a Foundation validator by name. AEM ships with built-in validators like `foundation.jcr.name` (valid JCR name). Custom validators can be registered in JavaScript and referenced here.

**Q4. What is the difference between Coral UI and Granite UI?**
> - **Coral UI**: Adobe's HTML/CSS/JS design system (visual component library — buttons, inputs, dialogs)
> - **Granite UI**: AEM's server-side rendering layer for Touch UI forms/dialogs. It uses Coral UI components but renders them server-side via Sling scripts. Dialog XML uses `granite/ui/components/...` resource types.

---

## ✅ Best Practices

- Use built-in Granite validation (required, maxlength) before writing custom JS
- Keep validation JS in a separate clientlib with `cq.authoring.dialog` category (author-only)
- Always do server-side validation too — client-side can be bypassed
- Provide clear, localized error messages using `Granite.I18n.get()`

---

## 🛠️ Hands-on Task

Create a dialog with:
1. Required title field (built-in validation)
2. Email field with custom URL/email regex validation via `$.validator.register()`
3. Number field limited to 1–10
