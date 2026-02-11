---
name: htl-scripting
description: >
  Write, edit, and review AEM HTL (HTML Template Language) scripts. HTL is the server-side
  template system for AEM components — HTML files with ${expression} syntax and data-sly-*
  block attributes. Use when creating or modifying .html component scripts, fixing XSS context
  errors, building iteration/conditional markup, wiring Sling Models via data-sly-use, or
  constructing reusable HTL templates. Also activate when the user mentions HTL, Sightly,
  data-sly-*, or AEM component markup.
---

# HTL Scripting

HTL (HTML Template Language) is AEM's server-side template language. It replaces JSP for AEM
components. HTL files are valid HTML — all logic lives in `${expressions}` and `data-sly-*`
attributes which are stripped from output.

**Key mental model:** HTL = HTML + expressions (`${...}`) + block attributes (`data-sly-*`).
There is no imperative code in HTL. All business logic belongs in Java (Sling Models) or
JavaScript Use-API objects, referenced via `data-sly-use`.

## Expressions

Syntax: `${ expression @ option1, option2=value }`

```html
${myVar}                          <!--/* simple identifier */-->
${myObject.key}                   <!--/* dot access */-->
${myObject['key']}                <!--/* bracket access */-->
${myArray[0]}                     <!--/* array index */-->
${true}  ${42}  ${'literal'}      <!--/* literals */-->
${[1, 2, 3]}                      <!--/* array literal */-->
```

### Operators

```html
${!myVar}                              <!--/* NOT */-->
${a && b}                              <!--/* AND (returns first falsy or last value) */-->
${a || b}                              <!--/* OR  (returns first truthy or last value) */-->
${cond ? valA : valB}                  <!--/* ternary — ':' MUST have surrounding spaces */-->
${a == b}  ${a != b}                   <!--/* equality (strict, no type coercion) */-->
${a < b}  ${a <= b}  ${a > b}  ${a >= b}  <!--/* comparison (same-type only) */-->
${'bc' in 'abcd'}                      <!--/* string contains */-->
${item in myArray}                     <!--/* array/list membership */-->
${'key' in myMap}                      <!--/* object/map key check */-->
```

**Default value pattern** (|| returns first truthy operand):
```html
${properties.pageTitle || properties.jcr:title || resource.name}
```

### Boolean casting
Falsy: `false`, `0`, `''`, `""`, `[]` (empty iterable), `null`
Truthy: everything else (including `"false"` string, `[0]`)

### Expression options (`@ ...`)

Options modify expressions. Syntax: `${expr @ opt1, opt2=val}`.

**Display context** — override automatic XSS escaping:
```html
${value @ context='html'}        <!--/* allow safe HTML tags */-->
${value @ context='uri'}         <!--/* validate as URI */-->
${value @ context='text'}        <!--/* default for text nodes: encode HTML entities */-->
${value @ context='unsafe'}      <!--/* DISABLES all XSS protection — avoid */-->
```
See [references/xss-contexts.md](references/xss-contexts.md) for the full context table.

**Format** — string interpolation, dates, numbers:
```html
${'Asset {0} of {1}' @ format=[current, total]}
${'yyyy-MM-dd' @ format=myDate}
${'#,###.00' @ format=1000}
```

**i18n** — internationalization:
```html
${'Assets' @ i18n}
${'Assets' @ i18n, locale='de', hint='menu label'}
```

**Array join:**
```html
${['a','b','c'] @ join=', '}     <!--/* outputs: a, b, c */-->
```

**URI manipulation** — modify URL parts without string concatenation:
```html
${'page.html' @ selectors='print', extension='pdf'}
${'path/page.html' @ prependPath='/content/site', appendPath='jcr:content'}
${request.requestURL @ addQuery={'lang': 'en'}, fragment='section1'}
${url @ scheme='https', domain='example.com', removeSelectors}
```
URI options: `scheme`, `domain`, `path`, `prependPath`, `appendPath`, `selectors`,
`addSelectors`, `removeSelectors`, `extension`, `suffix`, `prependSuffix`, `appendSuffix`,
`query`, `addQuery`, `removeQuery`, `fragment`.

## Block Statements (`data-sly-*`)

