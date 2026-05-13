# Day 23 — Dialog Validation & Advanced Dialog Patterns
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 Why Dialog Validation Matters

AEM dialogs are the forms authors use to configure components. Without proper validation:
- Authors can save empty required fields → blank component on live site
- Invalid URLs can break page rendering
- Malformed data can crash Sling Models
- Inconsistent input can cause unexpected behavior

Good validation improves author experience and prevents content quality issues.

---

## ✅ Server-Side Validation (Sling Model Level)

The first line of defense: gracefully handle bad data in your Sling Model.

```java
@Model(adaptables = Resource.class, defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class HeroModel {

    @ValueMapValue
    private String title;

    @ValueMapValue
    private String ctaLink;

    @ValueMapValue
    private String ctaLabel;

    private String resolvedCtaLink;
    private boolean isValid;
    private boolean isExternalLink;

    @PostConstruct
    protected void init() {
        // Validate and sanitize title
        if (StringUtils.isBlank(title)) {
            // Component renders in a degraded state, not a crash
            isValid = false;
            return;
        }

        // Title is present — proceed
        isValid = true;

        // Handle CTA link
        if (StringUtils.isNotBlank(ctaLink)) {
            isExternalLink = ctaLink.startsWith("http://")
                          || ctaLink.startsWith("https://")
                          || ctaLink.startsWith("//");

            if (!isExternalLink && ctaLink.startsWith("/")) {
                // Internal path: add .html
                resolvedCtaLink = ctaLink + ".html";
            } else {
                resolvedCtaLink = ctaLink;
            }
        }
    }

    public String getTitle()          { return title; }
    public String getResolvedCtaLink(){ return resolvedCtaLink; }
    public String getCtaLabel()       { return ctaLabel; }
    public boolean isValid()          { return isValid; }
    public boolean isExternalLink()   { return isExternalLink; }
    public boolean hasCta()           { return StringUtils.isNotBlank(resolvedCtaLink)
                                            && StringUtils.isNotBlank(ctaLabel); }
}
```

---

## 🖥️ Built-in Granite UI Validation (Required Fields & Patterns)

AEM's Granite UI provides built-in validators — no JavaScript needed for common cases.

### Required Field
```xml
<title
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
    fieldLabel="Title"
    name="./title"
    required="{Boolean}true"
    fieldDescription="Required: Enter the component title"/>
```

### Min/Max Length
```xml
<title
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
    fieldLabel="Title"
    name="./title"
    maxlength="{Long}100"
    minlength="{Long}3"/>
```

### Pattern Validation (Regex)
```xml
<productCode
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
    fieldLabel="Product Code"
    name="./productCode"
    validation="foundation.jcr.name"
    fieldDescription="Format: ABC-12345"
    pattern="^[A-Z]{3}-[0-9]{5}$"/>
```

### Number Field with Range
```xml
<columnCount
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/numberfield"
    fieldLabel="Number of Columns"
    name="./columnCount"
    min="{Long}1"
    max="{Long}6"
    value="{Long}3"/>
```

---

## 🎯 Custom JavaScript Validation — Coral UI

For more complex validation (cross-field dependencies, async checks), you need custom JavaScript.

### Client Library Setup

```xml
<!-- /apps/mysite/components/content/hero/_cq_dialog/.content.xml -->
<!-- Add clientlibs reference to the dialog -->
<jcr:root
    xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    jcr:primaryType="nt:unstructured"
    jcr:title="Hero Banner"
    sling:resourceType="cq/gui/components/authoring/dialog"
    extraClientlibs="[mysite.dialog.hero]"/>
```

```xml
<!-- /apps/mysite/components/content/hero/dialog-clientlib/.content.xml -->
<jcr:root xmlns:cq="http://www.day.com/jcr/cq/1.0"
          xmlns:jcr="http://www.jcp.org/jcr/1.0"
          jcr:primaryType="cq:ClientLibraryFolder"
          categories="[mysite.dialog.hero]"
          allowProxy="{Boolean}true"/>
```

