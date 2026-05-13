# Day 38 — Scenario-Based Architecture Interview Questions
**Target:** 2–4+ YOE | Demonstrating Enterprise Architecture & Problem Solving

---

> 📌 **Why this matters:** In senior interviews, you aren't asked definitions. You are given a business problem and asked to architect a solution. Use these 50 enterprise scenarios to practice "talking through" your architectural decision-making process.

---

## 🔌 SECTION 1: Third-Party Integrations & APIs

**Scenario 1: Live Flight Status**
* **The Scenario:** The client wants to display live flight departure times on their homepage using an external airline API.
* **The Challenge:** Hitting the API on every page load will crash the site and exceed API rate limits.
* **The Solution:** Use an OSGi Scheduler that runs every 1 minute to fetch the API data and store it in an OSGi cache (like Guava Cache) or JCR (under `/var`). The AEM component's Sling Model reads from this local cache.
* **Example:** A Delta Airlines homepage component reads from AEM memory instantly, while a background Java thread updates that memory every 60 seconds.

**Scenario 2: Secure CRM Form Submission**
* **The Scenario:** Users must submit a "Contact Sales" form that sends PII (Personal Identifiable Information) directly to Salesforce.
* **The Challenge:** Storing PII in AEM's JCR is a security violation.
* **The Solution:** Build a SlingServlet handling a POST request. The Servlet immediately forwards the payload to Salesforce via an HTTP Client, logs success/failure, and returns a JSON response to the browser. The data NEVER touches the JCR.
* **Example:** An enterprise B2B portal captures a lead. The AEM Servlet acts strictly as a secure pass-through proxy.

**Scenario 3: Displaying User Account Balances**
* **The Scenario:** Logged-in users need to see their bank account balance in the top navigation bar.
* **The Challenge:** Dispatcher caches HTML aggressively. If cached, User B might see User A's bank balance.
* **The Solution:** Dispatcher caches the base HTML page with an empty `<div>` for the balance. On page load, client-side JavaScript fires an AJAX GET request to a Sling Servlet. Dispatcher is configured to *bypass* cache for this specific Servlet.
* **Example:** The page loads instantly from the CDN, and 0.5 seconds later, the AJAX call populates `$5,432.00` in the header.

**Scenario 4: Handling Flaky Third-Party APIs**
* **The Scenario:** Your component fetches weather data, but the Weather API frequently times out or goes offline.
* **The Challenge:** If the API hangs, the AEM page rendering hangs, resulting in a blank page for the user.
* **The Solution:** Implement a Circuit Breaker pattern and set strict HTTP connection/read timeouts (e.g., 2000ms). If the API fails, catch the exception, log an error, and return a graceful fallback UI (or hide the component).
* **Example:** If the Weather API is down, the HTL `data-sly-test` sees a null model value and simply hides the weather widget rather than breaking the entire homepage.

**Scenario 5: Securely Storing API Keys**
* **The Scenario:** You need an API key to access Google Maps.
* **The Challenge:** Hardcoding the key in Java or HTL is a massive security risk.
* **The Solution:** Store the key in an OSGi Configuration file using AEM Crypto Support to encrypt the string. In AEM as a Cloud Service, use Cloud Manager Secret Environment Variables (`$[secret:GOOGLE_API_KEY]`).
* **Example:** A developer uses the AEM Crypto UI to generate an encrypted string, pastes it into the OSGi config, and the Java code seamlessly decrypts it in memory.

---

## ⚡ SECTION 2: Dispatcher, Caching & Performance

**Scenario 6: Personalized Hero Banners**
* **The Scenario:** A homepage hero banner must show a "Welcome Back" message for returning users, but a "Sign Up" message for new users.
* **The Challenge:** You cannot cache a page that looks different for different users at the Dispatcher level.
* **The Solution:** Use Sling Dynamic Includes (SDI) or Adobe Target. With SDI, the Dispatcher caches the page but leaves a Server-Side Include (SSI) or client-side AJAX tag for the hero banner.
* **Example:** The homepage is 95% cached. The 5% personalized banner is fetched dynamically, providing the best mix of speed and personalization.

