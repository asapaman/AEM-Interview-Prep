# Dispatcher — Complete Guide
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What is the Dispatcher?

The **Dispatcher** is an Apache httpd module (a `.so` plugin for Apache web server) that acts as AEM's **caching and security layer**. It sits in front of AEM Publish instances and performs three critical jobs:

1. **Caching** — Stores rendered HTML pages as flat files on disk. Serves them directly WITHOUT hitting AEM — dramatically reducing load.
2. **Load Balancing** — Distributes requests across multiple AEM Publish instances.
3. **Security** — Blocks requests to sensitive AEM paths before they ever reach AEM.

### Where Dispatcher Fits in the Architecture

```
Internet
   │
   ▼
[CDN - Fastly/Akamai/CloudFront]     ← Global edge caching
   │ Cache Miss
   ▼
[Apache httpd + Dispatcher Module]   ← Your Dispatcher server
   │ Cache Miss
   ▼
[AEM Publish Instance(s)]            ← AEM renders the page
   │
   └── Cache Result stored at Dispatcher
```

### Dispatcher is NOT

- ❌ A separate Java application — it's an Apache httpd module
- ❌ A Varnish or Nginx setup — it's Adobe's own module
- ❌ Running on AEM's JVM — it runs in Apache's process

---

## 📁 Dispatcher Configuration Structure

```
/etc/httpd/                                    ← Apache root
  ├── conf/
  │     └── httpd.conf                         ← Main Apache config (loads modules)
  ├── conf.d/
  │     ├── available_vhosts/
  │     │     ├── mysite-author.vhost           ← Author vhost
  │     │     └── mysite-publish.vhost          ← Publish vhost
  │     └── enabled_vhosts/                    ← Symlinks to active vhosts
  │           └── mysite-publish.vhost → ../available_vhosts/mysite-publish.vhost
  └── conf.dispatcher.d/
        ├── available_farms/
        │     ├── default.farm                  ← Main dispatcher farm config
        │     └── author.farm                   ← Author-specific farm
        ├── enabled_farms/
        │     └── default.farm → ../available_farms/default.farm
        ├── cache/
        │     └── rules.any                     ← Cache rules
        ├── filters/
        │     └── filters.any                   ← Security filter rules
        ├── renders/
        │     └── default_renders.any           ← AEM publish instance addresses
        └── vhosts/
              └── default_vhosts.any            ← Virtual host definitions
```

---

## 📄 Dispatcher Farm Configuration — Complete Breakdown

The `.farm` file is the heart of Dispatcher configuration:

```apache
# /etc/httpd/conf.dispatcher.d/available_farms/default.farm

/farms {

    /mysite {

        # ── VIRTUAL HOSTS ──────────────────────────────────────────────────
        # Which host names this farm handles
        /virtualhosts {
            "www.mysite.com"
            "mysite.com"
            "*.mysite.com"
        }

        # ── RENDER SERVERS (AEM Publish instances) ──────────────────────────
        /renders {
            /renderer0 {
                /hostname "aem-publish-01"    # AEM publish hostname/IP
                /port     "4503"
                /timeout  "60000"             # 60 seconds connection timeout
                /receiveTimeout "300000"      # 5 min for slow page renders
            }
            /renderer1 {
                /hostname "aem-publish-02"    # Second publish instance
                /port     "4503"
                /timeout  "60000"
            }
        }

        # ── SECURITY FILTERS ────────────────────────────────────────────────
        /filter {
            # DENY EVERYTHING first (default-deny security model)
            /0001 { /type "deny" /glob "*" }

            # Allow GET/HEAD to public content
            /0010 { /type "allow" /method "(GET|HEAD)" /url "/content/mysite/*" }
            /0011 { /type "allow" /method "(GET|HEAD)" /url "/content/dam/*" }

            # Allow static resources
            /0020 { /type "allow" /url "/etc.clientlibs/*" }
            /0021 { /type "allow" /url "/libs/granite/csrf/token.json" }

            # Allow AEM sitemap
            /0030 { /type "allow" /url "/sitemap.xml" }
            /0031 { /type "allow" /url "/robots.txt" }

            # Allow health check endpoint
            /0040 { /type "allow" /url "/bin/mysite/health" }

            # BLOCK sensitive paths (even if accidentally not blocked by deny-all)
            /0100 { /type "deny" /url "/system/*" }         # OSGi console
            /0101 { /type "deny" /url "/crx/*" }            # CRXDE
            /0102 { /type "deny" /url "/libs/granite/*" }   # Admin tools
            /0103 { /type "deny" /url "/bin/wcm/*" }        # WCM commands
            /0104 { /type "deny" /url "*/jcr:content/*" }   # Direct JCR access
            /0105 { /type "deny" /url "*.json" /selectors "[[(tidy|infinity|children|pages|angular|caconfig|1|2|3)]+" }
        }

        # ── CACHE CONFIGURATION ─────────────────────────────────────────────
        /cache {
            # Where to store cached files on disk
            /docroot "/var/www/vhost/mysite/cache"

            # ── Cache Rules ────────────────────────────────────────────────
            /rules {
                /0001 { /type "deny"  /glob "*" }              # Deny caching everything
                /0002 { /type "allow" /glob "*.html" }         # Cache HTML pages
                /0003 { /type "allow" /glob "*.css" }          # Cache CSS
                /0004 { /type "allow" /glob "*.js" }           # Cache JavaScript
                /0005 { /type "allow" /glob "*.png" }          # Cache images
                /0006 { /type "allow" /glob "*.jpg" }
                /0007 { /type "allow" /glob "*.gif" }
                /0008 { /type "allow" /glob "*.svg" }
                /0009 { /type "allow" /glob "*.ico" }
                /0010 { /type "allow" /glob "*.woff2" }        # Cache fonts
                /0011 { /type "allow" /glob "/robots.txt" }
                /0012 { /type "allow" /glob "/sitemap.xml" }
                /0013 { /type "deny"  /glob "*.nocache.html" } # Opt-out pattern
            }

            # ── What NOT to cache ──────────────────────────────────────────
            # Requests with these parameters won't be cached
            /ignoreUrlParams {
                /0001 { /type "allow" /glob "utm_*" }          # Ignore UTM params (cache same page)
                /0002 { /type "allow" /glob "fbclid" }         # Ignore Facebook click ID
                /0003 { /type "allow" /glob "gclid" }          # Ignore Google click ID
                # All other params: prevent caching
            }

            # ── Cookie-based cache invalidation ────────────────────────────
            /cookies {
                # Requests with these cookies are NOT cached
                /cookie0 { /name "wcmmode" }
                /cookie1 { /name "login-token" }       # Authenticated users
                /cookie2 { /name "AdobeOrg" }
            }

            # ── Invalidation (cache flush) ──────────────────────────────────
            /invalidate {
                /0000 { /type "allow" /glob "*" }
            }

            # ── Who can send flush requests ────────────────────────────────
            /allowedClients {
                /0000 { /type "deny"  /glob "*" }
                /0001 { /type "allow" /glob "10.0.0.*" }       # AEM Publish IP range
                /0002 { /type "allow" /glob "127.0.0.1" }
            }

            # ── Header preservation ────────────────────────────────────────
            /headers {
                "Cache-Control"
                "Content-Disposition"
                "Content-Type"
                "Expires"
                "Last-Modified"
                "X-Content-Type-Options"
            }

            # ── Serve stale content while re-fetching (grace period) ────────
            /grace "10"                                         # 10 seconds
        }

        # ── LOAD BALANCING ──────────────────────────────────────────────────
        /statistics {
            /categories {
                /html { /glob "*.html" }
                /others { /glob "*" }
            }
        }
    }
}
```

---

## 🗂️ Virtual Host Configuration

```apache
# /etc/httpd/conf.d/available_vhosts/mysite-publish.vhost

<VirtualHost *:80>
    ServerName www.mysite.com
    ServerAlias mysite.com *.mysite.com

    # Redirect HTTP to HTTPS
    RewriteEngine On
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [L,R=301]
</VirtualHost>

<VirtualHost *:443>
    ServerName www.mysite.com
    ServerAlias mysite.com

    # SSL Configuration
    SSLEngine on
    SSLCertificateFile    /etc/ssl/certs/mysite.crt
    SSLCertificateKeyFile /etc/ssl/private/mysite.key

    # Document root (where Dispatcher caches files)
    DocumentRoot /var/www/vhost/mysite/cache

    # Dispatcher module configuration
    <Directory />
        <IfModule disp_apache2.c>
            ModMimeUsePathInfo On
        </IfModule>
        Options FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    # Handle .html files
    <IfModule disp_apache2.c>
        SetHandler dispatcher-handler
        ModMimeUsePathInfo On
    </IfModule>

    # Redirect /home to /en/home (URL mapping)
    Redirect permanent /home /content/mysite/en/home.html

    # Logging
    ErrorLog  /var/log/httpd/mysite-error.log
    CustomLog /var/log/httpd/mysite-access.log combined
</VirtualHost>
```

---

## 🔄 Cache Invalidation — How It Works

### Step-by-Step Flow

