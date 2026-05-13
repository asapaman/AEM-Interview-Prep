# Day 22 — Workflow & Scheduler
**Difficulty:** Hard | **4 YOE Focus**

---

## 📖 Workflows

AEM Workflows automate content processes (review, approval, translation, publication). They consist of **steps** connected in a model.

### Workflow Model Components
| Component | Description |
|-----------|------------|
| **Workflow Model** | The process definition (steps + transitions) |
| **Workflow Launcher** | Triggers a workflow based on JCR events |
| **Workflow Step** | Individual task unit (Process, Participant, Dialog) |
| **Workflow Instance** | A running execution of a workflow model |
| **Work Item** | Task assigned to a participant in the workflow |

### Common Workflow Step Types
- **Process Step**: Automated — runs a Java class implementing `WorkflowProcess`
- **Participant Step**: Manual — assigns task to a user/group to complete
- **Dialog Participant Step**: Collects additional data from user via a dialog
- **OR/AND Split**: Branching and joining parallel paths

---

## 🔧 Custom Workflow Process Step

```java
package com.mysite.core.workflows;

import com.adobe.granite.workflow.WorkflowException;
import com.adobe.granite.workflow.WorkflowSession;
import com.adobe.granite.workflow.exec.WorkItem;
import com.adobe.granite.workflow.exec.WorkflowProcess;
import com.adobe.granite.workflow.metadata.MetaDataMap;
import org.osgi.service.component.annotations.Component;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Component(
    service = WorkflowProcess.class,
    property = {"process.label=My Custom Process Step"}  // Label shown in workflow editor
)
public class MyCustomWorkflowProcess implements WorkflowProcess {

    private static final Logger LOG = LoggerFactory.getLogger(MyCustomWorkflowProcess.class);

    @Override
    public void execute(WorkItem workItem, WorkflowSession workflowSession, MetaDataMap args)
            throws WorkflowException {

        // Get the payload (usually a page path)
        String payloadPath = workItem.getWorkflowData().getPayload().toString();
        LOG.info("Processing workflow for payload: {}", payloadPath);

        // Get step arguments (configured in workflow model)
        String notifyEmail = args.get("PROCESS_ARGS", "default@company.com");

        try {
            // Get ResourceResolver from WorkflowSession
            javax.jcr.Session session = workflowSession.adaptTo(javax.jcr.Session.class);

            // Do your business logic here
            // e.g., update a JCR property, send notification, call API

            // Mark workflow data
            workItem.getWorkflowData().getMetaDataMap().put("processedBy", "MyCustomProcess");

        } catch (Exception e) {
            LOG.error("Error in workflow process step for payload: {}", payloadPath, e);
            throw new WorkflowException("Failed to execute custom step", e);
        }
    }
}
```

---

## 🚀 Workflow Launcher

Launchers automatically trigger workflows based on JCR events:

| Setting | Example Value |
|---------|-------------|
| **Event Type** | Node Created, Node Modified, Node Removed |
| **Path** | `/content/mysite/(.*)` |
| **Node Type** | `cq:Page` |
| **Condition** | `jcr:content/jcr:mimeType == 'image/jpeg'` |
| **Workflow Model** | `/var/workflow/models/dam/update_asset` |

---

## ⏰ Scheduler (OSGi Scheduler)

Schedulers run code on a schedule using cron expressions.

