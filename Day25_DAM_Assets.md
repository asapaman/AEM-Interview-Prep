# DAM & Asset Management in AEM
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What is AEM DAM?

**AEM DAM (Digital Asset Management)** is AEM's built-in system for storing, organizing, processing, and delivering digital assets — images, videos, PDFs, documents, and more. It's built on JCR and stored under `/content/dam/`.

### Key DAM Concepts

| Concept | Description |
|---------|-------------|
| **Asset** | A file (image, PDF, video) stored in the DAM |
| **Rendition** | A processed version of an asset (thumbnail, web-optimized) |
| **Metadata** | Properties describing the asset (title, tags, author, license) |
| **Asset Workflow** | Automated processing pipeline on asset upload |
| **Smart Tags** | AI-powered automatic tagging (Adobe Sensei) |
| **Dynamic Media** | Cloud-based advanced image/video delivery |
| **Asset Compute** | Serverless asset processing (AEM Cloud) |

---

## 📁 Asset Structure in JCR

```
/content/dam/mysite/products/
  └── laptop-hero.jpg               (dam:Asset)
        └── jcr:content             (dam:AssetContent)
              ├── jcr:title = "Laptop Pro 2024 Hero Image"
              ├── jcr:description = "Hero image for product page"
              ├── cq:tags = ["mysite:products/electronics"]
              ├── metadata           (nt:unstructured)
              │     ├── dc:title = "Laptop Pro 2024"
              │     ├── dc:description = "..."
              │     ├── dc:creator = "John Smith"
              │     ├── dam:assetCreatedDate = ...
              │     ├── dam:assetLastModified = ...
              │     ├── dam:MIMEtype = "image/jpeg"
              │     ├── dam:size = "2048576"     ← bytes
              │     ├── tiff:ImageWidth = "1920"
              │     └── tiff:ImageLength = "1080"
              │
              └── renditions        (nt:folder)
                    ├── original/   (nt:file) ← The original uploaded file
                    │     └── jcr:content (nt:resource)
                    │           └── jcr:data = [binary]
                    ├── cq5dam.thumbnail.48.48.png    (nt:file)
                    ├── cq5dam.thumbnail.140.100.png  (nt:file)
                    ├── cq5dam.thumbnail.319.319.png  (nt:file)
                    ├── cq5dam.web.1280.1280.jpeg     (nt:file) ← Web rendition
                    └── cq5dam.zoom.2048.2048.jpeg    (nt:file) ← Zoom rendition
```

---

## 💻 Working with Assets in Java

### Reading Asset Properties

```java
package com.mysite.core.models;

import com.day.cq.dam.api.Asset;
import com.day.cq.dam.api.DamConstants;
import com.day.cq.dam.api.Rendition;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.api.resource.ResourceResolver;
import org.apache.sling.api.resource.ValueMap;
import org.apache.sling.models.annotations.*;
import org.apache.sling.models.annotations.injectorspecific.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.annotation.PostConstruct;

@Model(adaptables = Resource.class, defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class AssetInfoModel {

    private static final Logger LOG = LoggerFactory.getLogger(AssetInfoModel.class);

    // Path to the DAM asset (from component dialog fileReference field)
    @ValueMapValue(name = "fileReference")
    private String fileReference;

    @SlingObject
    private ResourceResolver resourceResolver;

    // Populated from the asset
    private String title;
    private String description;
    private String mimeType;
    private long fileSize;
    private int width;
    private int height;
    private String altText;
    private String webRenditionPath;

    @PostConstruct
    protected void init() {
        if (fileReference == null || fileReference.isEmpty()) return;

        // Resolve the asset resource
        Resource assetResource = resourceResolver.getResource(fileReference);
        if (assetResource == null) {
            LOG.warn("Asset not found at path: {}", fileReference);
            return;
        }

        // Adapt to Asset (DAM API)
        Asset asset = assetResource.adaptTo(Asset.class);
        if (asset == null) {
            LOG.warn("Resource at '{}' is not a DAM Asset", fileReference);
            return;
        }

        // Read basic asset properties
        title       = asset.getMetadataValue(DamConstants.DC_TITLE);
        description = asset.getMetadataValue(DamConstants.DC_DESCRIPTION);
        mimeType    = asset.getMimeType();

        // Read metadata map for technical properties
        ValueMap metadata = assetResource.getChild("jcr:content/metadata")
                                         .adaptTo(ValueMap.class);
        if (metadata != null) {
            fileSize = metadata.get("dam:size",    Long.class)  != null
                       ? metadata.get("dam:size",  0L) : 0L;
            width    = metadata.get("tiff:ImageWidth",  0);
            height   = metadata.get("tiff:ImageLength", 0);

            // Alt text is often stored in dc:description or a custom field
            altText = metadata.get("dc:description", String.class);
        }

        // Get a specific rendition
        Rendition webRendition = asset.getRendition("cq5dam.web.1280.1280.jpeg");
        if (webRendition != null) {
            webRenditionPath = webRendition.getPath();
        } else {
            // Fallback to original
            webRenditionPath = fileReference;
        }

        // Auto-generate alt text from title if not set
        if (altText == null && title != null) {
            altText = title;
        }
    }

    public String getTitle()          { return title; }
    public String getDescription()    { return description; }
    public String getMimeType()       { return mimeType; }
    public long getFileSize()         { return fileSize; }
    public int getWidth()             { return width; }
    public int getHeight()            { return height; }
    public String getAltText()        { return altText; }
    public String getWebRenditionPath() { return webRenditionPath; }
    public String getFileReference()  { return fileReference; }
    public boolean isImage() {
        return mimeType != null && mimeType.startsWith("image/");
    }
}
```

