# Day 22 — Workflows & Schedulers
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What is an AEM Workflow?

An **AEM Workflow** is a structured process that automates a series of steps for content lifecycle management. Think of it as a business process engine.

### Real-World Examples
- **Content Review/Approval:** Author creates page → Manager reviews → Legal approves → Content published
- **Asset Processing:** Upload image → DAM workflow → Generate renditions → Extract metadata → Tag with AI
- **Page Expiry:** Scheduled workflow checks page expiry dates → Deactivates expired pages
- **Translation:** Send pages to translation service → Wait for completion → Import translations

### AEM vs Custom Workflow

| | AEM Workflow | Custom Scheduler |
|--|---|---|
| **Triggered by** | Content events (create, modify, delete) or manually | Time-based (cron) |
| **Payload** | Content node (page, asset, etc.) | No payload — runs standalone |
| **State** | Maintains state across steps | Stateless |
| **UI** | Visible in AEM Workflow Console | No UI |
| **Use when** | Content needs structured review/processing | Periodic background tasks |

---

## 🔧 Custom Workflow Process Step

### Step 1: Implement `WorkflowProcess`

```java
package com.mysite.core.workflows;

import com.adobe.granite.workflow.WorkflowSession;
import com.adobe.granite.workflow.exec.WorkItem;
import com.adobe.granite.workflow.exec.WorkflowProcess;
import com.adobe.granite.workflow.metadata.MetaDataMap;
import com.day.cq.wcm.api.Page;
import com.day.cq.wcm.api.PageManager;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.api.resource.ResourceResolver;
import org.osgi.framework.Constants;
import org.osgi.service.component.annotations.Component;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * Custom Workflow Process Step.
 *
 * @Component property "process.label" = the name shown in the Workflow Editor UI
 * This is how AEM identifies and displays your custom step in the workflow designer.
 */
@Component(
    service = WorkflowProcess.class,
    property = {
        Constants.SERVICE_DESCRIPTION + "=My Site: Send Page Notification",
        "process.label=My Site - Send Page Notification"     // ← Shows in Workflow Editor
    }
)
public class PageNotificationWorkflowProcess implements WorkflowProcess {

    private static final Logger LOG = LoggerFactory.getLogger(PageNotificationWorkflowProcess.class);

    @Reference
    private EmailNotificationService emailService;

    /**
     * execute() is called when this step is reached in the workflow.
     *
     * @param workItem   - The current workflow item (contains payload path, metadata)
     * @param workflowSession - The workflow session (advance/back/terminate workflow)
     * @param args       - Arguments configured in the workflow step dialog
     */
    @Override
    public void execute(WorkItem workItem, WorkflowSession workflowSession, MetaDataMap args)
            throws WorkflowException {

        // 1. GET THE PAYLOAD (what content this workflow is processing)
        String payloadPath = workItem.getWorkflowData().getPayload().toString();
        LOG.info("Processing workflow payload: {}", payloadPath);

        // 2. GET STEP ARGUMENTS (configured in Workflow Editor for this step)
        String recipientEmail = args.get("recipientEmail", String.class);
        String emailTemplate  = args.get("emailTemplate", "default-template");

        // 3. GET RESOURCE RESOLVER from workflow session
        ResourceResolver resolver = workflowSession.adaptTo(ResourceResolver.class);
        if (resolver == null) {
            throw new WorkflowException("Could not get ResourceResolver from workflow session");
        }

        // 4. ACCESS THE PAYLOAD CONTENT
        Resource payloadResource = resolver.getResource(payloadPath);
        if (payloadResource == null) {
            LOG.warn("Payload resource not found: {} — skipping workflow step", payloadPath);
            return;  // Don't throw — let workflow continue
        }

        // 5. ADAPT TO PAGE (if the payload is a page)
        PageManager pageManager = resolver.adaptTo(PageManager.class);
        Page page = pageManager != null ? pageManager.getContainingPage(payloadResource) : null;

        if (page != null) {
            String pageTitle = page.getTitle();
            String pageAuthor = workItem.getWorkflow().getInitiator();

            // 6. DO THE WORK
            try {
                emailService.sendNotification(
                    recipientEmail,
                    "Page Ready for Review: " + pageTitle,
                    "Page '" + pageTitle + "' submitted by " + pageAuthor + " is ready for review.",
                    page.getPath()
                );
                LOG.info("Notification sent for page: {}", page.getPath());
            } catch (Exception e) {
                LOG.error("Failed to send notification for page: {}", page.getPath(), e);
                // Don't throw — notification failure shouldn't block content lifecycle
            }
        }

        // WorkflowSession auto-advances to next step after execute() returns
    }
}
```