**Scenario 7: Global Site Cache Invalidation**
* **The Scenario:** A client has US, UK, and FR sites. Publishing a page on the US site clears the cache for the UK and FR sites, causing massive CPU spikes.
* **The Challenge:** The Dispatcher `statfileslevel` is configured incorrectly (likely set to 0 or 1).
* **The Solution:** Increase the `statfileslevel` to match the depth of the language roots (e.g., set to 3 or 4). 
* **Example:** With `statfileslevel=3`, publishing `/content/brand/us/en/home` places a `.stat` file in the `/us/en/` folder, flushing only the US English cache and protecting European traffic.

**Scenario 8: Heavy Query Crashing the Server**
* **The Scenario:** A "Store Locator" component searches the JCR for the nearest 50 stores out of 100,000 nodes. The server crashes on high traffic.
* **The Challenge:** The query is triggering a JCR Traversal (checking every node sequentially) because no index exists.
* **The Solution:** Create a custom Oak Lucene Index for the properties being queried (e.g., `zipCode` and `storeType`). 
* **Example:** You deploy an `oak:index/storeLocator` definition. Query execution time drops from 15 seconds to 15 milliseconds.

**Scenario 9: Out of Memory (OOM) on Asset Upload**
* **The Scenario:** When an author uploads a 2GB raw video file, the AEM Author server crashes.
* **The Challenge:** AEM is trying to load the entire 2GB file into RAM to process it through the DAM Update Asset workflow.
* **The Solution:** Offload heavy asset processing to AEM Asset Compute Service (microservices). Do not process massive video files directly on the Author JVM.
* **Example:** The author uploads the video. AEM instantly uploads the binary to Azure Blob Storage, and serverless Adobe workers generate the thumbnails safely in the cloud.

**Scenario 10: Clientlib Bloat**
* **The Scenario:** The homepage takes 8 seconds to load because it downloads a 5MB CSS/JS file containing code for every component on the entire site.
* **The Challenge:** The client is embedding all clientlibs into one massive global file.
* **The Solution:** Implement async loading and split clientlibs. Use the `categories` property effectively. Only load the global framework in the header, and load component-specific clientlibs dynamically using HTL where they are actually used.
* **Example:** The "Video Player" JS is only downloaded if the user actually visits a page containing the Video component.

---

## 🧩 SECTION 3: Component & Template Architecture

**Scenario 11: Multi-Tenant Architecture**
* **The Scenario:** Your company owns 5 different brands (Brand A, Brand B, etc.). They want to share the same AEM instance to save money.
* **The Challenge:** Brand A and Brand B need completely different logos, CSS, and component designs, but you don't want to write duplicate Java code.
* **The Solution:** Use Core Components and AEM Style System. Create one set of global components. Use Editable Template Policies to apply Brand A's CSS to Brand A's site, and Brand B's CSS to Brand B's site.
* **Example:** Both brands use the exact same `Hero` component. Brand A's authors select "Rounded Red Button" from the policy dropdown, while Brand B gets "Square Blue Button" automatically.

**Scenario 12: Restricting Author Chaos**
* **The Scenario:** Authors keep putting massive Image components inside tiny sidebar columns, breaking the website layout.
* **The Challenge:** Authors have too much freedom in the parsys (Responsive Grid).
* **The Solution:** Use Content Policies. Configure the Layout Container (parsys) policy for the sidebar to ONLY allow Text and Button components, explicitly disabling the Image component.
* **Example:** When the author clicks "Insert Component" in the sidebar, the "Image" option doesn't even appear in the menu.

**Scenario 13: Deprecating an Old Component**
* **The Scenario:** You built a "V1 Slider" two years ago. You just built a "V2 Carousel". You want authors to stop using V1, but not break old pages.
* **The Challenge:** Deleting V1 code will cause exceptions on thousands of old pages.
* **The Solution:** Create a Component Group called ".hidden". Change V1's `componentGroup` to ".hidden". It disappears from the authoring menu for new pages, but existing V1 components keep rendering perfectly.
* **Example:** Authors can no longer drag V1 onto new pages, forcing them to use V2. Developers can slowly run a script to migrate old pages to V2 in the background.

**Scenario 14: Sharing Header/Footer Across 1000 Pages**
* **The Scenario:** The client wants a global footer. If they update a copyright date, it must update on all 1000 pages instantly.
* **The Challenge:** If the footer is a standard component on an Editable Template, authors have to unlock it and publish 1000 pages.
* **The Solution:** Use an Experience Fragment (XF). Build the footer as an XF. Place the XF component in the template structure. 
* **Example:** The author edits the single XF node at `/content/experience-fragments/footer`. They publish the XF once, and it instantly updates across the entire site without republishing any pages.

