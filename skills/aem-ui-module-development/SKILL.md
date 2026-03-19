---
name: aem-ui-module-development
description: >
  Build, structure, and register frontend JavaScript modules for AEM components using Vite 
  and the DOM-driven Registry pattern (Island Architecture). Covers the transition away from 
  HTL-injected script tags (`<sly>`) to a `MutationObserver`-driven global entry point (`app.ts`), 
  dynamic module loading (`registry.ts`), and safe memory management (`WeakMap` and `AbortController`). 
  Use when creating or refactoring frontend JS for AEM, handling DOM event bindings, ensuring 
  AEM authoring drag-and-drop compatibility, or optimizing Vite chunking and bundle sizes. 
  Also activate when the user mentions data-ui-module, top-level side effects, UI teardown, 
  or AEM frontend performance.
---

# AEM + Vite Registry Architecture (Island Pattern)

In modern AEM implementations using Vite, we avoid injecting fragmented `<script type="module">` 
tags via server-side HTL (`<sly data-sly-use... />`). Instead, we use a single entry point (`app.ts`) 
that relies on a `MutationObserver` to watch the DOM for specific data attributes (`data-ui-module`). 

When an element is found (either on page load or when an AEM author drags a component onto the page), 
the global app dynamically imports the required script via `registry.ts` and initializes it.

**Benefits:**
1. **Perfect Vite Chunking:** One entry point allows Vite to resolve the full dependency graph and generate optimized vendor chunks.
2. **Authoring DX:** Components initialize instantly upon AJAX injection in the AEM Edit dialog without page reloads.
3. **No Side Effects:** Scripts are pure and only execute when explicitly invoked with a target DOM element.
4. **Memory Safety:** Teardown is handled automatically when AEM removes a component from the DOM.

## Architecture File Structure

```text
ui.frontend/src/
├── app.ts                  ← The main entry point (MutationObserver + WeakMap)
├── registry.ts             ← Maps DOM attribute strings to dynamic Vite imports
├── types/
│   └── UIModule.ts         ← TypeScript interface for module contract
└── components/
    └── carousel/
        ├── carousel.ts     ← The component JS (exports initModule)
        └── carousel.scss   ← The component styling
```

## The Module Contract (`UIModule.ts`)

Every UI module must adhere to this signature. It prevents top-level execution and enforces cleanup.

```typescript
import type { AppContext } from "@ui/types/AppContext";

export interface UIModule {
  /** Returns a teardown function, or void if no teardown is required. */
  destroy?: () => void;
}

export interface UIModuleDefinition {
  initModule: (el: HTMLElement, context: AppContext) => Promise<UIModule | void> | UIModule | void;
}
```

## Agent Workflow: Creating a New Frontend Module

### Step 1: Add the trigger to the HTL

Tag the root element of your AEM component with `data-ui-module`. Do **not** use `<sly>` to include the JS.

```html
<div class="cmp-my-component" data-ui-module="components/my-component">
    <button class="cmp-my-component__btn">Click Me</button>
</div>
```

### Step 2: Create the component script

Write the TypeScript module. **Do not execute any DOM queries at the top level.** Use an `AbortController` to manage event listeners so they can be easily destroyed.

```typescript
// components/my-component/my-component.ts
import { createConsolePrefix } from "@ui/utils/common-utils";
import type { AppContext } from "@ui/types/AppContext";

const cp = createConsolePrefix("my-component");

export async function initModule(el: HTMLElement, ctx: AppContext) {
  const btn = el.querySelector<HTMLButtonElement>(".cmp-my-component__btn");
  if (!btn) return;

  // 1. Create state for cleanup
  const abortController = new AbortController();
  const { signal } = abortController;

  // 2. Bind events using the signal
  btn.addEventListener("click", () => {
    console.log(...cp, "Button clicked!");
  }, { signal });

  // 3. Return the teardown function
  return {
    destroy: () => {
      console.debug(...cp, "Cleaning up my-component");
      abortController.abort(); // Instantly removes all event listeners
    }
  };
}
```

