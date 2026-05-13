# AEM SPA Editor — React & Angular Integration
**Target:** 2–4 YOE | **Versions:** AEM 6.5 & Cloud Service

---

## 🌟 What is the AEM SPA Editor?

The **AEM SPA Editor** bridges the gap between Single-Page Applications (React/Angular) and AEM's visual page authoring. It allows authors to edit content in a React or Angular SPA using AEM's standard drag-and-drop authoring UI — while developers build modern JavaScript frontend apps.

### Traditional AEM vs SPA Editor

| Aspect | Traditional AEM | SPA Editor |
|--------|----------------|-----------|
| **Frontend** | HTL + Server-side rendering | React or Angular |
| **Rendering** | AEM server renders HTML | Browser renders JS app |
| **Author experience** | Rich drag-and-drop | Same drag-and-drop |
| **Data format** | HTML | JSON (Sling Model Exporter) |
| **URL routing** | AEM page structure | React Router / Angular Router |
| **Use case** | Content-heavy sites | App-like experiences |

---

## 🏗️ SPA Editor Architecture

```
Author Environment:
─────────────────────────────────────────────────────────
  [AEM Page Editor iframe]
       │  loads
       ▼
  [React/Angular SPA]
       │  Fetches page model
       ▼
  [AEM Page Model API]    →  /content/mysite/en/home.model.json
       │  Returns JSON tree
       ▼
  [AEM SPA SDK (ModelManager)]
       │  Maps JSON to components
       ▼
  [React Components]  ←── [ComponentMapping.map('resourceType', ReactComp)]
       │  Author edits React components visually in iframe
       ▼
  [AEM Editor overlays drag handles, edit bars on top of SPA]

Publish Environment:
─────────────────────────────────────────────────────────
  User visits URL → AEM returns page.model.json → React app renders → Hydrated SPA
```

---

## 📦 Required SDK Libraries

```json
// package.json
{
  "dependencies": {
    // AEM SPA React SDK
    "@adobe/aem-react-editable-components": "^2.x",
    "@adobe/aem-spa-component-mapping": "^1.x",
    "@adobe/aem-spa-page-model-manager": "^1.x",

    // React
    "react": "^18.x",
    "react-dom": "^18.x",
    "react-router-dom": "^6.x"
  }
}
```

---

## 💻 Building SPA Components — React

### Step 1: AEM Component Structure

```
ui.apps/src/main/content/jcr_root/apps/mysite/components/content/hero/
  ├── .content.xml                          ← Component definition
  │     sling:resourceType = "mysite/components/content/hero"
  ├── _cq_dialog/.content.xml               ← Author dialog (same as normal AEM)
  └── (NO hero.html — React renders this!)
```

### Step 2: Sling Model with JSON Exporter

```java
package com.mysite.core.models;

import com.adobe.cq.export.json.ComponentExporter;
import com.adobe.cq.export.json.ExporterConstants;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.models.annotations.*;
import org.apache.sling.models.annotations.injectorspecific.ValueMapValue;

@Model(
    adaptables   = Resource.class,
    adapters     = {HeroModel.class, ComponentExporter.class},   // ← ComponentExporter is key
    resourceType = "mysite/components/content/hero",
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
@Exporter(
    name       = ExporterConstants.SLING_MODEL_EXPORTER_NAME,   // "jackson"
    extensions = ExporterConstants.SLING_MODEL_EXTENSION         // "model"
)
public class HeroModel implements ComponentExporter {

    @ValueMapValue private String title;
    @ValueMapValue private String description;
    @ValueMapValue private String ctaLabel;
    @ValueMapValue private String ctaLink;
    @ValueMapValue private String backgroundImage;

    // Getters — all public fields are serialized to JSON
    public String getTitle()           { return title; }
    public String getDescription()     { return description; }
    public String getCtaLabel()        { return ctaLabel; }
    public String getCtaLink()         { return ctaLink; }
    public String getBackgroundImage() { return backgroundImage; }

    // REQUIRED by ComponentExporter — tells SPA which React component to use
    @Override
    public String getExportedType() {
        return "mysite/components/content/hero";  // Must match component resourceType
    }
}
```

**Test the JSON output:**
```
GET /content/mysite/en/home/jcr:content/root/hero.model.json

Response:
{
  "title": "Welcome to My Site",
  "description": "Discover amazing products",
  "ctaLabel": "Shop Now",
  "ctaLink": "/content/mysite/en/products.html",
  "backgroundImage": "/content/dam/mysite/hero.jpg",
  ":type": "mysite/components/content/hero"   ← Used by ComponentMapping
}
```