### Custom Validation JavaScript

```javascript
// /apps/mysite/components/content/hero/dialog-clientlib/js/validation.js

(function($, $document) {
    'use strict';

    /**
     * Wait for dialog to fully open before attaching events.
     * "dialog-ready" fires when the dialog DOM is ready.
     */
    $document.on('dialog-ready', function() {

        // ─── Example 1: CTA Label required when CTA Link is filled ───

        var $ctaLink  = $('[name="./ctaLink"]');
        var $ctaLabel = $('[name="./ctaLabel"]');

        function validateCtaPair() {
            var linkValue  = $ctaLink.val().trim();
            var labelValue = $ctaLabel.val().trim();

            if (linkValue && !labelValue) {
                // Show error on CTA Label field
                showError($ctaLabel, 'CTA Label is required when a CTA Link is provided.');
                return false;
            }

            clearError($ctaLabel);
            return true;
        }

        // Validate when either field changes
        $ctaLink.on('change', validateCtaPair);
        $ctaLabel.on('change', validateCtaPair);


        // ─── Example 2: URL format validation ───

        var $linkField = $('[name="./link"]');

        $linkField.on('change', function() {
            var value = $(this).val().trim();

            if (!value) return; // Empty is OK (required validation handles empty)

            var isInternal = value.startsWith('/');
            var isExternal = value.startsWith('http://') || value.startsWith('https://');
            var isAnchor   = value.startsWith('#');

            if (!isInternal && !isExternal && !isAnchor) {
                showError($linkField,
                    'Invalid URL. Must start with /, http://, https://, or #');
            } else {
                clearError($linkField);
            }
        });


        // ─── Example 3: Tab-level validation before dialog submit ───

        var $form = $('.cq-dialog-form');

        $form.on('submit', function(event) {
            var isValid = validateCtaPair();

            if (!isValid) {
                // Prevent dialog from closing
                event.preventDefault();
                event.stopImmediatePropagation();
                // Switch to the tab containing the invalid field
                activateTabContaining($ctaLabel);
            }
        });


        // ─── Helper Functions ───

        /**
         * Shows a Coral-styled validation error on a field.
         * Uses Coral UI's native invalid state.
         */
        function showError($field, message) {
            var $coral = $field.closest('coral-textfield, coral-pathfield, coral-select');
            if ($coral.length) {
                $coral.attr('invalid', '');
                // Add error message label
                var $label = $field.closest('.coral-Form-fieldwrapper');
                $label.find('.coral-Form-errormessage').remove();
                $label.append(
                    '<p class="coral-Form-errormessage">' + Granite.I18n.get(message) + '</p>'
                );
            }
        }

        function clearError($field) {
            var $coral = $field.closest('coral-textfield, coral-pathfield, coral-select');
            $coral.removeAttr('invalid');
            $field.closest('.coral-Form-fieldwrapper')
                  .find('.coral-Form-errormessage').remove();
        }

        function activateTabContaining($field) {
            var $tab = $field.closest('coral-panel');
            if ($tab.length) {
                var panelId = $tab.attr('id');
                $('[panel="' + panelId + '"]').click();
            }
        }
    });

})(Granite.$, $(document));
```

---

## 🔗 Dialog Field Dependencies (Show/Hide Fields)

```javascript
// Show "CTA Label" field only when "CTA Link" has a value
(function($, $document) {
    'use strict';

    $document.on('dialog-ready', function() {

        var $ctaLinkField = $('[name="./ctaLink"]');
        var $ctaLabelWrapper = $('[name="./ctaLabel"]').closest('.coral-Form-fieldwrapper');

        function toggleCtaLabel() {
            var hasLink = $ctaLinkField.val().trim().length > 0;
            if (hasLink) {
                $ctaLabelWrapper.show();
            } else {
                $ctaLabelWrapper.hide();
            }
        }

        // Run on load and on change
        toggleCtaLabel();
        $ctaLinkField.on('change input', toggleCtaLabel);
    });

})(Granite.$, $(document));
```