```java
package com.mysite.core.schedulers;

import org.apache.sling.commons.scheduler.ScheduleOptions;
import org.apache.sling.commons.scheduler.Scheduler;
import org.osgi.service.component.annotations.*;
import org.osgi.service.metatype.annotations.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@ObjectClassDefinition(name = "My Site - Report Scheduler Config")
@interface ReportSchedulerConfig {
    @AttributeDefinition(name = "Cron Expression", description = "e.g. 0 0 3 * * ? = daily at 3AM")
    String schedulerExpression() default "0 0 3 * * ?";

    @AttributeDefinition(name = "Scheduler Name")
    String schedulerName() default "ReportScheduler";

    @AttributeDefinition(name = "Enabled")
    boolean enabled() default true;
}

@Component(service = Runnable.class, immediate = true)
@Designate(ocd = ReportSchedulerConfig.class)
public class ReportScheduler implements Runnable {

    private static final Logger LOG = LoggerFactory.getLogger(ReportScheduler.class);

    @Reference
    private Scheduler scheduler;

    private String schedulerName;
    private boolean enabled;

    @Activate
    @Modified
    protected void activate(ReportSchedulerConfig config) {
        this.schedulerName = config.schedulerName();
        this.enabled = config.enabled();

        // Remove any existing scheduled job first
        removeScheduledJob();

        if (enabled) {
            addScheduledJob(config.schedulerExpression());
        }
    }

    @Deactivate
    protected void deactivate() {
        removeScheduledJob();
    }

    private void addScheduledJob(String cronExpression) {
        ScheduleOptions scheduleOptions = scheduler.EXPR(cronExpression);
        scheduleOptions.name(schedulerName);
        scheduleOptions.canRunConcurrently(false);  // IMPORTANT: prevent overlapping runs
        scheduler.schedule(this, scheduleOptions);
        LOG.info("Scheduled job '{}' with cron: {}", schedulerName, cronExpression);
    }

    private void removeScheduledJob() {
        scheduler.unschedule(schedulerName);
    }

    @Override
    public void run() {
        if (!enabled) return;
        LOG.info("ReportScheduler running at: {}", java.time.LocalDateTime.now());

        // IMPORTANT: Get ResourceResolver using service user, NOT from request
        // Use ResourceResolverFactory to get a resource resolver
        try {
            // Your scheduled logic here
            generateReport();
        } catch (Exception e) {
            LOG.error("Error in ReportScheduler", e);
        }
    }

    private void generateReport() {
        // Business logic
        LOG.info("Generating report...");
    }
}
```

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between a Workflow and a Scheduler?**
> - **Workflow**: Content-centric process triggered by events or manually. Has steps (process, participant), tracks state, supports human tasks/approvals. Used for content review, activation, translation.
> - **Scheduler**: Time-based execution of code (cron jobs). No content payload, no manual steps. Used for cleanup tasks, report generation, cache invalidation.

**Q2. What is a Workflow Launcher?**
> A Workflow Launcher is a JCR event listener that automatically starts a workflow when specified conditions are met. Configured in AEM UI (`Tools → Workflow → Launchers`). Key config: event type (Node Created/Modified), path pattern, node type filter, and the workflow model to trigger.

**Q3. How do you avoid memory leaks in a Scheduler?**
> 1. Always close `ResourceResolver` in a `finally` block or use try-with-resources
> 2. Use `canRunConcurrently(false)` to prevent overlapping executions
> 3. Unschedule in `@Deactivate` to prevent ghost jobs after bundle restart
> 4. Never store request-scoped objects as class fields
> 5. Use a separate thread pool if doing heavy work (don't block the scheduler thread)

**Q4. How do you get a ResourceResolver inside a Scheduler?**
> Never use a request's resolver. Use `ResourceResolverFactory` with a service user:
```java
@Reference
private ResourceResolverFactory resolverFactory;

private void doWork() {
    Map<String, Object> params = new HashMap<>();
    params.put(ResourceResolverFactory.SUBSERVICE, "my-service-user");
    try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(params)) {
        // Use resolver safely
        Resource resource = resolver.getResource("/content/mysite");
    } catch (LoginException e) {
        LOG.error("Failed to get ResourceResolver", e);
    }
}
```

**Q5. What are common AEM workflow models?**
> - `DAM Update Asset`: Triggered on asset upload — generates renditions, metadata
> - `Request for Activation`: Sends content for approval before publishing
> - `Request for Deactivation`: Unpublish approval workflow
> - `Page Version Purge`: Cleans up old page versions

**Q6. What is a cron expression? Give an example.**
> Cron expression: `seconds minutes hours day-of-month month day-of-week year`
> - `0 0 3 * * ?` → Every day at 3:00 AM
> - `0 */15 * * * ?` → Every 15 minutes
> - `0 0 9-17 ? * MON-FRI` → Every hour 9AM–5PM on weekdays

---

## ✅ Best Practices

- Always use `canRunConcurrently(false)` unless you explicitly need parallel runs
- Always unschedule in `@Deactivate` to prevent memory leaks
- Use service users for JCR access in schedulers — never use admin session
- Keep scheduler logic in the `run()` method thin — delegate to services
- Add `enabled` flag via OSGi config so schedulers can be disabled per environment
- Log start/end of scheduler execution for monitoring

---

## 🛠️ Hands-on Tasks

1. Create a cron-based scheduler that runs daily at 2AM
2. Build a custom workflow process step that adds a JCR property to the payload page
3. Configure a workflow launcher to trigger on `/content/mysite/en` Node Modified events
