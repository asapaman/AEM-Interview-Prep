# Day 37 — Mock Interview — 75 Rapid-Fire Q&A (Project-Based)
**Target:** 2–4 YOE | Last-Day Revision Before Your Interview

---

> 📌 **How to use this file:** Cover the answer and try to answer each question out loud. The answers have been rewritten to include **Project Examples** so you can explain concepts using real-world scenarios during your interview.

---

## 🏛️ SECTION 1: Architecture & Fundamentals (15 Questions)

**Q1. What are the main layers of AEM's architecture?**
> **Answer:** AEM is built on four core layers: JCR (Oak) for data storage, Sling for RESTful web framework and resource resolution, OSGi for the Java plugin and service architecture, and AEM WCM (Web Content Management) for the authoring UI. Dispatcher sits in front for caching.
> **Project Example:** Imagine a Retail site. The **JCR** stores the "Summer Sale" banner text. **Sling** maps the URL `/content/retail/summer-sale` to that banner node. **OSGi** runs a Java service fetching live product prices. **Dispatcher** caches the final HTML so thousands of users can view the sale instantly without crashing the server.

**Q2. What is the role of Apache Sling in AEM?**
> **Answer:** Sling is a RESTful framework that maps HTTP URLs directly to JCR content nodes using `sling:resourceType`, then resolves which script (HTL) or servlet should render that content. 
> **Project Example:** When a user visits `www.bank.com/credit-cards.html`, Sling looks up the node at `/content/bank/credit-cards`. It sees `sling:resourceType="bank/components/page/creditCardPage"` and automatically runs the HTL script found at `/apps/bank/components/page/creditCardPage/creditCardPage.html`.

**Q3. What is the difference between Author and Publish instances?**
> **Answer:** Author is the internal environment where content creators log in, build pages, and draft content. It is un-cached and secure. Publish is the public-facing environment that serves the final, approved web pages to end-users. Content is pushed from Author to Publish via replication.
> **Project Example:** The marketing team logs into the **Author** instance to create a new "Black Friday" landing page. Once approved, they hit "Publish". AEM replicates the page to the **Publish** instances, making it live for internet shoppers.

**Q4. What does the Dispatcher do?**
> **Answer:** Dispatcher is AEM's caching and security module for the Apache HTTP Web Server. Its three main roles are Caching (storing rendered HTML/JSON as flat files), Load Balancing (distributing traffic across multiple Publish instances), and Security (blocking malicious URLs).
> **Project Example:** If 10,000 users request your homepage, the **Dispatcher** serves the cached `home.html` file from its hard drive 9,999 times. The AEM Publish server only had to render the page once, saving massive CPU power.

**Q5. What is a JCR node type?**
> **Answer:** A node type defines the structure constraints of a node—which properties it must have, and what child nodes are allowed. `cq:Page` must have a `jcr:content` child. `nt:unstructured` allows any property.
> **Project Example:** When creating a user profile in the JCR, you might use `nt:unstructured` so you can freely add custom properties like `phoneNumber` or `loyaltyPoints` without strict schema constraints.

**Q6. What is `sling:resourceType`?**
> **Answer:** A critical property on a JCR node that tells Sling which component should be used to render it. It points to a path under `/apps/`.
> **Project Example:** If an author drags a "Hero Image" component onto a page, AEM creates a node with `sling:resourceType="myproject/components/hero"`. Sling reads this and executes the HTL script at that path.

**Q7. What is `sling:resourceSuperType`?**
> **Answer:** It defines inheritance. If a component cannot find a specific script, Sling falls back to the `sling:resourceSuperType` path.
> **Project Example:** You want an Accordion component, but you want it to look slightly different than the Core Component. You set your component's `sling:resourceSuperType="core/wcm/components/accordion/v1/accordion"`. You only write the CSS; the Java and dialog logic are inherited.

**Q8. What is the difference between run modes?**
> **Answer:** Run modes configure AEM for specific environments (`dev`, `qa`, `prod`) and roles (`author`, `publish`). This allows OSGi configurations to change automatically based on where AEM is running.
> **Project Example:** Your Payment Gateway API URL needs to be the sandbox URL in QA, but the real URL in Prod. You create `config.author.qa` and `config.author.prod` OSGi config files. AEM automatically picks the right one.

