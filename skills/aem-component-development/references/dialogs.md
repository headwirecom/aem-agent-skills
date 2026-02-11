# Touch UI Dialogs (Granite UI)

## Dialog Structure

Dialogs are defined as `cq:dialog` nodes (`nt:unstructured`) with `sling:resourceType`
set to `cq/gui/components/authoring/dialog`. The structure follows a fixed nesting pattern:

```
cq:dialog (nt:unstructured)
  └── content (container)
        └── items
              └── tabs (coral/foundation/tabs)
                    └── items
                          └── <tab-name> (container)
                                └── items
                                      └── columns (fixedcolumns)
                                            └── items
                                                  └── column (container)
                                                        └── items
                                                              ├── <field-1>
                                                              └── <field-2>
```

For single-tab dialogs, omit the `tabs` layer and put fields directly in `content/items`.

## Field Name Convention

The `name` property on dialog fields maps to JCR property paths relative to the component's
content node:

| Name value | Writes to |
|---|---|
| `./title` | `title` property on component node |
| `./jcr:title` | `jcr:title` property on component node |
| `./image/fileReference` | `fileReference` under `image` child node |
| `../../jcr:title` | `jcr:title` on parent node (use carefully) |

Always use the `./` prefix for component-scoped properties.

## Common Granite UI Field Types

All under `sling:resourceType = "granite/ui/components/coral/foundation/form/<type>"`:

### Text Input
```xml
<title jcr:primaryType="nt:unstructured"
       sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
       fieldLabel="Title"
       fieldDescription="The component title"
       name="./title"
       required="{Boolean}true"
       maxlength="120"
       emptyText="Enter a title"/>
```

### Textarea
```xml
<description jcr:primaryType="nt:unstructured"
             sling:resourceType="granite/ui/components/coral/foundation/form/textarea"
             fieldLabel="Description"
             name="./description"
             rows="5"/>
```

### Rich Text Editor
```xml
<text jcr:primaryType="nt:unstructured"
      sling:resourceType="cq/gui/components/authoring/dialog/richtext"
      fieldLabel="Text"
      name="./text"
      useFixedInlineToolbar="{Boolean}true">
    <rtePlugins jcr:primaryType="nt:unstructured">
        <format jcr:primaryType="nt:unstructured"
                features="bold,italic"/>
        <links jcr:primaryType="nt:unstructured"
               features="modifylink,unlink"/>
        <lists jcr:primaryType="nt:unstructured"
               features="*"/>
    </rtePlugins>
</text>
```

### Checkbox
```xml
<hideTitle jcr:primaryType="nt:unstructured"
           sling:resourceType="granite/ui/components/coral/foundation/form/checkbox"
           fieldDescription="Hide the title"
           name="./hideTitle"
           text="Hide title"
           value="{Boolean}true"
           uncheckedValue="{Boolean}false"/>
```

### Select / Dropdown
```xml
<alignment jcr:primaryType="nt:unstructured"
           sling:resourceType="granite/ui/components/coral/foundation/form/select"
           fieldLabel="Alignment"
           name="./alignment">
    <items jcr:primaryType="nt:unstructured">
        <left jcr:primaryType="nt:unstructured" text="Left" value="left"/>
        <center jcr:primaryType="nt:unstructured" text="Center" value="center"/>
        <right jcr:primaryType="nt:unstructured" text="Right" value="right"/>
    </items>
</alignment>
```

### Path Browser (page/asset picker)
```xml
<linkURL jcr:primaryType="nt:unstructured"
         sling:resourceType="granite/ui/components/coral/foundation/form/pathfield"
         fieldLabel="Link"
         name="./linkURL"
         rootPath="/content"
         filter="hierarchyNotFile"/>
```

### Image / File Upload
```xml
<file jcr:primaryType="nt:unstructured"
      sling:resourceType="cq/gui/components/authoring/dialog/fileupload"
      fieldLabel="Image"
      name="./image/file"
      fileNameParameter="./image/fileName"
      fileReferenceParameter="./image/fileReference"
      allowUpload="{Boolean}false"
      mimeTypes="[image/gif,image/jpeg,image/png,image/svg+xml]"/>
```