**Scenario 15: Component with 50 Configuration Fields**
* **The Scenario:** A complex "Interactive Map" component dialog has 50 different fields. Authors are overwhelmed and making mistakes.
* **The Challenge:** The dialog is a massive, scrolling mess of input fields.
* **The Solution:** Use Dialog Tabs (`cq:WidgetCollection`) to group fields logically (e.g., "Basic Setup", "Markers", "Advanced Settings"). Use `cq-dialog-dropdown-showhide` to dynamically hide irrelevant fields based on previous selections.
* **Example:** If the author selects "Map Type: Simple", the 30 fields relating to "Advanced 3D Map" instantly disappear from the dialog, cleaning up the UI.

---

## 🔒 SECTION 4: Security & Permissions

**Scenario 16: Temporary Vendor Access**
* **The Scenario:** An external SEO agency needs to edit metadata on the site for exactly 2 weeks.
* **The Challenge:** Giving them "admin" access is a severe security risk. Forgetting to remove their access in 2 weeks is a compliance violation.
* **The Solution:** Create a specific User Group ("SEO-Vendors"). Apply ACLs granting them ONLY `jcr:modifyProperties` on `/content/mysite`. Do not grant node creation/deletion. Set an expiration date on their user profiles.
* **Example:** The vendor can change the page title and description, but cannot delete pages or add new components. In 14 days, their login automatically stops working.

**Scenario 17: Service User for Content Migration**
* **The Scenario:** You wrote a Java OSGi service to automatically import thousands of articles from a legacy system into AEM.
* **The Challenge:** The service needs write-access to the JCR, but using `loginAdministrative()` is deprecated and blocked by Adobe.
* **The Solution:** Implement the Service User pattern. Create a system user `migration-service-user`. Grant it write access to `/content/articles`. Map it via OSGi `ServiceUserMapper`. Use `getServiceResourceResolver()`.
* **Example:** The Java code securely obtains a resolver that acts exactly like a user who is only allowed to edit the `/content/articles` folder.

**Scenario 18: Protecting Internal PDFs**
* **The Scenario:** The HR department uploads employee handbook PDFs to the DAM. They must be strictly hidden from the public internet.
* **The Challenge:** By default, assets published to the DAM are accessible if someone guesses the URL.
* **The Solution:** Create a Closed User Group (CUG) for the `/content/dam/hr` folder. Apply the CUG to the "Employees" group. Configure Dispatcher to deny access to this path unless a valid AEM login token is present.
* **Example:** If an anonymous user guesses `www.company.com/content/dam/hr/handbook.pdf`, they get a 401 Unauthorized redirect to a login screen.

**Scenario 19: Preventing Cross-Site Scripting (XSS)**
* **The Scenario:** A malicious author pastes `<script>stealCookies()</script>` into a Text component dialog.
* **The Challenge:** If AEM renders that script to the live site, customer data gets stolen.
* **The Solution:** HTL provides automatic context-aware escaping. If the variable is output in the HTML body, it converts the tags to HTML entities (`&lt;script&gt;`). Additionally, AEM's AntiSamy OSGi configuration automatically strips malicious tags during the authoring save process.
* **Example:** The malicious script is saved safely as plain text and displays harmlessly on the screen as raw text, not executable JavaScript.

**Scenario 20: CSRF Token Errors on Custom Forms**
* **The Scenario:** You build a custom "Feedback" form. When users submit it, the server rejects it with a `403 Forbidden` error.
* **The Challenge:** AEM's default security blocks POST requests lacking a CSRF token to prevent Cross-Site Request Forgery.
* **The Solution:** Have the frontend JavaScript make a GET request to `/libs/granite/csrf/token.json` first, retrieve the token, and attach it to the POST request header (`CSRF-Token`).
* **Example:** The browser proves to AEM that the form submission originated from the actual website, not a hacker's phishing email.

---

## 🗃️ SECTION 5: Backend & JCR Manipulations

**Scenario 21: Bulk Property Update**
* **The Scenario:** The client rebranded. You need to change the word "OldBrand" to "NewBrand" in the text properties of 10,000 pages.
* **The Challenge:** Doing this manually will take weeks. Writing a Java Servlet might time out the browser.
* **The Solution:** Write an OSGi Job or use the **Groovy Console**. A Groovy script can iterate through the JCR using QueryBuilder, update the property, and save the session in batches of 500 to prevent OOM errors.
* **Example:** You execute a 15-line Groovy script in the AEM Groovy Console. 5 minutes later, all 10,000 pages are updated perfectly without deploying any Java code.