**Q9. Explain the AEM request flow end-to-end.**
> **Answer:** Browser → CDN → Dispatcher → AEM Publish → Sling URL decomposition → Node resolution → Sling Model execution → HTL rendering → Response.
> **Project Example:** User clicks an article link. CDN misses. Dispatcher misses. The request hits AEM Publish. Sling breaks down the URL, finds the Article Page node. The Article Sling Model runs Java code to fetch the author's name. HTL renders the HTML. Dispatcher caches the HTML and sends it back to the user.

**Q10. What is the difference between `/apps` and `/libs`?**
> **Answer:** `/libs` is for Adobe's out-of-the-box system code. `/apps` is for your custom project code. Never modify `/libs`. If you create a path in `/apps` that matches `/libs`, Sling uses `/apps` first (Overlay).
> **Project Example:** You want to change the default AEM login screen. You don't edit `/libs/granite/core/content/login`. Instead, you copy it to `/apps/granite/core/content/login` and edit it there. AEM safely uses your version.

**Q11. What is Dispatcher cache invalidation?**
> **Answer:** When an author publishes a page, AEM sends a "flush" request to Dispatcher. Dispatcher updates a `.stat` file. The next time a user requests the page, Dispatcher sees the `.stat` file is newer than the cached HTML, deletes the old HTML, and asks AEM for a fresh copy.
> **Project Example:** You fix a typo on the Homepage. When you publish it, AEM tells Dispatcher to invalidate the cache. The next visitor gets the fixed typo, and Dispatcher saves that new version for future visitors.

**Q12. What is `statfileslevel` in Dispatcher?**
> **Answer:** It controls the granularity of cache invalidation. A level of 0 means publishing ONE page invalidates the ENTIRE site cache. A level of 4 means publishing a page only invalidates caches within its specific sub-folder.
> **Project Example:** For a global site (US, UK, FR), you set `statfileslevel=3`. Publishing an article in the `/us/en/` branch only flushes the US cache. The UK and FR caches remain completely unaffected.

**Q13. What is a Closed User Group (CUG)?**
> **Answer:** A security feature that restricts access to published pages to specific logged-in user groups. Unauthenticated users are redirected to a login page.
> **Project Example:** You have an "Employee Portal" section on your public website. You apply a CUG to the `/content/site/employee` folder. Now, only users belonging to the "Employees" group can view those pages.

**Q14. What is `/etc/map` in Sling?**
> **Answer:** It maps internal JCR repository paths to clean, public-facing URLs, and vice versa.
> **Project Example:** Your actual page is at `http://localhost:4502/content/mysite/us/en/home.html`. You configure `/etc/map` so the public URL is just `https://www.mysite.com/home`. Sling handles translating it back and forth.

**Q15. What is the difference between TarMK and MongoMK?**
> **Answer:** TarMK stores data locally in `.tar` files (default for Publish instances). MongoMK uses an external MongoDB cluster, allowing multiple Author instances to share the same repository for high availability.
> **Project Example:** For an intranet site with 500 simultaneous content authors, a single Author instance would crash. You use MongoMK to run 3 Author instances in a cluster sharing one MongoDB database.

---

## 🏗️ SECTION 2: Components & Dialogs (10 Questions)

**Q16. What is the minimal structure of an AEM component?**
> **Answer:** A `.content.xml` file defining the component (`jcr:primaryType="cq:Component"`) and an HTML file for the rendering logic (e.g., `button.html`).
> **Project Example:** To create a custom "Contact Button", you create a folder `contact-button`. Inside, you place a `contact-button.html` containing `<a href="#">${properties.buttonText}</a>`, and a `_cq_dialog` so authors can type the text.

