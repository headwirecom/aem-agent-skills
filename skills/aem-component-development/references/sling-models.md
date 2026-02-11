# Sling Models

Sling Models are the preferred Java Use-API for AEM components. They are annotation-driven
POJOs that adapt from request or resource.

## Basic Model

```java
package com.myproject.core.models;

import org.apache.sling.api.SlingHttpServletRequest;
import org.apache.sling.models.annotations.Model;
import org.apache.sling.models.annotations.DefaultInjectionStrategy;
import org.apache.sling.models.annotations.injectorspecific.ValueMapValue;

@Model(
    adaptables = SlingHttpServletRequest.class,
    adapters = MyComponent.class,
    resourceType = "myproject/components/mycomponent",
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class MyComponentImpl implements MyComponent {

    @ValueMapValue
    private String title;

    @ValueMapValue
    private String description;

    @Override
    public String getTitle() {
        return title;
    }

    @Override
    public String getDescription() {
        return description;
    }

    @Override
    public boolean isEmpty() {
        return title == null && description == null;
    }
}
```

## @Model Annotation Parameters

| Parameter | Purpose |
|---|---|
| `adaptables` | What the model adapts from: `SlingHttpServletRequest.class` or `Resource.class` |
| `adapters` | Interface(s) the model implements — enables `modelFactory.getModelFromResource()` |
| `resourceType` | Ties model to a specific component resource type (for `data-sly-use` auto-resolution) |
| `defaultInjectionStrategy` | `OPTIONAL` = null for missing values. `REQUIRED` = throw if missing |

**Prefer `SlingHttpServletRequest`** as adaptable — it gives access to request attributes,
selectors, WCM mode, and anything injected via `@ScriptVariable`.

## Injection Annotations

### Property Injection

```java
@ValueMapValue                          // from resource's ValueMap
private String title;

@ValueMapValue(name = "jcr:title")      // explicit property name
private String jcrTitle;

@ValueMapValue
@Default(values = "Untitled")           // default value
private String heading;
```

### Resource Injection

```java
@ChildResource                          // child resource as Resource object
private Resource image;

@ChildResource(name = "items")          // explicit child name
private List<Resource> items;

@ResourcePath(path = "/content/dam")    // absolute path
private Resource damRoot;
```

### Request / Script Injection

```java
@Self                                   // the adaptable itself
private SlingHttpServletRequest request;

@SlingObject                            // Sling API objects
private ResourceResolver resourceResolver;

@SlingObject
private Resource resource;

@ScriptVariable                         // HTL script bindings
private Page currentPage;

@ScriptVariable
private Style currentStyle;

@ScriptVariable
private Designer designer;

@ScriptVariable
private com.day.cq.wcm.api.components.Component component;
```

### OSGi Service Injection

```java
@OSGiService
private ModelFactory modelFactory;

@OSGiService
private PageManagerFactory pageManagerFactory;

@OSGiService(filter = "(component.name=com.myproject.MyServiceImpl)")
private MyService myService;
```

### Request Attribute Injection (from HTL parameters)

```java
// HTL: <sly data-sly-use.model="${'com.myproject.Model' @ colour='red'}"/>
@RequestAttribute
private String colour;
```

## Interface + Impl Pattern

Preferred pattern for testability and Sling Model Exporter:

```java
// Interface (in models package)
public interface HeroComponent {
    String getTitle();
    String getImagePath();
    boolean isEmpty();
}

// Implementation
@Model(adaptables = SlingHttpServletRequest.class,
       adapters = {HeroComponent.class, ComponentExporter.class},
       resourceType = HeroComponentImpl.RESOURCE_TYPE,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
@Exporter(name = ExporterConstants.SLING_MODEL_EXPORTER_NAME,
          extensions = ExporterConstants.SLING_MODEL_EXTENSION)
public class HeroComponentImpl implements HeroComponent, ComponentExporter {

    static final String RESOURCE_TYPE = "myproject/components/hero";

    @ValueMapValue
    private String title;

    @ValueMapValue(name = "fileReference")
    private String imagePath;

    @Override
    public String getTitle() { return title; }

    @Override
    public String getImagePath() { return imagePath; }

    @Override
    public boolean isEmpty() {
        return title == null && imagePath == null;
    }

    @Override
    public String getExportedType() { return RESOURCE_TYPE; }
}
```

## @PostConstruct

Run initialization logic after injection:

```java
@PostConstruct
private void init() {
    if (currentPage != null) {
        pagePath = currentPage.getPath();
        pageTitle = currentPage.getTitle();
    }
}
```

## HTL Usage

```html
<!--/* Auto-resolves via resourceType mapping */-->
<sly data-sly-use.model="com.myproject.core.models.HeroComponent"/>

<!--/* Or with parameters */-->
<sly data-sly-use.model="${'com.myproject.core.models.HeroComponent' @ showDetails=true}"/>

<div data-sly-test="${!model.empty}" class="cmp-hero">
    <h1>${model.title}</h1>
    <img src="${model.imagePath}" alt="${model.title}"/>
</div>
```

## Sling Model Exporter (JSON)

The `@Exporter` annotation enables JSON export at `.model.json`:

```
/content/mysite/page/jcr:content/hero.model.json
```

Returns:
```json
{
    ":type": "myproject/components/hero",
    "title": "Welcome",
    "imagePath": "/content/dam/hero.jpg"
}
```

Required for AEM SPA Editor integration.

## WCMUsePojo (Legacy)

Still works but **not recommended** for new code. Not unit-testable, not cacheable.

```java
public class MyHelper extends WCMUsePojo {
    private String result;

    @Override
    public void activate() throws Exception {
        result = getProperties().get("title", "");
    }

    public String getResult() { return result; }
}
```

Prefer Sling Models for all new development.

## Testing

Sling Models are testable with [AEM Mocks](https://wcm.io/testing/aem-mock/):

```java
@ExtendWith(AemContextExtension.class)
class HeroComponentImplTest {

    private final AemContext ctx = new AemContext();

    @BeforeEach
    void setUp() {
        ctx.addModelsForClasses(HeroComponentImpl.class);
        ctx.load().json("/hero-content.json", "/content/test");
    }

    @Test
    void testGetTitle() {
        ctx.currentResource("/content/test/hero");
        HeroComponent model = ctx.request().adaptTo(HeroComponent.class);
        assertEquals("My Title", model.getTitle());
    }
}
```

## Common Pitfalls

1. **Missing `@Model` adaptables** — model returns null when adapted.
2. **`Resource.class` adaptable** can't inject `@ScriptVariable` or `@RequestAttribute` —
   use `SlingHttpServletRequest.class` instead.
3. **Missing `defaultInjectionStrategy = OPTIONAL`** — model fails to instantiate if any
   property is missing from content.
4. **Forgetting `resourceType`** in `@Model` — prevents auto-resolution from HTL
   `data-sly-use` without FQCN.
5. **`@Inject` ambiguity** — prefer specific annotations (`@ValueMapValue`, `@ChildResource`,
   etc.) over generic `@Inject` to avoid injector priority surprises.