---

## 📋 Dialog Field Types Quick Reference

| Granite Resource Type | Use Case |
|----------------------|---------|
| `coral/foundation/form/textfield` | Single-line text |
| `coral/foundation/form/textarea` | Multi-line text |
| `coral/foundation/form/pathfield` | JCR path browser (pages, DAM) |
| `coral/foundation/form/numberfield` | Number with min/max |
| `coral/foundation/form/checkbox` | Boolean toggle |
| `coral/foundation/form/select` | Dropdown |
| `coral/foundation/form/radiogroup` | Radio buttons |
| `coral/foundation/form/colorfield` | Color picker |
| `coral/foundation/form/datepicker` | Date/time picker |
| `coral/foundation/form/multifield` | Repeatable group |
| `cq/gui/components/authoring/dialog/richtext` | Rich text editor |
| `coral/foundation/form/tagsautocomplete` | Tag picker |
| `coral/foundation/form/hidden` | Hidden value |

---

## ❓ Interview Questions & Detailed Answers

**Q1. How do you make a dialog field required?**

> **Answer:** Add `required="{Boolean}true"` to the field definition in the dialog XML. AEM's Granite UI handles the validation — it shows a red border and prevents dialog submission if the field is empty. For custom cross-field validation (e.g., "Label required when Link is filled"), you need a client-side JavaScript validator attached via `extraClientlibs` on the dialog.

**Q2. How do you add custom validation to an AEM dialog?**

> **Answer:**
> 1. Add a `clientlib` with your validation JavaScript
> 2. Reference it in the dialog's `.content.xml` via `extraClientlibs="[my.dialog.validator]"`
> 3. In the JS, listen to `$document.on('dialog-ready', ...)` to ensure the DOM is ready
> 4. On field change or form submit, check conditions and call `showError()` / `clearError()` on the Coral elements
> 5. For form submit validation, listen to `$('.cq-dialog-form').on('submit', ...)` and call `event.preventDefault()` to block invalid saves

**Q3. How do you show or hide a dialog field based on another field's value?**

> **Answer:** In your dialog clientlib JavaScript, listen to the `dialog-ready` event, then attach a `change`/`input` listener to the controlling field. Based on its value, use jQuery's `.show()` / `.hide()` on the target field's wrapper element (`closest('.coral-Form-fieldwrapper')`). Run the check once on load for initial state.

**Q4. What is `extraClientlibs` in a dialog?**

> **Answer:** `extraClientlibs` is a property on the dialog node that loads additional client libraries specifically when the dialog is opened. Unlike page clientlibs (loaded when the page loads), these are lazy-loaded only when needed. Use it for: custom validation scripts, dialog-specific UI enhancements, or field dependency logic.

**Q5. What is the `foundation.jcr.name` validation?**

> **Answer:** It's a built-in Granite UI validator that ensures a field value is a valid JCR node name (no special characters that are illegal in JCR). Useful for fields that will be used as JCR node names.

---

## ✅ Best Practices

1. **Required fields first** — use `required="{Boolean}true"` for simple mandatory fields
2. **JavaScript validation for cross-field rules** only — don't over-engineer simple cases
3. **Always validate on the server too** (Sling Model) — JS can be bypassed
4. **Use `dialog-ready` event** — dialog DOM isn't ready in `document.ready`
5. **Show clear error messages** — "Required" is not enough; say why and what format
6. **Use `data-sly-test`** in HTL to handle missing data gracefully (don't let null cause NPE)
7. **Add `fieldDescription`** to complex fields — helps authors understand expected format

---

## 🛠️ Hands-on Practice

1. Create a dialog with: required title (textfield), optional CTA link (pathfield), CTA label (textfield)
2. Add Granite `required="{Boolean}true"` to the title field — verify it blocks saving when empty
3. Write JavaScript validator: show "CTA Label is required when CTA link is set" cross-field error
4. Add a select (dropdown) for "variant" — show/hide an additional field based on selection
5. Test all validations in the Page Editor