**Q17. What is the difference between `_cq_dialog` and `_cq_design_dialog`?**
> **Answer:** `_cq_dialog` allows authors to configure the component on a specific page. `_cq_design_dialog` (legacy) configured the component across the whole site. Modern AEM uses Content Policies instead of design dialogs.
> **Project Example:** An author uses `_cq_dialog` to type "Click Here" into one specific button. An admin uses a Content Policy (formerly `_cq_design_dialog`) to restrict ALL buttons on the site to only use "Red" or "Blue" colors.

**Q18. What are the three layers of an Editable Template?**
> **Answer:** Structure (locked elements authors can't change, like Headers), Initial Content (starting content for new pages that authors CAN edit), and Policies (rules on what components are allowed).
> **Project Example:** For a "Blog Post Template", the Structure has a locked "Subscribe" newsletter banner. The Initial Content has a blank "Text" component ready to be typed in. The Policy ensures authors can only add Image and Text components, but not Video components.

**Q19. What is a Content Policy?**
> **Answer:** A reusable configuration that restricts a component's allowed behavior and defines Style System options.
> **Project Example:** You have a "Title" component. Instead of writing custom code to limit it to `<h1>` and `<h2>`, you create a Content Policy for the Title component that explicitly disables `<h3>` through `<h6>`.

**Q20. What is the AEM Style System?**
> **Answer:** It allows developers to define CSS classes in a Content Policy. Authors can then select these styles from a dropdown in the component toolbar, applying visual changes without needing developer code.
> **Project Example:** Instead of creating a "Dark Theme Teaser" component and a "Light Theme Teaser" component, you create ONE Teaser. You use the Style System to add a "Dark Mode" option. When the author clicks it, AEM simply adds the `theme-dark` CSS class to the HTML.

**Q21. What is a parsys?**
> **Answer:** A paragraph system. It is a drop-zone container that allows authors to drag and drop other components into it.
> **Project Example:** The main body of a blank page is a parsys (now called a Responsive Grid). An author drags a Text component into it, then drags an Image component below the text. 

**Q22. What is a Core Component and why use it?**
> **Answer:** Adobe's standard, out-of-the-box components (Image, Text, Button). You use them because they are secure, WCAG-accessible, and heavily tested, saving you from reinventing the wheel.
> **Project Example:** You need an Image component with lazy-loading and SEO alt-tags. Instead of writing 500 lines of Java and HTL, you just use the Adobe Core Image Component, which does all of that out-of-the-box.

**Q23. What is the delegation pattern for Core Components?**
> **Answer:** When extending a Core Component, you don't copy its Java code. You create your own Sling Model, inject the Core Model into it, and delegate standard methods to the Core Model, only overriding what you need.
> **Project Example:** You want the standard Core Title component, but you want to append " - Official" to every title. Your Sling Model delegates the `getLink()` method to the Core Model, but you override `getText()` to return `coreModel.getText() + " - Official"`.

**Q24. What is `allowProxy` in a clientlib?**
> **Answer:** It exposes clientlibs stored in `/apps/` to the public via `/etc.clientlibs/`. Dispatcher blocks `/apps/` for security, so proxying is mandatory.
> **Project Example:** Your custom CSS is at `/apps/mysite/clientlibs/main.css`. The Dispatcher blocks it. By setting `allowProxy="{Boolean}true"`, the browser requests `/etc.clientlibs/mysite/clientlibs/main.css`, which Dispatcher allows safely.

**Q25. What is the difference between embedding and depending on clientlibs?**
> **Answer:** "Dependencies" forces the browser to download another clientlib file first (multiple network requests). "Embed" physically merges the code of another clientlib into the current one (one large network request).
> **Project Example:** You have a `base-theme` clientlib and a `homepage` clientlib. If `homepage` EMBEDS `base-theme`, the browser downloads one massive CSS file. If it DEPENDS on it, the browser downloads `base-theme.css`, then `homepage.css`.

---

## ☕ SECTION 3: Sling Models (12 Questions)

**Q26. What is a Sling Model?**
> **Answer:** A Java class (POJO) annotated with `@Model` that automatically maps JCR data into Java variables. It acts as the "Backend Controller" for your HTL frontend.
> **Project Example:** A user types "Contact Us" into a component dialog. The Sling Model uses `@ValueMapValue private String heading;` to automatically fetch that text from the database so your HTML can display it.

**Q27. When should you use `Resource` vs `SlingHttpServletRequest` as adaptable?**
> **Answer:** Use `Resource` when you only need data saved in the component's dialog. Use `Request` when you need URL parameters, cookies, or page-level information.
> **Project Example:** For a simple Text component, use `Resource`. But if you are building a Search component that needs to read `?query=shoes` from the URL, you MUST adapt from the `SlingHttpServletRequest`.

**Q28. Why use `DefaultInjectionStrategy.OPTIONAL`?**
> **Answer:** If a required injection is missing (e.g., an author left a dialog field blank), Sling throws an exception and crashes the whole component. OPTIONAL just makes the variable `null`, allowing you to handle it gracefully.
> **Project Example:** You have a "Subtitle" field. If an author forgets to type a subtitle, a REQUIRED model will crash and show a stack trace on the live website. An OPTIONAL model just sets `subtitle = null` and hides the HTML tag.

**Q29. What does `@PostConstruct` do?**
> **Answer:** A method annotated with `@PostConstruct` runs immediately after Sling has finished injecting all the data. It's used for data formatting and business logic.
> **Project Example:** The dialog gives you a raw date string `2024-12-25`. You write an `init()` method with `@PostConstruct` to format it into `December 25th, 2024` before the HTML renders it.

**Q30. What is the difference between `@ValueMapValue` and `@Inject`?**
> **Answer:** `@ValueMapValue` explicitly tells Sling: "Look in the database properties for this." `@Inject` tells Sling: "Try guessing where this comes from by checking properties, OSGi services, and request attributes." Always use the specific one for performance.
> **Project Example:** When fetching `firstName` from a dialog, using `@ValueMapValue` is lightning fast. Using `@Inject` makes AEM waste time checking if `firstName` is actually an OSGi service before checking the database.

**Q31. What is `@ChildResource` used for?**
> **Answer:** It injects a child JCR node. It is primarily used for Multifields (lists of items in a dialog).
> **Project Example:** You have a "Carousel" component. The author adds 3 slides. These are saved as child nodes. You use `@ChildResource private List<Resource> slides;` to grab all 3 slides at once.

**Q32. How do you map a composite multifield to a List of Java POJOs?**
> **Answer:** Inject the child nodes as a `List<Resource>`. Then, create a second Sling Model for the "Item". In the main model's `@PostConstruct`, loop through the resources and adapt them to the Item model.
> **Project Example:** For a "Team Roster" component. Main model gets `List<Resource> members`. You have a `TeamMemberModel`. You loop through `members` and do `member.adaptTo(TeamMemberModel.class)`, resulting in a clean `List<TeamMemberModel>` for the HTL to loop over.

**Q33. What does `@Self @Via(type = ResourceSuperType.class)` do?**
> **Answer:** It is the core of the Delegation Pattern. It injects an instance of the parent Core Component's Sling Model into your custom Model.
> **Project Example:** You are customizing the Core Teaser. You inject the Core Teaser model using this annotation. When HTL asks for the image URL, your Java code just says `return coreTeaser.getImageUrl();` instead of writing the image fetching logic yourself.

**Q34. What is `@ScriptVariable`?**
> **Answer:** It injects AEM global objects that are usually only available in HTL, like `currentPage`, `pageManager`, or `currentStyle`.
> **Project Example:** You are building a Breadcrumb component. You need to know the name of the current page. You use `@ScriptVariable private Page currentPage;` to instantly get the page object.

**Q35. What is `@OSGiService`?**
> **Answer:** It injects a global Java backend service into your Sling Model so you can use its methods.
> **Project Example:** You have an OSGi service called `SalesforceConnector`. Inside your Sling Model, you use `@OSGiService private SalesforceConnector sfApi;` and then call `sfApi.getLeads()` to display data on the page.

**Q36. How do you debug a Sling Model that isn't adapting (returning null)?**
> **Answer:** Check `/system/console/status-slingmodels` to see if the class registered. Ensure the `adaptables` matches what HTL is passing. Ensure all required injections exist.
> **Project Example:** Your HTL `data-sly-use.model="com.MyModel"` is failing. You realize your Model is adapting from `SlingHttpServletRequest`, but you forgot to pass the request in HTL. It failed silently.

**Q37. What happens if a `@ValueMapValue` field has a different name than the JCR property?**
> **Answer:** Sling looks for a property matching the exact Java variable name. If they differ, you must explicitly declare the name using the annotation parameter.
> **Project Example:** The dialog saves the property as `jcr:title`. Your Java variable is `pageTitle`. You must write `@ValueMapValue(name = "jcr:title") private String pageTitle;` otherwise it will return null.

---

## ⚙️ SECTION 4: OSGi Services (10 Questions)

**Q38. What is the OSGi lifecycle of a component?**
> **Answer:** Installed → Resolved → Activating (`@Activate`) → Active → Deactivating (`@Deactivate`).
> **Project Example:** When you deploy your code, your `EmailService` moves to "Installed". When AEM resolves its dependencies, it becomes "Active" and stays there, ready to send emails whenever a user submits a form.

**Q39. What does `@Designate` do?**
> **Answer:** It links an OSGi Java component to a Configuration interface, allowing administrators to configure the service via the AEM Web Console.
> **Project Example:** You build an `ApiIntegrationService`. You use `@Designate` so an admin can log into `/system/console/configMgr` and type in the API Key and Timeout limits in a nice UI.

**Q40. What is the difference between `@Activate`, `@Modified`, and `@Deactivate`?**
> **Answer:** `@Activate` runs when the service starts. `@Modified` runs when an admin changes a setting in the console (without restarting the service). `@Deactivate` runs when the service is stopping and is used for memory cleanup.
> **Project Example:** In `@Activate`, you open a connection to a database. In `@Modified`, if the admin changes the timeout setting, you update the timeout variable. In `@Deactivate`, you MUST close the database connection to prevent memory leaks.

**Q41. What causes an OSGi component to fail to activate?**
> **Answer:** Missing dependencies (a required `@Reference` isn't running), throwing an exception in the `@Activate` method, or missing a required configuration file.
> **Project Example:** Your `WeatherService` depends on an `ApiKeyService`. If the `ApiKeyService` crashes or isn't deployed, OSGi refuses to start your `WeatherService` because the dependency is unsatisfied.

**Q42. What is `ReferenceCardinality.MULTIPLE`?**
> **Answer:** It allows you to inject a `List` of all running services that implement a specific interface, enabling a plugin architecture.
> **Project Example:** You build a `PaymentProcessor`. You use `MULTIPLE` to collect a list of `PaymentMethod` interfaces. Later, a dev creates a `PayPalMethod` class. OSGi automatically adds PayPal to your list without you touching the `PaymentProcessor` code.

**Q43. What is `immediate = true` in `@Component`?**
> **Answer:** It forces the service to start the exact second AEM boots up. Normally, services are "lazy" and only start the first time another class asks for them.
> **Project Example:** You write a `DailyCleanupScheduler` that deletes old files every night. If it's lazy, it will never start because nobody explicitly "calls" it. You must use `immediate=true` so it wakes up and starts its timer.

**Q44. What is the OSGi Config PID?**
> **Answer:** The Persistent Identity of a configuration. It is usually the fully qualified class name. The JSON configuration file must be named exactly the same as the PID.
> **Project Example:** Your class is `com.mysite.core.services.MyService`. To configure it for the Production environment, your file MUST be named `com.mysite.core.services.MyService.cfg.json`.

**Q45. How do you inject a service user in an OSGi service?**
> **Answer:** Inject `ResourceResolverFactory`. Then call `getServiceResourceResolver()` passing the name of your pre-configured subservice.
> **Project Example:** Your background service needs to read a secure folder. You configure a subservice mapped to the "data-reader" system user. Your Java code requests the resolver using that subservice name to bypass standard user permissions safely.

**Q46. What is the OSGi ConfigurationAdmin?**
> **Answer:** The core OSGi service that reads your `.cfg.json` files and injects those values into your Java classes.
> **Project Example:** When you deploy to QA, ConfigurationAdmin reads your `config.qa` folder, sees `timeout=5000`, and automatically passes that `5000` into your Java service's `@Activate` method.

**Q47. How do you create an OSGi Scheduled Task?**
> **Answer:** Make a class implement `Runnable`. Register it as a service. Then inject the AEM `Scheduler` service and pass it a Cron expression.
> **Project Example:** You want to pull stock prices every 5 minutes. You write a class implementing `Runnable` where `run()` fetches the prices. You schedule it with `0 */5 * * * ?`.

---

## 🌐 SECTION 5: Servlets, HTL & Security (10 Questions)

**Q48. What is the difference between resource-type and path-based servlets?**
> **Answer:** Path servlets are bound to a specific URL (`/bin/api/data`). Resource-type servlets are bound to a component type (`myapp/components/form`). Resource-type is safer because it automatically inherits AEM's security ACLs.
> **Project Example:** A form submission. If you use `/bin/submit`, anyone on the internet can hit it. If you use a resource-type servlet, they can only hit the servlet if they have read-access to the actual Page where the form lives.

**Q49. `SlingSafeMethodsServlet` vs `SlingAllMethodsServlet`?**
> **Answer:** `SafeMethods` only allows read-only HTTP operations (GET, HEAD). `AllMethods` allows destructive operations (POST, PUT, DELETE).
> **Project Example:** An endpoint returning a list of stores in JSON should extend `SlingSafeMethodsServlet`. An endpoint that accepts a "Contact Us" form submission must extend `SlingAllMethodsServlet` to handle the POST.

**Q50. What is context-aware escaping in HTL?**
> **Answer:** HTL automatically sanitizes variables based on where they are placed in the HTML to prevent Cross-Site Scripting (XSS) attacks.
> **Project Example:** A hacker types `<script>alert('hack')</script>` into a dialog. If HTL sees `${text}` inside a `<div>`, it converts the brackets to `&lt;` making it harmless text. If it sees it inside an `<a href="${text}">`, it URL-encodes it.

**Q51. Name 5 HTL statements (data-sly-*).**
> **Answer:** `data-sly-use` (loads Java models), `data-sly-test` (if conditions), `data-sly-list` (for loops), `data-sly-resource` (includes other components), `data-sly-attribute` (sets HTML attributes dynamically).
> **Project Example:** You want to loop over a list of products. You use `data-sly-list.product="${model.productList}"`. Inside the loop, you use `<h1 data-sly-test="${product.isOnSale}">` to only show the H1 if the sale flag is true.

**Q52. What is `<sly>`?**
> **Answer:** It is a fake HTML tag used entirely for HTL logic. It executes code but doesn't actually print an HTML tag to the final web page.
> **Project Example:** You need to load a Sling Model, but you don't want to add an empty `<div>` to your layout just to put the `data-sly-use` attribute on it. You use `<sly data-sly-use...></sly>` which is invisible in the DOM.

**Q53. What is the CSRF protection in AEM?**
> **Answer:** AEM blocks all POST requests unless they include a special Security Token to prove the request came from your actual website, not a hacker's forged form.
> **Project Example:** When a user submits a "Subscribe" form, JavaScript first makes a GET request to AEM to fetch a CSRF token. It then attaches this token to the POST request. AEM verifies it and accepts the form.

**Q54. Redirect vs Forward in a Servlet?**
> **Answer:** Redirect tells the user's browser to make a brand new request to a new URL. Forward handles the routing internally on the server, and the user's browser URL never changes.
> **Project Example:** After a user successfully pays, you `sendRedirect("/success.html")` so their browser URL updates and they can't refresh and double-pay. If a user hits `/legacy-url`, you `forward()` them to `/new-url` internally so the old URL still appears in their browser.

**Q55. What is a Sling filter?**
> **Answer:** A Java class that intercepts every single HTTP request before it reaches your Servlets or Components.
> **Project Example:** You want to log the IP address of every visitor to your site. Instead of writing logging code in 50 different components, you write one Sling Filter that catches all requests, logs the IP, and passes the request along.

**Q56. What sensitive paths must Dispatcher block?**
> **Answer:** Dispatcher must block administrative paths from the public internet.
> **Project Example:** If you don't block `/system/console`, a hacker can access the OSGi dashboard. If you don't block `*.infinity.json`, someone can download your entire database structure by simply typing `.infinity.json` at the end of a URL.

**Q57. What is `rep:policy` in JCR?**
> **Answer:** It is a hidden node under your content that stores the Access Control List (ACL) rules (who is allowed to view or edit this folder).
> **Project Example:** You restrict the `/content/hr-portal` folder so only the "HR-Group" can view it. AEM creates a `rep:policy` node under that folder storing a `rep:GrantACE` rule for the HR-Group.

---

## 📦 SECTION 6: DAM, Query & Cloud (10 Questions)

**Q58. What node type is a DAM asset?**
> **Answer:** `dam:Asset`.
> **Project Example:** When you upload `logo.png` to AEM Assets, it creates a `dam:Asset` node. Under it, the binary image file is stored, along with extracted metadata like the image width, height, and copyright info.

**Q59. How do you adapt a resource to an Asset?**
> **Answer:** `Asset myAsset = resource.adaptTo(Asset.class);`
> **Project Example:** Your component allows an author to select an image path. Your Java code gets that path as a `Resource`, adapts it to an `Asset`, and calls `myAsset.getMetadataValue("dc:title")` to automatically generate the SEO `alt` tag.

**Q60. What is the DAM Update Asset workflow?**
> **Answer:** The default background process that runs every time an image/video is uploaded.
> **Project Example:** You upload a massive 20MB raw photo. The workflow automatically extracts the location metadata, generates a tiny 100kb thumbnail for the authoring UI, and creates a medium web rendition for the live site.

**Q61. What is Query Builder?**
> **Answer:** A simplified API to search the AEM database using simple key-value pairs instead of complex SQL queries.
> **Project Example:** You need to find all PDF files modified last week. Instead of writing raw JCR-SQL2, you tell Query Builder: `type=dam:Asset`, `property=jcr:mimeType`, `property.value=application/pdf`, `daterange.property=jcr:lastModified`.

**Q62. What does `p.limit=-1` in Query Builder mean?**
> **Answer:** It tells the search to return ALL matching results, disabling pagination.
> **Project Example:** You search for "News Articles" with `p.limit=-1`. If there are 50,000 articles, AEM tries to load all 50,000 into RAM at once, crashing the server with an OutOfMemoryError. Always use pagination (e.g., `p.limit=20`).

**Q63. What is a traversal query in Oak?**
> **Answer:** A search query that runs without an index, forcing AEM to manually check every single node in the database.
> **Project Example:** You search for users with `loyaltyTier=Gold`. Since you didn't create a database index for `loyaltyTier`, AEM manually opens all 2 million user nodes to check the property. The query takes 30 seconds and causes a severe performance warning.

**Q64. What is the immutable architecture in AEM Cloud?**
> **Answer:** In AEM as a Cloud Service, application code (`/apps`, `/libs`) is read-only at runtime. You cannot change code directly on the server.
> **Project Example:** In older AEM, a developer could log into CRXDE on the Production server and fix a typo in a script directly. In AEM Cloud, this is blocked. To fix a typo, you MUST commit to Git and wait for the Cloud Manager CI/CD pipeline to deploy it.

**Q65. Content Fragment vs Experience Fragment?**
> **Answer:** Content Fragments are pure raw data (text, numbers) meant for Headless APIs. Experience Fragments are fully designed visual layouts (HTML/CSS) meant to be reused across web pages.
> **Project Example:** A recipe's Ingredients list (just raw text) is a **Content Fragment** sent via JSON to a mobile app. A visual "Header Menu" with a logo, CSS styling, and navigation links is an **Experience Fragment** included on every webpage.

**Q66. What is Sling Model Exporter?**
> **Answer:** A framework that automatically converts a Sling Model Java object into a JSON API endpoint.
> **Project Example:** You build an Article Sling Model that outputs `title` and `author`. By adding `@Exporter(name="jackson")` to the Java class, external apps can now hit `article.model.json` and get a fully formatted JSON response for free.

**Q67. What is a persisted GraphQL query?**
> **Answer:** In Headless AEM, instead of a frontend app sending a massive GraphQL query string in a POST request, you save the query on the AEM server and the frontend just calls it by name via a GET request.
> **Project Example:** An iOS app needs product details. Instead of sending the complex query, it just requests `GET /graphql/execute.json/myproject/get-product`. Because it's a GET request, the CDN/Dispatcher can cache the JSON result instantly.

---

## 🏆 SECTION 7: Senior-Level Questions (8 Questions)

**Q68. What is the service user pattern and why is it essential?**
> **Answer:** It is a security pattern. Instead of a background service running with full "admin" rights, you create a specific system user with the absolute bare minimum permissions required.
> **Project Example:** You build a service that writes to `/content/logs`. Instead of using an admin session, you create a `log-writer-service` user who ONLY has write access to that specific folder. If a hacker exploits your service, they can't touch the rest of the database.

**Q69. A component works in Author but not on Publish. What could be wrong?**
> **Answer:** Permissions, Caching, Missing Content, or Run mode differences.
> **Project Example:** 1) The image wasn't replicated to Publish. 2) The component relies on a Service User that wasn't created on the Publish server. 3) Dispatcher is serving a cached version from yesterday. 4) The API Key OSGi config is missing in the `config.publish` folder.