All `data-sly-*` attributes are removed from rendered output. Multiple blocks can coexist on
one element — they execute in priority order (see end of section).

### data-sly-use — load logic

```html
<!--/* Sling Model (preferred for AEM as a Cloud Service) */-->
<sly data-sly-use.model="com.example.core.models.MyModel"/>
${model.title}

<!--/* Relative HTL template file */-->
<sly data-sly-use.tmpl="partials/card.html"/>

<!--/* With parameters */-->
<sly data-sly-use.nav="${'com.example.Nav' @ depth=2}"/>
```
- Identifier (`.model`) is **global** — usable anywhere after declaration.
- Without identifier, the object is available as `useBean`.
- Prefer **Sling Models** (Java Use-API) for AEM as a Cloud Service.
  JavaScript Use-API is deprecated for AEMaaCS.

### data-sly-text — set element text

```html
<p data-sly-text="${properties.jcr:title}">placeholder</p>
<!--/* outputs: <p>Actual Title</p> — placeholder replaced */-->
```
Applies `text` context (HTML-encodes) by default.

### data-sly-attribute — set attributes

```html
<!--/* Single attribute */-->
<div data-sly-attribute.class="${cssClass}"></div>

<!--/* Multiple from map */-->
<input data-sly-attribute="${{'id': 'foo', 'class': 'bar'}}" type="text"/>

<!--/* Boolean attribute */-->
<input data-sly-attribute.checked="${isChecked}"/>
<!--/* true → <input checked/>, false → <input/> */-->
```
- Empty string `${''}` **removes** the attribute.
- `on*` event handlers and `style` **cannot** be set (XSS risk).
- Processed left-to-right when multiple attribute blocks exist.

### data-sly-element — change tag name

```html
<div data-sly-element="${headingLevel}">Title</div>
<!--/* headingLevel='h2' → <h2>Title</h2> */-->
```
Restricted to a safe allowlist of element names (no `script`, `style`, `form`, `input`).

### data-sly-test — conditional rendering

```html
<p data-sly-test="${wcmmode.edit}">Edit mode only</p>

<!--/* Capture result in identifier (global scope, keeps original type) */-->
<sly data-sly-test.hasTitle="${properties.jcr:title}"/>
<h1 data-sly-test="${hasTitle}">${hasTitle}</h1>
```
- Omitted value = `false` (element never renders).
- Identifier stores the **original value**, not a boolean cast.

### data-sly-list — iterate (repeats content only)

```html
<ul data-sly-list="${currentPage.listChildren}">
    <li>${item.title} (${itemList.count})</li>
</ul>

<!--/* Custom identifier */-->
<ul data-sly-list.child="${pages}">
    <li class="${childList.first ? 'first' : ''}">${child.title}</li>
</ul>

<!--/* Iteration control */-->
<ul data-sly-list="${items @ begin=0, end=9, step=2}">
    <li>${item.name}</li>
</ul>
```
- Default item variable: `item`. Metadata: `itemList`.
- Custom `data-sly-list.foo` → item is `foo`, metadata is `fooList`.
- Metadata properties: `index` (0-based), `count` (1-based), `first`, `middle`, `last`, `odd`, `even`.
- For Maps: `item` = key, access value with `${myMap[item]}`.
- **Element is hidden** if collection is empty.

### data-sly-repeat — iterate (repeats entire element)

```html
<div data-sly-repeat.article="${articles}" id="${article.id}">
    ${article.excerpt}
</div>
<!--/* Outputs N <div> elements, one per article */-->
```
Same options/metadata as `data-sly-list`. Difference: `list` repeats **inner content**,
`repeat` repeats the **host element**.

### data-sly-include — include another script

```html
<sly data-sly-include="header.html"/>
<sly data-sly-include="${'template.html' @ prependPath='partials'}"/>
```
- Replaces element content. Host element is **not rendered**.
- Variables from current scope are **not** passed to included script.
- AEM extension: `wcmmode` option controls WCM mode for included script.

### data-sly-resource — include a resource (sub-request)

```html
<div data-sly-resource="./header"></div>
<div data-sly-resource="${'./content' @ resourceType='myapp/components/text'}"></div>
<div data-sly-resource="${'./list' @ selectors='summary', removeSelectors}"></div>
```
- Creates a **new rendering context** (separate request).
- Options: `resourceType`, `selectors`, `addSelectors`, `removeSelectors`, `appendPath`,
  `prependPath`, `requestAttributes`.