### Step 2: Register in Workflow Model

1. Go to: `Tools → Workflow → Models`
2. Create/edit a workflow model
3. Drag a **Process Step** into the workflow
4. Edit it → Process tab → select "My Site - Send Page Notification"
5. Add arguments: `recipientEmail=reviews@mysite.com`

### Step 3: Access Workflow Arguments

```java
// Arguments are configured as key=value pairs in the workflow step dialog
// In the "Process" tab of the step, under "Arguments":
// recipientEmail=reviews@mysite.com
// emailTemplate=review-request

// In your code:
String recipientEmail = args.get("recipientEmail", String.class);
// Or with default:
String template = args.get("emailTemplate", "default-template");
```

---

## ⏰ OSGi Schedulers — Complete Guide

A **Scheduler** runs background tasks on a time-based schedule (like a Java cron job), without any user or content trigger.

### When to Use Schedulers
- Nightly report generation
- Hourly cache refresh
- Daily expiry check for pages
- Periodic sync with external systems (CRM, PIM)
- Cleanup of temporary/old JCR content

### Implementation: `Runnable` + Scheduler Service

```java
package com.mysite.core.schedulers;

import com.mysite.core.services.ContentSyncService;
import org.apache.sling.api.resource.*;
import org.apache.sling.commons.scheduler.ScheduleOptions;
import org.apache.sling.commons.scheduler.Scheduler;
import org.osgi.service.component.annotations.*;
import org.osgi.service.metatype.annotations.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Collections;
import java.util.Map;

@Component(immediate = true)
@Designate(ocd = ContentSyncScheduler.Config.class)
public class ContentSyncScheduler implements Runnable {

    private static final Logger LOG = LoggerFactory.getLogger(ContentSyncScheduler.class);

    // OSGi Configuration
    @ObjectClassDefinition(name = "My Site - Content Sync Scheduler")
    public @interface Config {

        @AttributeDefinition(
            name = "Cron Expression",
            description = "When to run. Examples: '0 0 2 * * ?' = 2am daily, " +
                          "'0 0/30 * * * ?' = every 30 min"
        )
        String schedulerExpression() default "0 0 2 * * ?";  // 2 AM daily

        @AttributeDefinition(name = "Enabled")
        boolean enabled() default true;

        @AttributeDefinition(name = "Concurrent Execution")
        boolean schedulerConcurrent() default false;  // Don't allow overlapping runs
    }

    private static final String JOB_NAME = "ContentSyncScheduler";

    @Reference
    private Scheduler scheduler;

    @Reference
    private ResourceResolverFactory resolverFactory;

    @Reference
    private ContentSyncService contentSyncService;

    private boolean enabled;

    @Activate
    protected void activate(Config config) {
        this.enabled = config.enabled();
        if (enabled) {
            registerScheduler(config);
        }
    }

    @Modified
    protected void modified(Config config) {
        removeScheduler();    // Remove old schedule
        activate(config);     // Register with new config
    }

    @Deactivate
    protected void deactivate() {
        removeScheduler();    // ALWAYS unschedule on deactivate!
    }

    private void registerScheduler(Config config) {
        ScheduleOptions options = scheduler.EXPR(config.schedulerExpression());
        options.name(JOB_NAME);
        options.canRunConcurrently(config.schedulerConcurrent());  // false = safe
        scheduler.schedule(this, options);
        LOG.info("ContentSyncScheduler registered with expression: {}",
            config.schedulerExpression());
    }

    private void removeScheduler() {
        scheduler.unschedule(JOB_NAME);
        LOG.info("ContentSyncScheduler unscheduled");
    }

    @Override
    public void run() {
        if (!enabled) {
            LOG.debug("ContentSyncScheduler is disabled — skipping run");
            return;
        }

        LOG.info("ContentSyncScheduler: starting run");
        long startTime = System.currentTimeMillis();

        // ALWAYS use try-with-resources for ResourceResolver
        Map<String, Object> params = Collections.singletonMap(
            ResourceResolverFactory.SUBSERVICE, "content-sync-service"
        );

        try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(params)) {
            // Do the actual work
            int syncCount = contentSyncService.syncContent(resolver);
            LOG.info("ContentSyncScheduler: synced {} items in {}ms",
                syncCount, System.currentTimeMillis() - startTime);
        } catch (LoginException e) {
            LOG.error("ContentSyncScheduler: Cannot get ResourceResolver. "
                + "Check service user 'content-sync-service' configuration", e);
        } catch (Exception e) {
            LOG.error("ContentSyncScheduler: Unexpected error during execution", e);
            // Don't re-throw — scheduler shouldn't crash on business logic errors
        }
    }
}
```