```
1. Author publishes page /content/mysite/en/home

2. AEM Author → Replication Agent → sends page to AEM Publish

3. AEM Publish receives content AND sends cache invalidation
   HTTP DELETE to Dispatcher:
   DELETE /dispatcher/invalidate.cache
   CQ-Action: Activate
   CQ-Handle: /content/mysite/en/home
   CQ-Path: /content/mysite/en/home

4. Dispatcher receives invalidation signal

5. Dispatcher marks cache files as INVALID
   Strategy: stat file invalidation
   - Finds /var/www/vhost/mysite/cache/content/mysite/en/.stat
   - Touches (updates timestamp) this stat file

6. Next request for /content/mysite/en/home.html:
   - Dispatcher compares page file's timestamp vs .stat file
   - Page file is OLDER than .stat → STALE → forward to AEM
   - AEM renders fresh content
   - Dispatcher stores new HTML file
   - Subsequent requests: serve from cache ✅
```

### The .stat File System

```
/var/www/vhost/mysite/cache/
  └── content/
        └── mysite/
              └── .stat                    ← Stats at this level affect all files below
              └── en/
                    ├── .stat              ← Stats at en/ level
                    ├── home.html          ← Cached page
                    ├── about.html
                    └── products/
                          ├── .stat
                          └── laptop.html
```

When `/content/mysite/en` is invalidated:
- Dispatcher touches `/content/mysite/en/.stat`
- All HTML files in `en/` and subdirectories are now considered stale
- They'll be re-fetched from AEM on next request

### Statfile Level

```apache
/statfileslevel "3"   # How many directory levels above the cached file to place .stat files

# Example for /content/mysite/en/home.html:
# Level 0 → /var/cache/.stat (invalidates EVERYTHING)
# Level 1 → /var/cache/content/.stat
# Level 2 → /var/cache/content/mysite/.stat
# Level 3 → /var/cache/content/mysite/en/.stat (most granular)

# Higher = more granular invalidation = better cache hit ratio
# Recommended: set to match your content hierarchy depth
```

---

## 🌐 URL Rewriting in Dispatcher

```apache
# /etc/httpd/conf.d/rewrites/mysite-rewrite.rules

# Remove /content/mysite from URLs
RewriteRule ^/content/mysite(/.*)?$ $1 [R=301,L]

# Rewrite short URLs to AEM content paths
RewriteRule ^/about$ /content/mysite/en/about.html [PT,L]
RewriteRule ^/products$ /content/mysite/en/products.html [PT,L]

# Redirect old URLs
RewriteRule ^/old-page$ /content/mysite/en/new-page.html [R=301,L]
```

---

## ☁️ Dispatcher in AEM Cloud Service

| Aspect | AEM 6.5 | AEM Cloud Service |
|--------|---------|-----------------|
| **Dispatcher setup** | Self-managed Apache httpd | Adobe-managed (Fastly CDN) |
| **Config location** | Server filesystem | `dispatcher/` module in Git repo |
| **Deploy method** | Manual file update | Cloud Manager pipeline |
| **CDN** | Separate (Akamai, etc.) | Fastly built-in |
| **Config format** | Apache + Dispatcher `.any` files | Same format, deployed via pipeline |
| **Cache invalidation** | Replication agent to Dispatcher | Same mechanism |

### AEM Cloud Dispatcher Config in Git

```
dispatcher/
  └── src/
        ├── conf.d/
        │     ├── available_vhosts/
        │     │     └── mysite.vhost
        │     ├── enabled_vhosts/
        │     └── rewrites/
        └── conf.dispatcher.d/
              ├── available_farms/
              │     └── default.farm
              ├── enabled_farms/
              ├── cache/
              ├── filters/
              └── renders/
```