**Scenario 22: ResourceResolver Memory Leaks**
* **The Scenario:** The AEM server becomes slower over the course of a week until it finally crashes with a Heap OutOfMemoryError.
* **The Challenge:** A developer wrote a Servlet that opens a `ResourceResolver` but forgot to close it.
* **The Solution:** Always use Java's `try-with-resources` block when instantiating a ResourceResolver. This guarantees the resolver is closed automatically, even if an exception is thrown in the middle of the code.
* **Example:** `try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(map)) { ... }` ensures the memory is instantly freed when the block ends.

**Scenario 23: Listening for JCR Changes**
* **The Scenario:** Every time an author uploads a new PDF to a specific DAM folder, you need to instantly send an email to the legal team.
* **The Challenge:** You need a way to detect the upload event the moment it happens.
* **The Solution:** Create an OSGi Event Listener or a JCR Observation Event Handler listening specifically to the `NODE_ADDED` event on the `/content/dam/legal` path.
* **Example:** The author clicks "Upload". The JCR fires an event. Your Java class catches the event, extracts the file name, and triggers an email API.

**Scenario 24: Finding Unused Assets**
* **The Scenario:** The DAM is 5 Terabytes. The client wants to delete images that aren't being used on any pages to save disk space.
* **The Challenge:** How do you cross-reference 5TB of assets against 100,000 pages?
* **The Solution:** Use ACS AEM Commons **Report Builder** or write a script utilizing AEM's built-in `ReferenceSearch` API. The API checks the repository for any hardcoded links to the asset's path.
* **Example:** You generate a CSV report showing exactly which images have zero JCR references pointing to them, allowing safe deletion.

**Scenario 25: Custom Workflow Step**
* **The Scenario:** When an author submits a page for approval, a custom Java process must validate that the page contains a Title and a SEO Description before moving to the "Approver" inbox.
* **The Challenge:** Standard workflows don't know your business rules.
* **The Solution:** Create a custom OSGi service implementing `WorkflowProcess`. Add it as a "Process Step" in the AEM Workflow Model. 
* **Example:** If the Java code sees `seoDescription` is null, it routes the workflow back to the "Author" step with a comment: "Please add SEO data."

---

## 🚀 SECTION 6: Headless & Modern Architecture

**Scenario 26: Mobile App Content Delivery**
* **The Scenario:** An iOS app needs to display articles created in AEM. The app doesn't understand HTML.
* **The Challenge:** AEM natively outputs HTML web pages.
* **The Solution:** Use Content Fragments for the articles. Expose them using AEM's native GraphQL API. The iOS app sends a GraphQL query asking only for the `title` and `bodyText`.
* **Example:** The mobile app receives a clean, lightweight JSON response instead of parsing through `<div class="article-wrapper">` tags.

**Scenario 27: React Frontend with AEM Authoring**
* **The Scenario:** The client wants a cutting-edge React frontend, but marketers still want to use the AEM Authoring UI to drag-and-drop components.
* **The Challenge:** Traditional React apps are disconnected from AEM's authoring overlay.
* **The Solution:** Implement the AEM SPA Editor. Build React components that map to AEM components using the `@adobe/aem-react-editable-components` SDK. Use Sling Model Exporter to feed JSON to React.
* **Example:** The author drags a "Hero" component into the AEM UI. AEM generates JSON. The React app reads the JSON and dynamically renders the `<HeroReactComponent />` instantly in the author's preview window.

**Scenario 28: Third-Party PIM Integration**
* **The Scenario:** Product data (price, SKU, inventory) lives in a Product Information Management (PIM) system. AEM needs to display it on the product page.
* **The Challenge:** Copying 1 million products into AEM's JCR daily is inefficient and causes data lag.
* **The Solution:** Use AEM Commerce Integration Framework (CIF). Do not store product data in the JCR. Store only a generic "Product Component" in AEM. The component makes real-time GraphQL calls to the PIM via CIF.
* **Example:** When the page loads, AEM fetches the HTML layout, but the price "$19.99" is fetched directly from the PIM's API in real-time.