### Step 3: React Component with AEM Editing Support

```jsx
// src/components/Hero/Hero.jsx
import React from 'react';
import { MapTo } from '@adobe/aem-react-editable-components';

// Define edit config — tells AEM Editor when to show placeholder
const HeroEditConfig = {
    emptyLabel: 'Hero Component',
    isEmpty: (props) => !props.title,  // Show placeholder when no title set
    resourceType: 'mysite/components/content/hero'
};

// The actual React component — receives props from page model JSON
const Hero = (props) => {
    const {
        title,
        description,
        ctaLabel,
        ctaLink,
        backgroundImage
    } = props;

    if (!title) {
        return <div className="hero-placeholder">Configure the Hero component</div>;
    }

    return (
        <section
            className="hero"
            style={{ backgroundImage: `url(${backgroundImage})` }}
        >
            <div className="hero__content">
                <h1 className="hero__title">{title}</h1>
                {description && (
                    <p className="hero__description">{description}</p>
                )}
                {ctaLabel && ctaLink && (
                    <a className="hero__cta btn btn--primary" href={ctaLink}>
                        {ctaLabel}
                    </a>
                )}
            </div>
        </section>
    );
};

// MAP the React component to the AEM resourceType
// When AEM sends a component with :type="mysite/components/content/hero",
// this React component renders it
export default MapTo('mysite/components/content/hero')(Hero, HeroEditConfig);
```

### Step 4: App Entry Point with ModelManager

```jsx
// src/index.js
import React from 'react';
import ReactDOM from 'react-dom';
import { ModelManager, Constants } from '@adobe/aem-spa-page-model-manager';
import App from './App';

// Initialize ModelManager — fetches the page's .model.json
ModelManager.initialize().then((model) => {
    ReactDOM.render(
        <App model={model} cqChildren={model[Constants.CHILDREN_PROP]}
             cqItems={model[Constants.ITEMS_PROP]}
             cqItemsOrder={model[Constants.ITEMS_ORDER_PROP]}
             cqPath={ModelManager.rootPath}
             locationPathname={window.location.pathname}
        />,
        document.getElementById('root')
    );
});
```

```jsx
// src/App.jsx
import React from 'react';
import { Page, withModel, EditorContext, Utils } from '@adobe/aem-react-editable-components';
import { BrowserRouter as Router } from 'react-router-dom';

// Import all component mappings (this causes MapTo calls to execute)
import './components/Hero/Hero';
import './components/Text/Text';
import './components/Image/Image';
import './components/Navigation/Navigation';

const App = ({ model }) => {
    const isInEditor = Utils.isInEditor();

    return (
        <Router>
            <EditorContext.Provider value={isInEditor}>
                <div className="App">
                    <Page
                        cqChildren={model[':children']}
                        cqItems={model[':items']}
                        cqItemsOrder={model[':itemsOrder']}
                        cqPath="/"
                        locationPathname={window.location.pathname}
                    />
                </div>
            </EditorContext.Provider>
        </Router>
    );
};

export default withModel(App);
```

---

## 🔄 Page Model JSON Structure

The complete page structure is served as JSON:

```json
// GET /content/mysite/en/home.model.json
{
  ":type": "mysite/components/page/homepage",
  ":path": "/content/mysite/en/home",
  ":children": {
    "/content/mysite/en/about": { ... },
    "/content/mysite/en/products": { ... }
  },
  ":items": {
    "root": {
      ":type": "wcm/foundation/components/responsivegrid",
      ":items": {
        "hero": {
          ":type": "mysite/components/content/hero",
          "title": "Welcome to My Site",
          "description": "Discover amazing products",
          "ctaLabel": "Shop Now",
          "ctaLink": "/content/mysite/en/products.html"
        },
        "text": {
          ":type": "mysite/components/content/text",
          "text": "<p>Body content here</p>",
          "textIsRich": true
        }
      }
    }
  }
}
```

---

## 🌐 Routing in SPA Editor

The SPA Router must sync with AEM's page structure:

```jsx
// src/App.jsx — Route-based rendering
import { Route, Switch, useHistory } from 'react-router-dom';
import { ModelManager } from '@adobe/aem-spa-page-model-manager';

function App() {
    const history = useHistory();

    // Listen for AEM Editor navigation (when author clicks between pages)
    useEffect(() => {
        return ModelManager.addListener(':path', (pagePath) => {
            history.push(pagePath.replace('/content/mysite', ''));
        });
    }, []);

    return (
        <Switch>
            <Route
                path="*"
                render={(routeProps) => (
                    <Page
                        {...routeProps}
                        cqPath={`/content/mysite${routeProps.location.pathname}`}
                    />
                )}
            />
        </Switch>
    );
}
```

---

## ☁️ Remote SPA (AEM Cloud)

**Remote SPA** is an evolution where the React app lives on an external server (Vercel, Netlify, etc.) — AEM only provides content via JSON APIs.

```
Traditional SPA Editor: React app built inside AEM Maven project → served by AEM
Remote SPA Editor:      React app lives externally (e.g., Vercel) → AEM is pure CMS
```

```jsx
// Remote SPA Setup — minimal AEM integration
import { AEMHeadless } from '@adobe/aem-headless-client-js';

const aemHeadless = new AEMHeadless({
    serviceURL: 'https://publish-p12345-e67890.adobeaemcloud.com',
    endpoint: 'content/graphql/global/endpoint.gql',
    auth: ['apiKey', 'your-api-key']  // For author preview
});

// Fetch content via GraphQL
const { data } = await aemHeadless.runPersistedQuery('mysite/hero-content', {
    pagePath: '/content/mysite/en/home'
});
```

---

## ❓ Interview Questions & Detailed Answers

**Q1. What is the AEM SPA Editor and how does it work?**

> **Answer:** The AEM SPA Editor allows authors to edit React/Angular SPA components using AEM's visual authoring UI. The SPA fetches a `.model.json` representation of the page from AEM (generated by Sling Model Exporters). The SDK's `ModelManager` provides this JSON to the app. `ComponentMapping.map()` registers which React component renders which AEM `resourceType`. The AEM Editor renders an iframe overlay with drag handles and edit bars on top of the SPA. Changes are saved to JCR as normal AEM component data.

**Q2. What is `ComponentExporter` and why is it needed?**

> **Answer:** `ComponentExporter` is a Sling interface with one method: `getExportedType()` which returns the component's `resourceType`. Implementing it (combined with `@Exporter(name="jackson")`) enables Sling Model Exporter to serialize the model as JSON at `<resource>.model.json`. This JSON is what the React app reads. Without it, the component won't appear in the page model JSON.

**Q3. What does `MapTo()` do in a React SPA component?**

> **Answer:** `MapTo('resourceType')` registers a React component as the renderer for a specific AEM resourceType in the SPA SDK's component map. When the `ModelManager` provides the page JSON and encounters `":type": "mysite/components/hero"`, it looks up the component map and renders the registered React component, passing all the JSON properties as React props.

**Q4. What is the difference between AEM SPA Editor and Headless AEM?**

> **Answer:** SPA Editor: React app is tightly integrated with AEM authoring — authors edit content visually in the AEM Editor. The React app must use the AEM SPA SDK. Headless AEM: React/Vue/Next.js app is completely decoupled — fetches content via GraphQL or REST APIs, no AEM SDK required, no visual editing in AEM. Use SPA Editor when marketers need visual authoring. Use headless when the frontend team wants full independence.

---

## ✅ Best Practices

1. **Use `ComponentExporter` interface** on all Sling Models that the SPA will consume
2. **Keep React components pure** — they receive data as props, no AEM API calls inside components
3. **Implement `HeroEditConfig.isEmpty()`** — gives authors a helpful placeholder in empty components
4. **Use persisted GraphQL queries** for headless content — not inline queries
5. **Test `.model.json` directly** before writing React code — ensure all needed data is exported
6. **Version your SPA SDK dependencies** — AEM SDK updates can break the editor integration

---

## 🛠️ Hands-on Practice

1. Create an AEM component with `@Exporter(name="jackson")` and test at `<resource>.model.json`
2. Create a minimal React SPA with `ModelManager.initialize()` and `MapTo()` for one component
3. Test the SPA in the AEM Editor — verify the edit overlay appears on the React component
4. Add a new author dialog field to your AEM component and verify it appears in the `.model.json`
5. (Advanced) Set up a Remote SPA with AEM Cloud as the content source via GraphQL