---

### Reading All Renditions of an Asset

```java
import com.day.cq.dam.api.Asset;
import com.day.cq.dam.api.Rendition;
import java.util.*;

public List<Map<String, Object>> getRenditions(String assetPath) {
    Resource assetResource = resourceResolver.getResource(assetPath);
    if (assetResource == null) return Collections.emptyList();

    Asset asset = assetResource.adaptTo(Asset.class);
    if (asset == null) return Collections.emptyList();

    List<Map<String, Object>> renditions = new ArrayList<>();

    for (Rendition rendition : asset.getRenditions()) {
        Map<String, Object> info = new HashMap<>();
        info.put("name",     rendition.getName());
        info.put("path",     rendition.getPath());
        info.put("mimeType", rendition.getMimeType());
        info.put("size",     rendition.getSize());
        renditions.add(info);
    }

    return renditions;
    // Returns: [
    //   {name: "original", path: "/.../renditions/original", mimeType: "image/jpeg", size: 2048576}
    //   {name: "cq5dam.thumbnail.48.48.png", path: "/.../renditions/cq5dam.thumbnail.48.48.png", ...}
    //   {name: "cq5dam.web.1280.1280.jpeg", ...}
    // ]
}
```

---

### Creating Assets Programmatically

```java
import com.day.cq.dam.api.AssetManager;

// Service user must have write access to /content/dam
public Asset createAsset(String damPath, InputStream imageStream, String mimeType) {
    AssetManager assetManager = resourceResolver.adaptTo(AssetManager.class);
    if (assetManager == null) {
        LOG.error("Could not get AssetManager — check service user permissions");
        return null;
    }

    try {
        // Create or update asset
        Asset asset = assetManager.createAsset(
            damPath,       // e.g., "/content/dam/mysite/products/new-image.jpg"
            imageStream,   // InputStream of the file content
            mimeType,      // "image/jpeg"
            true           // autoSave: true = commit immediately
        );

        LOG.info("Asset created at: {}", asset.getPath());

        // Set metadata
        Resource assetResource = resourceResolver.getResource(asset.getPath());
        Resource metadataResource = assetResource.getChild("jcr:content/metadata");
        if (metadataResource != null) {
            ModifiableValueMap mvm = metadataResource.adaptTo(ModifiableValueMap.class);
            if (mvm != null) {
                mvm.put(DamConstants.DC_TITLE, "New Product Image");
                mvm.put(DamConstants.DC_DESCRIPTION, "Auto-imported from product feed");
                resourceResolver.commit();
            }
        }

        return asset;
    } catch (Exception e) {
        LOG.error("Failed to create asset at {}", damPath, e);
        return null;
    }
}
```

---

## 🔄 Asset Processing Workflows

When an asset is uploaded to DAM, AEM triggers the **DAM Update Asset** workflow — a chain of processing steps:

```
UPLOAD ──► DAM Update Asset Workflow
              │
              ├── Extract Metadata (EXIF, IPTC, XMP)
              ├── Create Thumbnails (48x48, 140x100, 319x319)
              ├── Create Web Renditions (1280x1280 JPEG)
              ├── Create Zoom Rendition (2048x2048 JPEG)
              ├── Apply Smart Tags (if Sensei configured)
              ├── PDF Rasterization (for PDF assets)
              ├── Video Encoding (for video assets)
              └── COMPLETE
```

### Custom Asset Workflow Step

```java
package com.mysite.core.workflow;

import com.adobe.granite.workflow.WorkflowSession;
import com.adobe.granite.workflow.exec.WorkItem;
import com.adobe.granite.workflow.exec.WorkflowProcess;
import com.adobe.granite.workflow.metadata.MetaDataMap;
import com.day.cq.dam.api.Asset;
import org.apache.sling.api.resource.ResourceResolver;

import org.osgi.service.component.annotations.Component;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Component(
    service = WorkflowProcess.class,
    property = {"process.label=My Site - Add Watermark to Image"}
)
public class WatermarkWorkflowStep implements WorkflowProcess {

    private static final Logger LOG = LoggerFactory.getLogger(WatermarkWorkflowStep.class);

    @Override
    public void execute(WorkItem item, WorkflowSession session, MetaDataMap args)
            throws Exception {

        String payloadPath = item.getWorkflowData().getPayload().toString();
        LOG.info("Processing asset: {}", payloadPath);

        ResourceResolver resolver = session.adaptTo(ResourceResolver.class);
        if (resolver == null) return;

        Asset asset = resolver.getResource(payloadPath)
                              .adaptTo(Asset.class);
        if (asset == null || !asset.getMimeType().startsWith("image/")) {
            LOG.debug("Not an image, skipping watermark: {}", payloadPath);
            return;
        }

        // Add watermark logic here
        // e.g., read original rendition, process with ImageIO, save new rendition

        // Example: add metadata tag
        resolver.getResource(payloadPath + "/jcr:content/metadata")
                .adaptTo(ModifiableValueMap.class)
                .put("mysite:watermarked", true);

        resolver.commit();
        LOG.info("Watermark applied to: {}", payloadPath);
    }
}
```

---

## 🖼️ Image Handling — Adaptive Images

AEM's **Adaptive Image Servlet** and **Core Image Component** handle responsive image delivery:

```html
<!-- Core Image component renders: -->
<img src="/content/dam/mysite/hero.coreimg.82.1600.jpg"
     srcset="/content/dam/mysite/hero.coreimg.80.320.jpg 320w,
             /content/dam/mysite/hero.coreimg.80.480.jpg 480w,
             /content/dam/mysite/hero.coreimg.82.800.jpg 800w,
             /content/dam/mysite/hero.coreimg.82.1280.jpg 1280w,
             /content/dam/mysite/hero.coreimg.82.1600.jpg 1600w"
     sizes="(max-width: 320px) 280px, (max-width: 480px) 440px, 800px"
     alt="Hero Image"
     loading="lazy"/>
```

```
URL structure for adaptive images:
/content/dam/mysite/hero.coreimg.82.1280.jpg
                              ↑   ↑   ↑
                         selector quality width
```

### Dynamic Media (AEM Cloud)

```java
// With Dynamic Media, images are served from CDN with on-the-fly transformations
// No pre-generated renditions needed — all sizes served on demand

// Dynamic Media URL:
// https://s7g10.scene7.com/is/image/mysite/hero?wid=1280&hei=720&qlt=82&fit=crop

// In Java — build Dynamic Media URL
import com.day.cq.dam.scene7.api.S7ConfigService;

String dynamicMediaUrl = scene7Service.getImageUrl(
    "/content/dam/mysite/hero.jpg",
    "?wid=1280&hei=720&qlt=82"
);
```

---

## 📋 Key DAM Constants

```java
import com.day.cq.dam.api.DamConstants;

// Common metadata field names:
DamConstants.DC_TITLE        // "dc:title"
DamConstants.DC_DESCRIPTION  // "dc:description"
DamConstants.DC_CREATOR      // "dc:creator"
DamConstants.DC_SUBJECT      // "dc:subject" (tags)
DamConstants.DC_FORMAT       // "dc:format" (MIME type)
DamConstants.DC_RIGHTS       // "dc:rights" (license)

// DAM-specific
"dam:size"               // File size in bytes
"dam:MIMEtype"           // MIME type
"dam:assetCreatedDate"   // Creation date
"dam:assetLastModified"  // Last modification date

// Technical metadata
"tiff:ImageWidth"        // Image width in pixels
"tiff:ImageLength"       // Image height in pixels
"exif:GPSLatitude"       // GPS latitude (if EXIF present)
"exif:GPSLongitude"      // GPS longitude
```