**Scenario 29: Persisted GraphQL Queries**
* **The Scenario:** Your Headless AEM setup is receiving massive GraphQL POST requests from a web app. The CDN cannot cache POST requests, so AEM is taking heavy traffic load.
* **The Challenge:** GraphQL inherently relies on dynamic POST requests.
* **The Solution:** Use Persisted Queries. The developer saves the complex query inside AEM beforehand. The web app now makes a simple HTTP GET request to `/graphql/execute.json/myproject/article-query`.
* **Example:** Because it is a standard GET request, the Dispatcher and CDN cache the JSON response for 10 minutes, protecting the AEM server from traffic spikes.

**Scenario 30: Omnichannel Content Reusability**
* **The Scenario:** You need the exact same "Privacy Policy" text to appear on the website footer, the iOS app, and an in-store kiosk.
* **The Challenge:** Hardcoding the text in three different platforms creates a maintenance nightmare for the legal team.
* **The Solution:** Create a single Content Fragment in AEM. The website includes the CF using the AEM Content Fragment component. The iOS app and Kiosk fetch the exact same CF via GraphQL API.
* **Example:** The legal team edits the text in ONE place (AEM). Once published, all three platforms update simultaneously.

---

**Scenario 31: Blue/Green Deployment with Dispatcher**
* **The Scenario:** You need to deploy a massive code update with zero downtime.
* **The Challenge:** Restarting AEM Publish instances causes 5 minutes of downtime.
* **The Solution:** Use a Blue/Green deployment strategy at the Load Balancer/Dispatcher level. Stand up a new "Green" AEM server, deploy code to it, test it, and then switch the Dispatcher traffic from the "Blue" server to the "Green" server instantly.
* **Example:** Users on the site experience zero connection drops. If the Green server fails, you instantly flip the Dispatcher route back to the Blue server.

**Scenario 32: Synchronizing Multi-Region User Data**
* **The Scenario:** Users can update their profile in the US, but then travel to Europe and log into the EU site.
* **The Challenge:** AEM Publish instances don't share user data natively. Replicating user profiles back to Author and out to EU Publish takes too long.
* **The Solution:** AEM is not a CRM. Do not store live, global user state in AEM. Integrate AEM with an external Identity Provider (Auth0, Okta) or a centralized CRM (Salesforce).
* **Example:** The AEM login component authenticates against Okta via JWT. The user's profile data is fetched dynamically from Okta, ensuring it is always perfectly synchronized globally.

**Scenario 33: A/B Testing Component Variations**
* **The Scenario:** Marketing wants to test if a Red Button gets more clicks than a Blue Button on the homepage.
* **The Challenge:** Building custom logic into the Sling Model for every A/B test pollutes the codebase.
* **The Solution:** Integrate AEM with Adobe Target. Build an Experience Fragment for the Red version and one for the Blue version. Export both to Target.
* **Example:** Adobe Target handles the logic of serving Red to 50% of users and Blue to 50% of users. The AEM codebase remains clean and completely agnostic to the A/B test.

**Scenario 34: Dealing with Large DAM Migrations**
* **The Scenario:** The client has 2 million images in an old FTP server that need to be in AEM tomorrow.
* **The Challenge:** Uploading them via the AEM Web UI or standard API will take 3 weeks.
* **The Solution:** Use the AEM Bulk Import Tool or a specialized script (like Oak Run) to ingest assets directly at the Oak Repository level, bypassing the heavy workflow processing, and run the processing later via Asset Compute.
* **Example:** The binaries are dumped straight into the Azure Datastore, and metadata is mapped via a massive CSV file, drastically cutting migration time.

**Scenario 35: Handling GDPR & Data Deletion**
* **The Scenario:** A user requests their account and all personal data be deleted from your platform (GDPR Right to be Forgotten).
* **The Challenge:** Personal data might be scattered across AEM user nodes, custom log files, or submitted form nodes.
* **The Solution:** Design the architecture so PII is never stored in AEM. If it must be (e.g., AEM Communities), use AEM's built-in GDPR UI or write an OSGi service that queries the JCR for the specific User ID and deletes the associated nodes.
* **Example:** The OSGi service receives the webhook from the Legal system, searches `/home/users` and `/var/forms` for the user's ID, and hard-deletes the nodes.

