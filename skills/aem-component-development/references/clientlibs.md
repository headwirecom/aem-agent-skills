# Client Libraries (clientlibs)

Client libraries manage CSS and JS for AEM components. They handle dependency resolution,
ordering, minification, and deduplication.

## Structure

A client library is a `cq:ClientLibraryFolder` node:

```
clientlib/
├── .content.xml     ← cq:ClientLibraryFolder definition
├── css.txt          ← CSS file manifest
├── js.txt           ← JS file manifest
├── css/
│   └── styles.css
└── js/
    └── script.js
```

### .content.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:cq="http://www.day.com/jcr/cq/1.0"
          xmlns:jcr="http://www.jcp.org/jcr/1.0"
    jcr:primaryType="cq:ClientLibraryFolder"
    categories="[myproject.components.hero]"
    allowProxy="{Boolean}true"/>
```

### Manifest files (css.txt / js.txt)
```
#base=css
styles.css
```

```
#base=js
script.js
```

`#base=` sets the relative root. Each subsequent line is a file path. Order matters.

## Key Properties

| Property | Type | Purpose |
|---|---|---|
| `categories` | `String[]` | Category names used to reference this library |
| `dependencies` | `String[]` | Categories that must load before this one |
| `embed` | `String[]` | Categories to inline into this library's output |
| `allowProxy` | `Boolean` | **Required for `/apps`** — serves via `/etc.clientlibs/` proxy |
| `channels` | `String[]` | Device channels (`touch`, `!touch`) |
| `cssProcessor` | `String[]` | CSS minification config |
| `jsProcessor` | `String[]` | JS minification config |

## Loading in HTL

Use the AEM clientlib helper template:

```html
<!--/* Load both CSS and JS */-->
<sly data-sly-use.clientlib="/libs/granite/sightly/templates/clientlib.html"/>
<sly data-sly-call="${clientlib.all @ categories='myproject.base'}"/>

<!--/* CSS only (in <head>) */-->
<sly data-sly-call="${clientlib.css @ categories='myproject.base'}"/>

<!--/* JS only (before </body>) */-->
<sly data-sly-call="${clientlib.js @ categories='myproject.base'}"/>

<!--/* Multiple categories */-->
<sly data-sly-call="${clientlib.all @ categories=['myproject.base', 'myproject.vendor']}"/>
```

## Category Naming Conventions

```
myproject.base          ← site-wide CSS/JS
myproject.vendor        ← third-party libraries
myproject.components    ← all component CSS/JS (if bundled)
myproject.components.hero ← per-component clientlib
cq.authoring.dialog     ← loaded in all authoring dialogs
```

## Proxy Servlet (`allowProxy`)

Libraries under `/apps` are not publicly accessible. Set `allowProxy="{Boolean}true"` and
they become available at `/etc.clientlibs/<path-under-apps>`.

Example:
- Library at `/apps/myproject/clientlibs/base`
- Accessible at `/etc.clientlibs/myproject/clientlibs/base.css`

Static resources (images, fonts) must be in a `resources/` subfolder:
- File: `/apps/myproject/clientlibs/base/resources/logo.png`
- URL: `/etc.clientlibs/myproject/clientlibs/base/resources/logo.png`

## Dependencies vs Embed

**Dependencies:** Load other libraries before this one. Separate HTTP requests.
```xml
<jcr:root ...
    categories="[myproject.components.hero]"
    dependencies="[myproject.base]"/>
```

**Embed:** Merge other libraries INTO this one. Single HTTP request.
```xml
<jcr:root ...
    categories="[myproject.base]"
    embed="[myproject.vendor.jquery, myproject.vendor.handlebars]"/>
```

Use embed to reduce HTTP requests. Use dependencies for shared libraries that should be
cached independently.

## Per-Component Clientlib Pattern

Co-locate clientlib with the component:

```
apps/myproject/components/hero/
├── .content.xml
├── hero.html
└── clientlib/
    ├── .content.xml    ← categories="[myproject.components.hero]"
    ├── css.txt
    ├── js.txt
    ├── css/styles.css
    └── js/script.js
```

Then embed all component clientlibs into a base library:
```xml
<!--/* In a central clientlib */-->
<jcr:root ...
    categories="[myproject.components]"
    embed="[myproject.components.hero, myproject.components.teaser, ...]"/>
```

## Debugging

Append `?debugClientLibs=true` to any page URL to see individual unmerged files instead of
concatenated output. Useful for finding which source file causes an issue.

Library dump page: `http://localhost:4502/libs/granite/ui/content/dumplibs.html`

## Preprocessors

Default minifier: YUI Compressor. Can switch to Google Closure Compiler:

```xml
<jcr:root ...
    categories="[myproject.base]"
    jsProcessor="[default:none, min:gcc;compilationLevel=simple]"
    cssProcessor="[default:none, min:yui]"/>
```

`default:none` = no processing in development. `min:gcc` = use GCC for minified builds.

## Common Pitfalls

1. **Missing `allowProxy`** — CSS/JS returns 403 on publish if library is under `/apps`
   without this property.
2. **Missing `#base=` line** in css.txt/js.txt — files won't be found.
3. **Wrong category name** in HTL `clientlib.all` call — silent failure, no CSS/JS loads.
4. **Static resources not in `resources/`** subfolder — images/fonts 404 on publish.
5. **Circular embed** — two libraries embedding each other causes infinite loop.