Deploy via Cloud Manager **Web Tier Pipeline** (Dispatcher-only changes don't need full stack deployment).

---

## 🐛 Debugging Dispatcher Issues

### Check if Dispatcher is Caching

```bash
# Check the cache directory for the page
ls -la /var/www/vhost/mysite/cache/content/mysite/en/

# Check .stat file timestamps
find /var/www/vhost/mysite/cache/ -name ".stat" -exec ls -la {} \;

# Test request bypassing cache (add debug header)
curl -H "X-Dispatcher-Info: true" http://www.mysite.com/content/mysite/en/home.html -I
```

### Dispatcher Log

```bash
# Default log location
tail -f /var/log/httpd/dispatcher.log

# Enable debug logging in dispatcher.any
/loglevel "3"   # 0=error, 1=warn, 2=info, 3=debug, 4=trace

# Common log entries
# DISP_LOG_LEVEL 3
# cache file hit → serving from cache
# cache file miss → forwarding to AEM
# cache invalidation → clearing cache file
```

### Check Filter Blocking

```bash
# Test if a URL is being blocked by Dispatcher filter
curl -I http://www.mysite.com/system/console
# Should return 403 Forbidden (blocked by filter)

curl -I http://www.mysite.com/content/mysite/en/home.html
# Should return 200 OK
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is the Dispatcher and what are its three main roles?**

> **Answer:** The Dispatcher is an Apache httpd module (not a standalone Java app) that sits in front of AEM Publish instances. Its three roles:
> 1. **Caching:** Stores rendered HTML as static files on disk and serves them without touching AEM — dramatically reducing server load and improving response times
> 2. **Load Balancing:** Distributes incoming requests across multiple Publish instances for horizontal scaling
> 3. **Security:** Acts as a WAF (Web Application Firewall) — blocks requests to sensitive paths (`/system/console`, `/crx/de`, `/libs/granite/`) before they reach AEM

**Q2. How does Dispatcher cache invalidation work?**

> **Answer:** When an author publishes a page, the Replication Agent sends the content to Publish AND sends an HTTP DELETE request to the Dispatcher's `/dispatcher/invalidate.cache` endpoint with the CQ-Handle header containing the content path. The Dispatcher doesn't delete the cache file immediately — it updates the `.stat` file in the corresponding directory. On the next request, Dispatcher compares the page file's timestamp to the `.stat` file timestamp. If the page file is older (stale), Dispatcher fetches fresh content from AEM and updates the cache.

**Q3. What does `statfileslevel` control and why does it matter?**

> **Answer:** `statfileslevel` controls at which directory depth the `.stat` file is placed when invalidating cache. Level 0 = root (invalidates EVERYTHING). Level 3 = three directories deep (more granular). Higher levels mean fewer cache files are invalidated per publish event — better cache hit ratio. Set it to match your content hierarchy. For `/content/mysite/en/home.html`, level 3 places `.stat` at `content/mysite/en/` — only pages in the `en/` subtree are invalidated, not pages in other languages.

**Q4. What does the Dispatcher filter section do and what is the recommended approach?**

> **Answer:** The filter section controls which HTTP requests the Dispatcher forwards to AEM. Best practice is **default-deny**: start with `/0001 { /type "deny" /glob "*" }` to block everything, then explicitly allow only what should be public. This prevents AEM admin interfaces from being accidentally exposed. Critical paths to always block: `/system/console`, `/crx/de`, `/libs/granite/`, `/bin/wcm/`, direct `.json` servlet access to sensitive selectors.

**Q5. Why doesn't the Dispatcher cache requests with cookies or query strings by default?**

> **Answer:** Cookies typically indicate personalized or authenticated sessions — the same URL may return different content for different users, so caching would serve the wrong content. Query strings (e.g., `?search=laptop`) often produce unique results — caching all combinations would explode cache storage. Use `ignoreUrlParams` to whitelist tracking params (utm_*, fbclid) that don't affect content — these can then be safely cached as if the URL had no params.

**Q6. How do you configure Dispatcher in AEM Cloud Service?**

> **Answer:** In AEM Cloud, Dispatcher configuration lives in the `dispatcher/` Maven module of your project (in Git). It follows the same Apache + Dispatcher `.any` file format. Deploy changes via Cloud Manager's **Web Tier Pipeline** — changes only to Dispatcher/CDN config don't require a full application rebuild. Adobe manages the actual infrastructure; you just manage the rules.

---

## ✅ Best Practices

1. **Default-deny filter strategy** — block all, allow only what's needed
2. **Never expose `/system/console`, `/crx/de`, `/bin/wcm`** to the internet
3. **Cache HTML, JS, CSS, images** — these should always be cached
4. **Never cache authenticated content** — add cookie-based exclusions
5. **Use `statfileslevel` matching your URL depth** — more granular = better cache hit ratio
6. **Set `ignoreUrlParams` for tracking params** — don't bust cache on utm_source
7. **Test every URL pattern with Dispatcher** — use curl to verify filter behavior
8. **In AEM Cloud:** Keep Dispatcher configs in Git and deploy via Cloud Manager

---

## 🛠️ Hands-on Practice

1. **Security audit:** List 10 paths that MUST be blocked by Dispatcher filters. Verify each is blocked with curl.
2. **Cache rule exercise:** Write filter rules to cache `.html`, `.json` (only for specific selectors), fonts, and images — but NOT authenticated pages.
3. **Cache invalidation trace:** Publish a page on Author, tail the Dispatcher log, and trace the full cache invalidation flow.
4. **statfileslevel experiment:** Change statfileslevel from 3 to 0 — observe the broader invalidation impact.
5. **URL rewrite:** Write Apache RewriteRules to map `/products` → `/content/mysite/en/products.html` without a redirect.