**Scenario 36: Preventing Link Rot**
* **The Scenario:** Authors constantly rename pages or move them to different folders, breaking bookmarks for users.
* **The Challenge:** AEM's internal JCR links update automatically, but external users get 404 pages.
* **The Solution:** Implement ACS AEM Commons Redirect Manager or configure Dispatcher vanity URLs/Rewrite rules automatically upon page move.
* **Example:** When an author moves `/content/about` to `/content/company/about`, the Redirect Manager automatically creates a 301 Permanent Redirect rule. The user's old bookmark seamlessly redirects to the new page, preserving SEO ranking.

**Scenario 37: Complex Navigation Menus**
* **The Scenario:** A mega-menu needs to display the titles of 500 pages dynamically based on the JCR structure.
* **The Challenge:** Fetching 500 pages and their properties on every page load takes 3 seconds.
* **The Solution:** The Sling Model should traverse the tree ONCE and cache the resulting JSON/HTML structure in memory. Invalidate the cache only when the tree changes.
* **Example:** The first user to visit the site experiences a 3-second load. The next 100,000 users get a 5-millisecond load because the menu is read directly from the OSGi RAM cache.

**Scenario 38: Third-Party Authentication (SSO)**
* **The Scenario:** The client wants employees to log into AEM Author using their corporate Microsoft Azure AD credentials.
* **The Challenge:** AEM uses its own internal Oak user database.
* **The Solution:** Configure SAML 2.0 Authentication Handler in OSGi. Add the Identity Provider (IdP) certificate to the AEM TrustStore.
* **Example:** When an author goes to `localhost:4502`, they are redirected to Microsoft's login page. After authenticating, Microsoft sends a SAML assertion to AEM, which automatically creates/syncs an Oak user and logs them in.

**Scenario 39: Custom Clientlib Minification**
* **The Scenario:** Your frontend team uses modern ES6 JavaScript and SCSS, but AEM's default minifier (YUI) breaks on modern JS syntax.
* **The Challenge:** The AEM clientlib manager fails to compile the files.
* **The Solution:** Switch the OSGi configuration for HTML Library Manager from YUI to GCC (Google Closure Compiler), or bypass AEM entirely by using Webpack in your frontend build process to pre-compile the files before AEM sees them.
* **Example:** The frontend team builds a `dist/main.bundle.js` using Webpack. AEM simply serves this pre-compiled, heavily optimized file without trying to minify it itself.

**Scenario 40: Sharing Content Across Unrelated Domains**
* **The Scenario:** The client runs a Car site (`cars.com`) and a Boat site (`boats.com`) on the same AEM instance. They want to share the exact same "Financing Rates" table on both.
* **The Challenge:** Duplicating the content means someone has to update it twice when rates change.
* **The Solution:** Create the data in a Content Fragment located in a shared folder (`/content/dam/shared/rates`). Both the Car site and Boat site use a component that points to this single CF.
* **Example:** The finance team updates the rate to 5% in the CF. When they hit publish, both `cars.com` and `boats.com` display the 5% rate instantly.

**Scenario 41: Securing Sensitive Workflow Data**
* **The Scenario:** A custom workflow handles HR approvals for salary increases.
* **The Challenge:** All authors can see the workflow payloads in the inbox, exposing salary data.
* **The Solution:** Workflow instances run under specific permissions. You must isolate the payload in a secure JCR folder with a strict ACL, and ensure the workflow models are only assigned to specific CUGs.
* **Example:** The workflow payload is stored in `/var/hr-secure`. Standard authors cannot view this folder, so even if they try to search for workflow histories, Oak denies them access to the data.

**Scenario 42: Fixing "Stale" Component Dialogs**
* **The Scenario:** You added a new field to a component dialog, but authors are complaining they don't see it until they clear their browser cache.
* **The Challenge:** The author's browser aggressively cached the old `_cq_dialog.html` file.
* **The Solution:** AEM Cloud Service handles this automatically, but for on-prem, use a versioned clientlib URL or instruct authors to use a hard refresh.
* **Example:** You teach the authoring team to use `Ctrl+F5`. Alternatively, you increment the clientlib version so the browser is forced to fetch the new dialog structure.

**Scenario 43: Slow Package Deployments**
* **The Scenario:** Deploying a content package of 50,000 images takes 6 hours.
* **The Challenge:** The Package Manager is trying to extract, validate, and index all 50,000 assets simultaneously.
* **The Solution:** Break the package into smaller chunks. Disable workflow launchers for the `dam/update/asset` workflow *before* installing the package to prevent AEM from trying to process 50,000 images at once.
* **Example:** You install the assets quickly. Later, you run a bulk workflow execution script during off-peak hours to generate the thumbnails at a controlled pace.

