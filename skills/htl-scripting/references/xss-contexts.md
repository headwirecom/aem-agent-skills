# XSS Display Contexts

HTL provides automatic context-aware XSS protection. Override with `@ context='name'`.

## Context Table

| Context | Default for | Behavior |
|---|---|---|
| `text` | HTML text nodes | Encodes all HTML special characters (`<`, `>`, `&`, `"`) |
| `attribute` | Attribute values | Encodes all HTML special characters |
| `attributeName` | `data-sly-attribute` key names | Validates name; outputs nothing if invalid |
| `elementName` | `data-sly-element` | Validates against allowlist; outputs nothing if invalid |
| `html` | — | Filters HTML, removes dangerous tags (`script`, `style`, event handlers) while keeping safe markup |
| `number` | — | Validates value is a number; outputs nothing otherwise |
| `uri` | `href`, `src`, `action`, `formaction`, `cite`, `data`, `manifest`, `poster` | Validates URI; outputs nothing if validation fails |
| `scriptToken` | — | Validates as JS identifier, literal number, or literal string; outputs nothing otherwise |
| `scriptString` | — | Encodes characters that would break out of a JS string |
| `scriptComment` | — | Validates as JS comment content; outputs nothing otherwise |
| `styleToken` | — | Validates as CSS identifier, number, dimension, string, hex color, or function; outputs nothing otherwise |
| `styleString` | — | Encodes characters that would break out of a CSS string |
| `styleComment` | — | Validates as CSS comment content; outputs nothing otherwise |
| `unsafe` | — | **Disables all escaping and XSS protection. Avoid.** |
| `jsonString` | — | (Sling extension) Escapes text per ECMA-404 JSON string grammar |
| `comment` | HTML comments `<!-- -->` | Implied for expressions inside HTML comments |

## Automatic Context Detection

HTL detects context from position in markup:

```html
<p>${text}</p>                     <!--/* → text context */-->
<p title="${val}"></p>             <!--/* → attribute context */-->
<a href="${url}"></a>              <!--/* → uri context */-->
<img src="${imgPath}"/>            <!--/* → uri context */-->
<!-- ${val} -->                    <!--/* → comment context */-->
```

## URI-context Attributes

These attributes automatically use `uri` context:
- `action` (`<form>`)
- `cite` (`<blockquote>`, `<del>`, `<ins>`, `<q>`)
- `data` (`<object>`)
- `formaction` (`<button>`, `<input>`)
- `href` (`<a>`, `<area>`, `<link>`, `<base>`)
- `manifest` (`<html>`)
- `poster` (`<video>`)
- `src` (`<audio>`, `<embed>`, `<iframe>`, `<img>`, `<input>`, `<script>`, `<source>`, `<track>`, `<video>`)

## Script and Style Contexts

Expressions in `<script>` or `<style>` blocks produce **no output** unless an explicit context is set:

```html
<!--/* WRONG — outputs nothing */-->
<script>var x = ${value};</script>

<!--/* CORRECT */-->
<script>var name = '${value @ context="scriptString"}';</script>
<script>var fn = ${fnName @ context="scriptToken"};</script>

<style>
    .${cls @ context="styleToken"} {
        font-family: '${font @ context="styleString"}', sans-serif;
        color: #${hex @ context="styleToken"};
    }
</style>
```

## Allowed Element Names (`elementName` context)

`data-sly-element` only allows these tag names:

```
section, nav, article, aside, h1, h2, h3, h4, h5, h6, header, footer,
address, main, p, pre, blockquote, ol, li, dl, dt, dd, figure, figcaption,
div, a, em, strong, small, s, cite, q, dfn, abbr, data, time, code, var,
samp, kbd, sub, sup, i, b, u, mark, ruby, rt, rp, bdi, bdo, span, br,
wbr, ins, del, table, caption, colgroup, col, tbody, thead, tfoot, tr, td, th
```

Notably excluded: `script`, `style`, `form`, `input`, `button`, `select`, `textarea`, `iframe`, `object`, `embed`.