### Step 3: Register the module

Map the string used in the HTL to the dynamic import in the registry.

```typescript
// registry.ts
import type { UIModuleDefinition } from "./types/UIModule";

export const registry: Record<string, () => Promise<UIModuleDefinition>> = {
  "components/carousel": () => import("./components/carousel/carousel"),
  "components/my-component": () => import("./components/my-component/my-component"),
  // Add new modules here...
};
```

## The Engine: App Initialization (`app.ts`)

The core application relies on a `WeakMap` to track element state and a `MutationObserver` to handle AEM authoring lifecycle events. *(Reference this when debugging initialization flows).*

```typescript
// app.ts
import { registry } from "./registry";

const attribute = "data-ui-module";
const moduleInstances = new WeakMap<HTMLElement, "pending" | { destroy?: () => void }>();

export async function initModule(el: HTMLElement): Promise<void> {
  if (moduleInstances.has(el)) return; // Prevent double initialization

  const moduleName = el.getAttribute(attribute);
  if (!moduleName) return;

  const loader = registry[moduleName];
  if (!loader) {
    console.warn(`No module registered for: ${moduleName}`);
    return;
  }

  moduleInstances.set(el, "pending"); // Lock

  try {
    const uiModule = await loader();
    const instanceAPI = await uiModule.initModule(el, { /* AppContext */ });
    
    // Store the returned destroy function (if any)
    moduleInstances.set(el, instanceAPI || { destroy: undefined });
  } catch (error) {
    console.error(`Failed to load module: ${moduleName}`, error);
    moduleInstances.delete(el); // Unlock on failure so it can be retried
  }
}

// Observe DOM for AEM Component Additions/Removals
new MutationObserver((mutations) => {
  for (const mutation of mutations) {
    // Additions
    mutation.addedNodes.forEach((node) => {
      if (!(node instanceof HTMLElement)) return;
      if (node.hasAttribute(attribute)) void initModule(node);
      node.querySelectorAll<HTMLElement>(`[${attribute}]`).forEach(initModule);
    });

    // Removals (Teardown)
    mutation.removedNodes.forEach((node) => {
      if (!(node instanceof HTMLElement)) return;
      const elements = node.hasAttribute(attribute) 
        ? [node, ...node.querySelectorAll<HTMLElement>(`[${attribute}]`)]
        : Array.from(node.querySelectorAll<HTMLElement>(`[${attribute}]`));

      for (const el of elements) {
        const instance = moduleInstances.get(el);
        if (instance && instance !== "pending" && typeof instance.destroy === "function") {
          instance.destroy();
          moduleInstances.delete(el);
        }
      }
    });
  }
}).observe(document.body, { childList: true, subtree: true });
```

## Critical Rules

1. **NO top-level side effects:** Never run `document.querySelector` or `addEventListener` at the root level of a module. Code must only execute inside `initModule`.
2. **Always scope selectors to the element:** Use `el.querySelector`, not `document.querySelector`, to ensure you only manipulate the specific component instance.
3. **Always use `AbortController`:** Pass `{ signal: abortController.signal }` to all `addEventListener` calls, especially global ones (like `window.addEventListener('scroll')`).
4. **Return a `destroy` method:** If your component creates global listeners, intervals, or third-party library instances (like Swiper), you MUST return a `{ destroy: () => void }` object from `initModule`.
5. **NEVER use HTL script injection for UI modules:** Do not use `<sly data-sly-call="${... @ modulePath='...'}">` for standard component JavaScript. Rely strictly on `data-ui-module`. Web Components (Lit) are the only exception as they self-register.
6. **Use `WeakMap` for state:** Do not store component instances in global arrays or objects, as this causes memory leaks. Rely on the `app.ts` `WeakMap`.