### Number Field
```xml
<count jcr:primaryType="nt:unstructured"
       sling:resourceType="granite/ui/components/coral/foundation/form/numberfield"
       fieldLabel="Count"
       name="./count"
       min="1"
       max="100"
       step="1"/>
```

### Hidden Field
```xml
<resourceType jcr:primaryType="nt:unstructured"
              sling:resourceType="granite/ui/components/coral/foundation/form/hidden"
              name="./sling:resourceType"
              value="myproject/components/mycomponent"/>
```

### Multifield (repeatable fields)
```xml
<links jcr:primaryType="nt:unstructured"
       sling:resourceType="granite/ui/components/coral/foundation/form/multifield"
       fieldLabel="Links"
       composite="{Boolean}true">
    <field jcr:primaryType="nt:unstructured"
           sling:resourceType="granite/ui/components/coral/foundation/container"
           name="./links">
        <items jcr:primaryType="nt:unstructured">
            <text jcr:primaryType="nt:unstructured"
                  sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
                  fieldLabel="Label"
                  name="text"/>
            <url jcr:primaryType="nt:unstructured"
                 sling:resourceType="granite/ui/components/coral/foundation/form/pathfield"
                 fieldLabel="URL"
                 name="url"/>
        </items>
    </field>
</links>
```

Composite multifield stores data as child nodes. Non-composite stores as multi-value property.

### Color Picker
```xml
<color jcr:primaryType="nt:unstructured"
       sling:resourceType="granite/ui/components/coral/foundation/form/colorfield"
       fieldLabel="Background Color"
       name="./backgroundColor"
       showDefaultColors="{Boolean}true"/>
```

## Validation

### Required Fields
Set `required="{Boolean}true"` on any field.

### Regex Validation
```xml
<email jcr:primaryType="nt:unstructured"
       sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
       fieldLabel="Email"
       name="./email"
       validation="email"/>
```

Built-in validators: `email`, `url`, `number`, `date`.

Custom regex via Granite `foundation-validation` API in a client library with category
`cq.authoring.dialog`.

## Conditional Field Visibility (Granite show/hide)

Use `granite:class` and `granite:data` for client-side toggle:

```xml
<toggle jcr:primaryType="nt:unstructured"
        sling:resourceType="granite/ui/components/coral/foundation/form/checkbox"
        text="Show Advanced"
        name="./showAdvanced"
        value="true"
        granite:class="cmp-mycomponent__toggle"/>

<advancedField jcr:primaryType="nt:unstructured"
               sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
               fieldLabel="Advanced Setting"
               name="./advancedSetting"
               granite:class="cmp-mycomponent__advanced hide"/>
```

Requires companion clientlib JS listening to checkbox change events.

## Design Dialog (`cq:design_dialog`)

Same structure as `cq:dialog` but used for template-level policies (Style System, allowed
features). Defined as `_cq_design_dialog/.content.xml` with resource type
`cq/gui/components/authoring/dialog`.

## Dialog Overlay with Sling Resource Merger

To modify a parent component's dialog without copying it entirely:

```xml
<jcr:root ...
    jcr:primaryType="nt:unstructured"
    sling:resourceType="cq/gui/components/authoring/dialog"
    sling:resourceSuperType="core/wcm/components/text/v2/text/cq:dialog">
    <content jcr:primaryType="nt:unstructured">
        <items jcr:primaryType="nt:unstructured">
            <tabs jcr:primaryType="nt:unstructured">
                <items jcr:primaryType="nt:unstructured">
                    <customtab .../>  <!--/* adds a new tab */-->
                </items>
            </tabs>
        </items>
    </content>
</jcr:root>
```

Sling Resource Merger properties for overlays:
- `sling:hideResource` — hide inherited node
- `sling:hideChildren` — hide specific child names
- `sling:orderBefore` — reorder nodes
