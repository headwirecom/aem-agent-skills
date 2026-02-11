# Global Objects and Use-API

## Global Bindings

These objects are available in all HTL scripts and Use-API objects (via `javax.script.Bindings`
or request attributes):

| Object | Type | Description |
|---|---|---|
| `properties` | `ValueMap` | Properties of the current resource |
| `resource` | `Resource` | The current Sling resource |
| `request` | `SlingHttpServletRequest` | The current request |
| `response` | `SlingHttpServletResponse` | The current response |
| `resolver` | `ResourceResolver` | The resource resolver from the request |
| `currentNode` | `javax.jcr.Node` | JCR node of the current resource |
| `currentSession` | `javax.jcr.Session` | JCR session |
| `log` | `org.slf4j.Logger` | Logger |
| `out` | `java.io.PrintWriter` | Output writer |
| `reader` | `java.io.BufferedReader` | Input reader |
| `sling` | `SlingScriptHelper` | Access to OSGi services via `sling.getService()` |

### AEM-specific Global Objects

These are additionally available in AEM (not bare Sling):

| Object | Type | Description |
|---|---|---|
| `currentPage` | `com.day.cq.wcm.api.Page` | The current page |
| `pageProperties` | `ValueMap` | Properties of the current page's `jcr:content` |
| `wcmmode` | object | WCM mode with boolean properties: `.edit`, `.design`, `.preview`, `.disabled` |
| `currentDesign` | `Design` | Design of the current page |
| `currentStyle` | `Style` | Style of the current cell/component |
| `component` | `Component` | The current component |
| `pageManager` | `PageManager` | Page management API |
| `designer` | `Designer` | Design management API |
| `editContext` | `EditContext` | Edit context for the component |
| `resourceDesign` | `Design` | Design of the resource's page |
| `resourcePage` | `Page` | Page containing the current resource |
| `xssAPI` | `XSSAPI` | XSS protection utilities |

## Use-API

### Use Provider Priority (highest to lowest)

| Priority | Provider | Purpose |
|---|---|---|
| 100 | RenderUnitProvider | Load HTL templates via `data-sly-use` |
| 90 | JavaUseProvider | Sling Models, OSGi services, POJOs (Use interface or adaptable) |
| 80 | JsUseProvider | JavaScript `use()` function (deprecated for AEMaaCS) |
| 0 | ScriptUseProvider | Objects from other script engines |
| -10 | ResourceUseProvider | Load Resources by path |

### Sling Models (Preferred)

```java
@Model(adaptables = SlingHttpServletRequest.class,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class MyModel {

    @ValueMapValue
    private String title;

    @ChildResource
    private Resource image;

    @Self
    private SlingHttpServletRequest request;

    @ScriptVariable
    private Page currentPage;

    public String getTitle() {
        return title != null ? title : currentPage.getTitle();
    }
}
```

```html
<sly data-sly-use.model="com.example.core.models.MyModel"/>
<h1>${model.title}</h1>
```

### Passing Parameters to Sling Models

Parameters from HTL become request attributes, injectable with `@Inject`:

```html
<sly data-sly-use.model="${'com.example.Model' @ colour='red', year=2024}"/>
```

```java
@Model(adaptables = SlingHttpServletRequest.class)
public class Model {
    @Inject private String colour;
    @Inject private int year;
}
```

### POJO Use-API (Use interface)

For classes not registered as Sling Models:

```java
public class MyHelper implements org.apache.sling.scripting.sightly.pojo.Use {
    private String result;

    @Override
    public void init(Bindings bindings) {
        Resource resource = (Resource) bindings.get("resource");
        result = resource.getPath();
    }

    public String getResult() { return result; }
}
```

### Resource-backed Java Classes

Java files co-located with HTL scripts:

```
apps/myproject/components/page/
├── PageHelper.java
└── page.html
```

```html
<!--/* By simple class name (slower, but supports overlay) */-->
<sly data-sly-use.helper="PageHelper"/>

<!--/* By FQCN (faster, no overlay support) */-->
<sly data-sly-use.helper="apps.myproject.components.page.PageHelper"/>
```

Package name = resource path with invalid Java chars replaced by `_`.

### JavaScript Use-API (Deprecated for AEMaaCS)

```javascript
use(function() {
    var title = this.title || '';  // parameters available on 'this'
    return {
        formattedTitle: title.toUpperCase()
    };
});
```

```html
<sly data-sly-use.logic="${'logic.js' @ title=properties.jcr:title}"/>
<h1>${logic.formattedTitle}</h1>
```

**Caveat:** Server-side JS runs in Rhino. Strict equality (`===`) between Java and JS
objects returns `false` even for same values. Use `==` instead.

## Property Resolution Order

When accessing `object.identifier`:
1. Public field named `identifier`
2. Method named `identifier()` (no args)
3. Getter `getIdentifier()`
4. Boolean getter `isIdentifier()`
5. Returns `null`

## AEM-specific Extensions

### i18n

```html
${'Menu' @ i18n}
${'Menu' @ i18n, locale='fr', hint='Main navigation'}
${'Menu' @ i18n, basename='myapp'}  <!--/* Sling extension: resource bundle basename */-->
```

Backed by `com.day.cq.i18n` in AEM.

### data-sly-resource with Maps

AEM extends `data-sly-resource` to accept `Map` or `Record` objects:

```html
<!--/*
  map = {
    resourceName: "myText",
    "sling:resourceType": "core/wcm/components/text/v2/text",
    "text": "Hello World!"
  }
*/-->
<div data-sly-resource="${map}"></div>
```

Creates a `SyntheticResource` with the map properties. Requires `resourceName`. Falls back to
`resourceType` option or current resource type if `sling:resourceType` is missing.

### WCM Mode in data-sly-include

```html
<sly data-sly-include="${'header.html' @ wcmmode='disabled'}"/>
```

Controls the WCM mode for the included script. Values match `WCMMode` enum constants.