- AEM extension: accepts `Map` or `Record` objects with a `resourceName` property to create
  synthetic resources. If `sling:resourceType` is missing, falls back to `resourceType` option
  or current resource type.

### data-sly-template / data-sly-call — reusable templates

```html
<!--/* Define */-->
<template data-sly-template.card="${@ title, description, link}">
    <div class="card">
        <h3>${title}</h3>
        <p>${description}</p>
        <a href="${link}">Read more</a>
    </div>
</template>

<!--/* Call */-->
<sly data-sly-call="${card @ title='Hello', description=desc, link=url}"/>

<!--/* Load from external file */-->
<sly data-sly-use.lib="templates/cards.html"/>
<sly data-sly-call="${lib.card @ title='Hello', description=desc, link=url}"/>
```
- Template element is **never** shown.
- Missing parameters become empty string.
- Scope is **not** inherited — pass everything via parameters.

### data-sly-unwrap — remove host element

```html
<div data-sly-unwrap>Content shows without div wrapper</div>
<div data-sly-unwrap="${wcmmode.edit}">Unwrapped in edit mode only</div>
```

### data-sly-set — assign a variable

```html
<sly data-sly-set.fullName="${profile.firstName} ${profile.lastName}"/>
<p>${fullName}</p>
```
Global scope after declaration.

### The `<sly>` tag

A virtual container element — never renders in output. Use it to apply block logic without
adding markup:

```html
<sly data-sly-test="${showBanner}" data-sly-resource="./banner"/>
```

### Block priority order

When multiple `data-sly-*` appear on the same element:
1. `template`
2. `set`, `test`, `use`
3. `call`
4. `text`
5. `element`, `include`, `resource`
6. `unwrap`
7. `list`, `repeat`
8. `attribute`

Same-priority → evaluated left-to-right.

## Comments

```html
<!--/* HTL comment: stripped from output entirely */-->
<!-- HTML comment: kept in output, expressions inside ARE evaluated -->
```

## Common Patterns

### Conditional CSS classes
```html
<div class="${item.active ? 'nav-item active' : 'nav-item'}">
<div data-sly-attribute.class="${['nav-item', item.active ? 'active' : ''] @ join=' '}">
```

### Safe link rendering
```html
<a href="${page.path @ extension='html'}" title="${page.title}">${page.navTitle || page.title}</a>
```

### Component with edit placeholder
```html
<sly data-sly-use.model="com.example.MyModel"/>
<div data-sly-test="${model.hasContent}" class="my-component">
    ${model.text @ context='html'}
</div>
<div data-sly-test="${!model.hasContent && wcmmode.edit}" class="cq-placeholder">
    Configure this component
</div>
```

### Recursive template
```html
<template data-sly-template.nav="${@ items}">
    <ul data-sly-list="${items}">
        <li>
            ${item.title}
            <sly data-sly-call="${nav @ items=item.children}"/>
        </li>
    </ul>
</template>
<sly data-sly-call="${nav @ items=rootItems}"/>
```

## Critical Rules

1. **Never use `context='unsafe'`** unless explicitly required and security-reviewed.
2. **Script/style contexts require explicit context** — expressions in `<script>` or `<style>` produce no output without `@ context='scriptToken'` etc.
3. **`on*` and `style` attributes cannot be set** via `data-sly-attribute`.
4. **Ternary `:` requires spaces** — `${a ? b : c}` not `${a ? b:c}`.
5. **`==` is strict** — no type coercion. Compare same types.
6. **Prefer Sling Models** over JS Use-API for AEM as a Cloud Service.
7. **`data-sly-list` hides the host element** when collection is empty. Use `data-sly-test` first if you need fallback markup.
8. **Template scope is isolated** — pass all needed data as parameters to `data-sly-call`.

## Reference Files

- [references/xss-contexts.md](references/xss-contexts.md) — Full XSS display context table with allowed values
- [references/global-objects.md](references/global-objects.md) — AEM/Sling global objects, Use-API patterns, Sling Model integration