---

## 🕐 Cron Expression Guide

```
Cron format: Seconds Minutes Hours DayOfMonth Month DayOfWeek [Year]

Field         Allowed Values      Special Chars
─────────────────────────────────────────────────
Seconds       0-59               , - * /
Minutes       0-59               , - * /
Hours         0-23               , - * /
Day of Month  1-31               , - * / ? L W
Month         1-12 or JAN-DEC   , - * /
Day of Week   1-7 or SUN-SAT    , - * / ? L #
Year          (optional)

Special Characters:
* = any value          → * in hours = every hour
, = value list         → 1,3,5 in hours = at 1am, 3am, 5am
- = range              → 1-5 in day = Mon to Fri
/ = step               → 0/15 in minutes = every 15 mins from :00
? = no specific value  → use in DayOfMonth OR DayOfWeek (not both)
L = last               → L in DayOfMonth = last day of month
W = nearest weekday    → 15W = nearest weekday to the 15th

Common Cron Expressions:
0 0 2 * * ?          → 2:00 AM every day
0 0 * * * ?          → Every hour (at :00)
0 0/30 * * * ?       → Every 30 minutes
0 0 12 * * MON-FRI   → Noon, Monday to Friday
0 0 8 ? * MON        → 8 AM every Monday
0 0 0 1 * ?          → Midnight, first of every month
0 0 2 ? * SUN        → 2 AM every Sunday
```

---

## 🔄 Workflow vs Scheduler — Decision Matrix

| Scenario | Use |
|----------|-----|
| Send email when page is ready for review | Workflow |
| Sync product prices every night at 2 AM | Scheduler |
| Generate thumbnail renditions on image upload | Workflow (DAM asset workflow) |
| Clean up /var/log/mysite/ every Sunday | Scheduler |
| Translate pages after author submits | Workflow |
| Refresh home page recommendation cache every hour | Scheduler |
| Send newsletter on page publish | Workflow |
| Daily SEO report generation | Scheduler |

---

## 🔍 Workflow Console — Monitoring