**Q70. How do you prevent memory leaks in AEM?**
> **Answer:** Explicitly close all resources. Use `try-with-resources` for `ResourceResolver`, and clean up background tasks in `@Deactivate`.
> **Project Example:** Your Java code opens a `ResourceResolver` to read a file but forgets to call `resolver.close()`. This leaves an open database session in memory. After 10,000 users visit the page, the server runs out of RAM and crashes.

**Q71. `resourceResolver.commit()` vs `session.save()`?**
> **Answer:** Both save data to the database. `commit()` is the modern Sling API approach. `save()` is the older underlying JCR API approach. Use the Sling API whenever possible.
> **Project Example:** You write code to update a user's profile. Since you adapted the node to a Sling `Resource`, you should use `resourceResolver.commit()`. Never mix them—calling both in the same method throws a state exception.

**Q72. How does MSM (Multi-Site Manager) inheritance work?**
> **Answer:** A "Blueprint" is the master copy. "Live Copies" are regional child sites that inherit content from the blueprint.
> **Project Example:** You create a master "Global English" site (Blueprint). You create "US English" and "UK English" Live Copies. If you update the global homepage, the changes "roll out" to the US and UK. However, the UK team can "break inheritance" on a specific text component to localize it to British spelling.

**Q73. How do you design a component that calls an external API?**
> **Answer:** Never call APIs directly from HTL or during page rendering. Build an OSGi service to handle the API, wrap it in a caching layer, and handle timeouts gracefully.
> **Project Example:** A component shows real-time flight status. If the Airline API takes 10 seconds to respond, the AEM page takes 10 seconds to load for the user. Instead, an OSGi service fetches the data in the background every minute, caches it, and the component instantly reads from the cache.

**Q74. What SonarQube quality gate thresholds does Cloud Manager enforce?**
> **Answer:** Adobe Cloud Manager runs strict automated code checks. Your code must have 50% unit test coverage, 0 Critical vulnerabilities, and an A-tier reliability rating.
> **Project Example:** You write a brilliant new feature, but you skip writing JUnit tests. When you push to GitHub, the Cloud Manager pipeline fails your deployment with a "Code Coverage is 10%" error, preventing your code from reaching Production.

**Q75. What would you check if a Cloud Manager pipeline fails?**
> **Answer:** Check the Maven Build logs, JUnit reports, SonarQube quality gate dashboard, or the Deployment error logs.
> **Project Example:** The pipeline fails at the "Build" step. You download the Maven log and see `cannot resolve symbol`. You realize you forgot to add a new dependency to your `pom.xml`. If it fails at the "Deploy" step, you check the AEM `error.log` to see if your OSGi bundle crashed on startup.