---

## ☁️ AEM 6.5 vs Cloud — DAM Differences

| Feature | AEM 6.5 | AEM Cloud |
|---------|---------|-----------|
| **Rendition processing** | DAM Update Asset workflow | Asset Compute (serverless) |
| **Video encoding** | On-JVM processing | Dynamic Media or Asset Compute |
| **Smart Tags** | Requires Smart Content Service config | Built-in with Sensei |
| **Dynamic Media** | Add-on | Integrated |
| **Image delivery** | Sling adaptive image servlet | Asset Delivery API |
| **Binary storage** | JCR Blob Store (local filesystem or S3) | Azure Blob Storage (managed) |

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is the JCR structure of a DAM asset?**

> **Answer:** A DAM asset is a `dam:Asset` node. Its `jcr:content` node (of type `dam:AssetContent`) contains `metadata` (technical and descriptive metadata like dc:title, dam:size, tiff:ImageWidth) and `renditions` (a folder containing the original binary and all generated renditions like thumbnails and web renditions). Each rendition is an `nt:file` node with a `jcr:content/jcr:data` binary property.

**Q2. How do you adapt a Resource to an Asset?**

> **Answer:** `Asset asset = resource.adaptTo(Asset.class)`. The `Asset` interface (from `com.day.cq.dam.api`) provides methods like `getMimeType()`, `getMetadataValue()`, `getRenditions()`, `getRendition(name)`, and `getPath()`. Always null-check the result — the adaptation returns null if the resource is not a `dam:Asset` node.

**Q3. What is the DAM Update Asset workflow?**

> **Answer:** It's the built-in workflow that runs automatically when an asset is uploaded to the DAM. It processes the asset through multiple steps: extracting EXIF/IPTC/XMP metadata, generating standard thumbnails (48x48, 140x100, 319x319), creating web renditions (1280x1280 JPEG), applying smart tags (if configured), and encoding video (for video files). Custom workflow steps can be added to this workflow for project-specific processing (watermarking, custom rendition generation, integration with external systems).

**Q4. What is the difference between a rendition and the original asset?**

> **Answer:** The **original** is the unmodified file as uploaded by the user — stored at `renditions/original`. **Renditions** are derived versions generated by workflows: optimized for different use cases (web, thumbnail, print, zoom). They're stored as separate binary nodes under `jcr:content/renditions/`. Renditions are non-destructive — the original is always preserved. You should NEVER modify the original rendition in custom code.

**Q5. How does AEM Cloud handle asset processing differently from AEM 6.5?**

> **Answer:** In AEM 6.5, processing runs in the same JVM as AEM via DAM workflow steps — resource-intensive operations like video encoding can slow down the entire AEM instance. In AEM Cloud, asset processing uses **Asset Compute** — Adobe I/O Runtime (serverless) workers that run outside the AEM JVM. Each processing step (metadata extraction, rendition generation, smart tagging) is an isolated, scalable serverless function. This keeps AEM's JVM free for content serving.

---

## ✅ Best Practices

1. **Always use `asset.getMetadataValue()`** for metadata — don't read raw JCR properties
2. **Check `adaptTo(Asset.class)` for null** — not every resource is a DAM asset
3. **Use service users with DAM read access** for reading assets — not admin sessions
4. **Don't store assets outside `/content/dam/`** — AEM's DAM tools only work under this path
5. **Use renditions for display** — never serve the original to end users (it's often full resolution/size)
6. **Prefer Core Image component** for responsive image delivery over manual `<img>` tags
7. **Set alt text at the component level** — don't rely on DAM metadata alone (authors need control)

---

## 🛠️ Hands-on Practice

1. Upload an image to DAM — inspect its JCR structure in CRXDE (`jcr:content/metadata`, `jcr:content/renditions`)
2. Write a Sling Model that reads an asset's title, description, and file size from DAM metadata
3. List all renditions of an asset and their sizes
4. Write a custom DAM workflow step that adds a `processed=true` metadata property to all newly uploaded images
5. Build a "Media Gallery" component whose dialog accepts multiple asset paths — render each with alt text from metadata