```
Tools → Workflow → Instances     → Active workflow instances
Tools → Workflow → Archive       → Completed/terminated workflows
Tools → Workflow → Failures      → Failed workflow instances (CRITICAL to monitor!)
Tools → Workflow → Models        → Create/edit workflow models

If a workflow is STUCK or FAILED:
1. Check Tools → Workflow → Failures → identify failed instance
2. Click instance → see which step failed and why
3. Options:
   a. Retry the failed step (if transient error)
   b. Move to next step (skip the failed step)
   c. Terminate the workflow
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is an AEM Workflow and what are its components?**

> **Answer:** An AEM Workflow is a configurable business process for content management. Components:
> - **Workflow Model:** The blueprint — defines steps, conditions, and routing. Created in Workflow Editor (`Tools → Workflow → Models`).
> - **Workflow Instance:** A running execution of a model with a specific payload.
> - **Payload:** The content being processed (a page, asset, or JCR path).
> - **WorkItem:** Represents the current step in a running workflow.
> - **Workflow Process:** A Java class implementing `WorkflowProcess` — the custom step logic.
> - **Participant Step:** Assigns a task to a user/group (for human approval steps).

**Q2. How do you create a custom workflow process step?**

> **Answer:** Implement the `WorkflowProcess` interface with an `execute(WorkItem, WorkflowSession, MetaDataMap)` method. Register with `@Component(service = WorkflowProcess.class)` and include the `process.label` property — this label appears in the Workflow Editor UI when template authors configure the step. In `execute()`: get the payload path from `workItem.getWorkflowData().getPayload()`, adapt the ResourceResolver from `workflowSession`, access the content, perform your business logic.

**Q3. How is a Scheduler different from a Workflow?**

> **Answer:** A Scheduler runs Java code on a time-based cron schedule (every night, every hour) without any content-specific trigger. A Workflow runs in response to a content event (page created, asset uploaded, user triggers it) and processes a specific content payload through a series of defined steps. Use Schedulers for time-based background jobs (nightly cache refresh, daily reports). Use Workflows for content lifecycle processes (review, approval, translation).

**Q4. How do you write a production-safe Scheduler?**

> **Answer:** Key practices:
> 1. Implement `Runnable` and use `@Component(immediate = true)` with `@Designate` for config
> 2. Register in `@Activate`, unregister in `@Deactivate` — prevents orphan schedulers
> 3. Re-register in `@Modified` after config change
> 4. Use `canRunConcurrently(false)` — prevents pile-up if a run takes longer than the interval
> 5. Use try-with-resources for `ResourceResolver`
> 6. Never store ResourceResolver as a field
> 7. Catch ALL exceptions in `run()` — an uncaught exception kills the scheduler thread

**Q5. What does `canRunConcurrently(false)` do and why is it important?**

> **Answer:** It prevents multiple simultaneous runs of the same scheduler. If a scheduled job runs every 30 minutes but takes 45 minutes to complete, without `canRunConcurrently(false)`, a SECOND instance starts while the first is still running. This leads to: double processing, race conditions on JCR writes, and resource exhaustion. With `false`, AEM skips the scheduled run if a previous instance is still executing.

**Q6. How do you debug a workflow that got stuck or failed?**

> **Answer:**
> 1. Go to `Tools → Workflow → Failures` — find the failed instance
> 2. Click it to see the failed step and exception message
> 3. Check `error.log` at the time of failure for stack traces
> 4. Options: Retry the step, advance to next step, or terminate
> 5. Fix the bug in your WorkflowProcess, deploy the fix
> 6. Retry any queued failed instances

---

## ✅ Best Practices

1. **Always use `canRunConcurrently(false)`** in schedulers — prevents processing pile-ups
2. **Always unschedule in `@Deactivate`** — prevents orphan threads and memory leaks
3. **Catch all exceptions in `run()`** — uncaught exceptions terminate the scheduler
4. **Use try-with-resources** for ResourceResolver in schedulers
5. **Make workflow steps idempotent** — safe to retry if they fail
6. **Don't throw from workflow execute()** for non-critical errors — log and continue
7. **Monitor `Tools → Workflow → Failures`** in production regularly
8. **Keep workflow models in version control** — export as packages

---

## 🛠️ Hands-on Practice

**Workflow:**
1. Create a workflow model with 2 custom steps: "Validate Content" + "Send Notification"
2. Implement `ValidateContentProcess` that checks if a page has a title and meta description
3. Implement `NotificationProcess` that logs the page path and author name
4. Test by manually triggering the workflow on a page

**Scheduler:**
1. Create a `PageExpiryScheduler` that runs at 2 AM daily
2. It queries pages under `/content/mysite` with `expiryDate` property set in the past
3. Logs "Page expired: {path}" for each matching page
4. Add `enabled=false` default in config — enable via OSGi config manager