**Scenario 44: Custom Error Pages**
* **The Scenario:** When a user hits a broken link, they see AEM's default ugly `404 Resource Not Found` stack trace.
* **The Challenge:** Hardcoding a static HTML 404 page is bad because marketing can't edit it.
* **The Solution:** Use ACS AEM Commons Error Page Handler. Create a real AEM page at `/content/mysite/en/errors/404`. Configure the handler to intercept 404 errors and render that specific page.
* **Example:** Marketing builds a beautiful 404 page with a search bar and a "Return to Home" button. Dispatcher caches it, and all broken links serve this branded experience.

**Scenario 45: Preventing Infinite Loops in Events**
* **The Scenario:** You write an OSGi Event Listener that triggers when a property is updated. The listener updates another property on that same node.
* **The Challenge:** Updating the node triggers the event listener again, creating an infinite loop that crashes the server.
* **The Solution:** Check the `userId` in the event payload. If the event was triggered by your own Service User (`my-update-service`), ignore the event.
* **Example:** The listener fires. It checks the user: "Did *I* make this change? Yes. Okay, do nothing." The loop is broken instantly.

**Scenario 46: Cross-Origin Resource Sharing (CORS)**
* **The Scenario:** A standalone React app hosted on `myapp.vercel.app` is trying to fetch JSON from your AEM server, but the browser blocks it.
* **The Challenge:** Browsers block requests to different domains for security unless explicitly permitted.
* **The Solution:** Configure the "Adobe Granite CORS Policies" in AEM OSGi. Add `https://myapp.vercel.app` to the allowed origins list.
* **Example:** AEM responds with `Access-Control-Allow-Origin: https://myapp.vercel.app`, and the browser permits the React app to read the JSON data.

**Scenario 47: Scheduling Massive Content Deletions**
* **The Scenario:** The client wants to delete 10,000 expired promotional pages at midnight on January 1st.
* **The Challenge:** Nobody wants to log in at midnight on New Year's Eve to click "Delete" 10,000 times.
* **The Solution:** Use AEM's native "Manage Publication" feature with the "Later" option.
* **Example:** On December 20th, the author selects the folder, clicks "Unpublish Later", and sets the date to Jan 1st 00:00. AEM's scheduler handles the rest automatically while the team sleeps.

**Scenario 48: Mocking OSGi Services in Tests**
* **The Scenario:** You are writing a JUnit 5 test for a Sling Model. The model calls `PaymentService.getOptions()`. The real service hits a live Stripe API.
* **The Challenge:** Running unit tests should never hit real external APIs.
* **The Solution:** Use Mockito to create a mock `PaymentService`. Use `AemContext.registerService()` to inject your mock into the test context.
* **Example:** `when(mockPaymentService.getOptions()).thenReturn("Visa, Mastercard");`. Your Sling Model runs perfectly in the test environment using the fake data.

**Scenario 49: Syncing Environments**
* **The Scenario:** The QA server has missing images and old pages, making it impossible to test new code properly.
* **The Challenge:** QA content is out of sync with Production content.
* **The Solution:** Use a tool like VLT (Vault Tool) or Grabbit to clone content from Prod to QA. In modern AEM Cloud, use the Cloud Manager "Content Copy" feature.
* **Example:** You click a button in Cloud Manager. It takes a secure snapshot of Production `/content` and overlays it onto the Stage/QA environment automatically.

**Scenario 50: The "Blank Page" Emergency**
* **The Scenario:** A developer deploys a minor CSS fix to Production. Suddenly, the entire website goes blank (White Screen of Death).
* **The Challenge:** The client is losing $10,000 a minute. You need to find the cause instantly.
* **The Solution:** 
  1. Check the browser console (is the CSS file returning a 404?).
  2. Bypass the Dispatcher (add `?nocache=true` to the URL. If it loads, it's a Dispatcher cache corruption issue. If it's still blank, it's an AEM issue).
  3. Check AEM `error.log` for massive stack traces.
* **Example:** You use `?nocache=true` and see a NullPointerException in the log. A developer forgot to use `DefaultInjectionStrategy.OPTIONAL` on a Sling Model, crashing the entire layout. You quickly push a hotfix and manually clear the Dispatcher cache.
