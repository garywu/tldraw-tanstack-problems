# Building with tldraw, TanStack Start, and AI Agents: Every Problem We Hit (and How We Fixed Them)

We built Frontasy Studio -- an AI-powered design tool where you describe an app in a chat panel and an AI agent draws wireframes on a [tldraw](https://tldraw.dev) canvas. The stack: TanStack Start for the React SSR framework, tldraw v3 for the infinite canvas, and the Claude Code Agent SDK with Haiku subagents for design generation.

Every individual tool worked fine in isolation. The problems were all at the intersections -- where two tools' assumptions clashed with each other. This article documents every bug we hit, the wrong approaches we tried first, and the fixes that actually worked. If you are building with any combination of these tools, this will save you hours.

**What you will learn:**

- Why Biome v2 config files get corrupted by AI agents (and the three things that break)
- The exact model name format the Claude Agent SDK requires (and why "haiku" returns 404)
- What happens when a Haiku subagent rewrites your code using the wrong SDK
- The one Vite config line that fixes "React is not defined" in TanStack Start
- The two-file pattern that makes tldraw work with any SSR framework
- How tldraw v3.10+ broke every shape that uses text (and the ProseMirror format you need)
- Why Vite HMR becomes unrecoverable when AI agents rewrite files rapidly

---

## Table of Contents

1. [The Stack](#the-stack)
2. [Problem 1: Biome Config Corruption](#problem-1-biome-config-corruption)
3. [Problem 2: Claude Code Agent SDK Model Names](#problem-2-claude-code-agent-sdk-model-names)
4. [Problem 3: Haiku Subagent Rewrote Code with the Wrong SDK](#problem-3-haiku-subagent-rewrote-code-with-the-wrong-sdk)
5. [Problem 4: TanStack Start + "React is not defined"](#problem-4-tanstack-start--react-is-not-defined)
6. [Problem 5: tldraw + SSR Framework = Crash](#problem-5-tldraw--ssr-framework--crash)
7. [Problem 6: tldraw v3.10+ API Breaking Change -- text to richText](#problem-6-tldraw-v310-api-breaking-change----text-to-richtext)
8. [Problem 7: Vite HMR Breaks with Rapid File Rewrites](#problem-7-vite-hmr-breaks-with-rapid-file-rewrites)
9. [Patterns: What Breaks at the Intersections](#patterns-what-breaks-at-the-intersections)
10. [The Complete Working Setup](#the-complete-working-setup)
11. [Comparison: SSR Frameworks + Canvas Libraries](#comparison-ssr-frameworks--canvas-libraries)
12. [Anti-Patterns](#anti-patterns)
13. [References](#references)

---

## The Stack

Before diving into the problems, here is the architecture. Understanding the moving parts explains why the intersections are where things break.

```
+---------------------------+
|     Frontasy Studio       |
|  (TanStack Start app)     |
+---------------------------+
|  React 19 + Vite 6        |
|  TanStack Router (file)   |
|  Tailwind CSS v4           |
+---------------------------+
|       tldraw v3.15         |
|   (infinite canvas SDK)    |
+---------------------------+

+---------------------------+
|     Frontasy CLI           |
+---------------------------+
|  Claude Code Agent SDK     |
|  @anthropic-ai/claude-code |
|  Haiku subagent for design |
+---------------------------+
|  Design Bank (YAML)        |
|  Screenplay Schema (Zod)   |
+---------------------------+

+---------------------------+
|     Tooling                |
+---------------------------+
|  Biome v2 (lint/format)    |
|  Husky + lint-staged       |
|  commitlint (conventional) |
+---------------------------+
```

The Studio is the visual frontend. The CLI spawns AI agents to generate designs. Biome + Husky enforce code quality on commit. Three separate systems that all need to work together.

### Key Dependencies

```json
{
  "@tanstack/react-start": "^1.120.3",
  "@tanstack/react-router": "^1.120.3",
  "@tldraw/tldraw": "^3.12.0",
  "react": "^19.0.0",
  "vite": "^6.2.0",
  "@biomejs/biome": "^2.4.7",
  "@anthropic-ai/claude-code": "latest"
}
```

---

## Problem 1: Biome Config Corruption

**Severity:** Blocks all commits. Nothing gets past lint-staged.

**When it happened:** Early setup, when Claude Code (the AI agent) was scaffolding the project.

### The Symptom

Every `git commit` fails:

```
> biome check --write

biome.json: Failed to parse configuration file
error: expected value at line 1, column 1
```

### What Went Wrong

Three separate things broke, each independently fatal.

#### Issue 1A: The file contained literal text "undefined"

The AI agent wrote `biome.json` with the JavaScript value `undefined` rendered as a string:

```
$ cat biome.json
undefined
```

Not `{}`. Not an empty file. The literal word "undefined". This happens when an AI agent uses `JSON.stringify()` on a variable that is `undefined` in a template, or when a file-write tool receives `undefined` as content and serializes it.

Our git log tells the story -- 30+ commits of "chore: update biome.json" as the agent kept trying to fix it by writing new configs on top of the corrupted file:

```
88c6a98 chore: update biome.json   # content: "undefined"
1b75ec1 chore: update biome.json   # content: "undefined"
0d0e04e chore: update biome.json   # content: "undefined"
d17ccf9 chore: update biome.json   # ...you get the idea
```

**Fix:** Delete the file, write a valid JSON config from scratch. The agent could not self-correct because it kept invoking the same broken write path.

#### Issue 1B: Biome v2 uses `includes`, not `include`

Biome v2 renamed the file-matching configuration keys:

| Biome v1 | Biome v2 |
|----------|----------|
| `files.include` | `files.includes` |
| `files.ignore` | `files.includes` (with negation `!pattern`) |

The AI agent "knew" Biome from its training data (v1) and wrote:

```json
{
  "files": {
    "include": ["**/*.ts", "**/*.tsx"]
  }
}
```

Biome v2 silently ignores `include` (unknown key). No error. It just processes *all* files instead of the ones you specified. This manifests as linting files you did not intend to lint (node_modules, dist, etc.) and produces hundreds of spurious errors.

**Fix:** Use the v2 key name:

```json
{
  "files": {
    "includes": ["**/*.ts", "**/*.tsx", "**/*.js", "**/*.jsx", "**/*.json", "**/*.css", "**/*.md"]
  }
}
```

The [Biome v2 upgrade guide](https://biomejs.dev/guides/upgrade-to-biome-v2/) documents this change, but only if you read it. The `biome migrate` command is supposed to handle it automatically, but has [known issues with subdirectory patterns](https://github.com/biomejs/biome/issues/6272).

#### Issue 1C: Schema version mismatch causes hard failure

The `$schema` URL in `biome.json` must match the installed version:

```json
{
  "$schema": "https://biomejs.dev/schemas/2.4.8/schema.json"
}
```

If you have `@biomejs/biome@2.4.7` installed but your schema URL points to `2.4.8`, Biome fails to validate the config. The error message does not mention the version mismatch -- it just says the config is invalid.

**Fix:** Ensure the schema version in the URL matches `package.json`:

```json
{
  "$schema": "https://biomejs.dev/schemas/2.4.7/schema.json"
}
```

Or better: do not pin the schema URL at all and let the CLI resolve it.

### The Working Config

After all three issues, this is what works:

```json
{
  "$schema": "https://biomejs.dev/schemas/2.4.8/schema.json",
  "files": {
    "includes": ["**/*.ts", "**/*.tsx", "**/*.js", "**/*.jsx", "**/*.json", "**/*.css", "**/*.md"]
  },
  "formatter": { "enabled": false },
  "linter": { "enabled": false },
  "css": { "parser": { "tailwindDirectives": true } }
}
```

We disabled formatter and linter in Biome (using Prettier and ESLint elsewhere), but kept it for Tailwind CSS directive parsing. The key insight is that `tailwindDirectives: true` is required for Biome to not choke on `@apply`, `@tailwind`, and `@theme` directives in CSS files.

> **Key insight:** When an AI agent is writing config files, it draws on training data that may be one major version behind. Biome v1 to v2 renamed fundamental keys. Always verify config key names against the current version's docs, not the agent's memory.

### How to Diagnose

```bash
# Check if biome.json is valid JSON
cat biome.json | python3 -c "import sys,json; json.load(sys.stdin); print('Valid JSON')"

# Check if biome can parse it
npx biome check --config-path biome.json --no-errors-on-unmatched .

# Check installed version vs schema version
cat biome.json | grep schema
cat node_modules/@biomejs/biome/package.json | grep version
```

---

## Problem 2: Claude Code Agent SDK Model Names

**Severity:** 404 error. Agent cannot start.

**When it happened:** Wiring up the CLI to spawn Haiku subagents for design generation.

### The Symptom

```
Error: 404 Not Found
  model: claude-3-5-haiku-20241022
```

### What Went Wrong

The [`@anthropic-ai/claude-code`](https://www.npmjs.com/package/@anthropic-ai/claude-code) package (now called [`@anthropic-ai/claude-agent-sdk`](https://www.npmjs.com/package/@anthropic-ai/claude-agent-sdk)) exports a `query()` function. It accepts a `model` option:

```typescript
import { query } from "@anthropic-ai/claude-code";

const stream = query({
  prompt: "Generate a wireframe",
  options: {
    model: "haiku",  // <-- THIS DOES NOT WORK
  },
});
```

The SDK resolves short names internally. But the resolution is not what you expect:

| You pass | SDK resolves to | Result |
|----------|----------------|--------|
| `"haiku"` | `claude-3-5-haiku-20241022` | **404** -- model not available |
| `"sonnet"` | `claude-sonnet-4-5-20250929` | Works (sometimes) |
| `"claude-haiku-4-5"` | `claude-haiku-4-5` | Works |
| `"claude-sonnet-4-6"` | `claude-sonnet-4-6` | Works |
| `"claude-opus-4-6"` | `claude-opus-4-6` | Works |

The problem: `"haiku"` resolves to an older model identifier that no longer exists on the API. Short aliases are unreliable because they change meaning as new model versions ship.

### The Fix

Always use full model identifiers:

```typescript
function resolveModel(model: string): string {
  const map: Record<string, string> = {
    haiku: "claude-haiku-4-5",
    sonnet: "claude-sonnet-4-6",
    opus: "claude-opus-4-6",
  };
  return map[model] || model;
}

const stream = query({
  prompt: fullPrompt,
  options: {
    model: resolveModel("haiku"),  // "claude-haiku-4-5"
  },
});
```

### The API Shape Surprise

The `query()` function signature from the [official TypeScript SDK reference](https://platform.claude.com/docs/en/agent-sdk/typescript):

```typescript
function query({
  prompt,
  options,
}: {
  prompt: string | AsyncIterable<SDKUserMessage>;
  options?: Options;
}): Query;
```

The `Options` type includes:

```typescript
interface Options {
  model?: string;                    // Full model ID
  systemPrompt?: string | {          // System prompt config
    type: 'preset';
    preset: 'claude_code';
    append?: string;
  };
  maxTurns?: number;                 // Max agentic turns
  tools?: string[] | {              // Tool configuration
    type: 'preset';
    preset: 'claude_code';
  };
  permissionMode?: PermissionMode;
  // ... many more options
}
```

Note: there IS a `systemPrompt` option in the current SDK. In earlier versions of the package (`@anthropic-ai/claude-code` before it was renamed to `@anthropic-ai/claude-agent-sdk`), the `systemPrompt` option either did not exist or behaved differently. If you are on an older version and `systemPrompt` is not working, the workaround is to prepend your system context to the `prompt` string:

```typescript
const fullPrompt = `${systemPrompt}\n\n---\n\n${userPrompt}`;

const stream = query({
  prompt: fullPrompt,
  options: { model: "claude-haiku-4-5" },
});
```

This is what we did in production. It works reliably across all SDK versions.

> **Key insight:** Model name resolution in SDKs is a moving target. Short aliases like "haiku" resolve to specific model versions at SDK build time, not at call time. Pin your model IDs explicitly. When a 404 comes back, the first thing to check is what model string actually hit the API.

### How to Diagnose

```typescript
// Add logging before the query call
const model = resolveModel(opts.model);
console.log(`Spawning agent with model: ${model}`);

// If using the Agent SDK, check supported models
const q = query({ prompt: "test", options: { model } });
const init = await q.initializationResult();
console.log("Supported models:", init.models);
```

---

## Problem 3: Haiku Subagent Rewrote Code with the Wrong SDK

**Severity:** Code compiles but crashes at runtime with mysterious type errors.

**When it happened:** Using Haiku (via Claude Code) as a subagent to implement the design agent harness.

### The Symptom

After letting Haiku implement the agent harness, the code looked reasonable:

```typescript
// What Haiku wrote:
import { Anthropic } from '@anthropic-ai/claude-code';

export async function runDesignAgent(brief: string, opts: { model: string }) {
  const client = new Anthropic();

  const response = await client.messages.create({
    model: opts.model,
    max_tokens: 4096,
    system: systemPrompt,
    messages: [{ role: 'user', content: brief }],
  });

  return response.content[0].text;
}
```

This is the **wrong SDK API**. The code imports from `@anthropic-ai/claude-code` but uses the API pattern from `@anthropic-ai/sdk`. These are completely different packages:

| | `@anthropic-ai/sdk` | `@anthropic-ai/claude-code` |
|---|---|---|
| **What it is** | Standard Anthropic API client | Claude Code Agent SDK |
| **Main export** | `new Anthropic()` constructor | `query()` function |
| **API pattern** | `client.messages.create({...})` | `query({ prompt, options })` |
| **Returns** | `Message` object | `AsyncGenerator<SDKMessage>` |
| **Has tools** | You define them | Built-in (file read, bash, edit) |
| **Streaming** | Optional via `.stream()` | Always streams (async generator) |

Haiku knew the `@anthropic-ai/sdk` API perfectly (it is well-represented in training data). It did not know the `@anthropic-ai/claude-code` API (newer, less training data). So when asked to "implement the agent harness," it reached for the familiar pattern.

The import line `import { Anthropic } from '@anthropic-ai/claude-code'` does not even work -- `@anthropic-ai/claude-code` does not export an `Anthropic` class. But the error only surfaces at runtime because TypeScript `any` types and dynamic imports can mask the mismatch at compile time.

### What We Tried

1. **Asked Haiku to fix it.** Haiku doubled down on the same pattern, trying variations of `new Anthropic()` with different import paths.
2. **Provided the correct API in the prompt.** Haiku acknowledged it but still produced `client.messages.create()` code in the implementation.
3. **Human rescue.** We rewrote the harness ourselves using the correct `query()` API.

### The Fix

The correct implementation:

```typescript
import { query } from "@anthropic-ai/claude-code";

export async function runDesignAgent(
  brief: string,
  opts: { model: string; domain?: string }
): Promise<DesignResult> {
  const model = resolveModel(opts.model);
  const systemPrompt = buildSystemPrompt({ brief, patterns });
  const fullPrompt = `${systemPrompt}\n\n---\n\n${brief}`;

  const queryStream = query({
    prompt: fullPrompt,
    options: { model },
  });

  let responseText = "";
  for await (const message of queryStream) {
    if (message.type === "assistant") {
      const textContent = message.message.content
        .map((c: { type: string; text?: string }) =>
          c.type === "text" ? c.text : ""
        )
        .join("");
      responseText += textContent;
    }
    if (message.type === "result") {
      console.log(`Agent finished (${message.subtype}). Cost: $${message.total_cost_usd?.toFixed(4)}`);
    }
  }

  // Parse YAML from response
  const yamlText = extractYaml(responseText);
  const parsed = parseYaml(yamlText);
  return ScreenplaySchema.parse(parsed);
}
```

The key differences from what Haiku wrote:

1. Import `query`, not `Anthropic`
2. Call `query({ prompt, options })`, not `client.messages.create()`
3. Iterate the async generator with `for await...of`, not await a single response
4. Handle `message.type === "assistant"` and `message.type === "result"` events
5. System prompt goes in the prompt string (or use `systemPrompt` option on newer SDK versions)

### The Lesson

> **Key insight:** For domain-specific APIs that are not well-represented in training data, do not let cheap/fast models improvise the implementation. The task plan must include the exact code patterns -- import statements, function signatures, and usage examples. If the model does not have the API in its weights, no amount of prompting will make it produce correct code.

This is a general rule for AI agent architectures: **the smaller the model, the more specific the plan must be.** Opus or Sonnet can often figure out an unfamiliar API from docs. Haiku will default to the closest familiar pattern, which may be a completely different library.

### Prevention Pattern

When using an AI agent to implement code with a niche API:

```typescript
// BAD: vague instruction
const prompt = "Implement the agent harness using @anthropic-ai/claude-code";

// GOOD: include the exact API shape
const prompt = `
Implement the agent harness. The API is:

import { query } from "@anthropic-ai/claude-code";

const stream = query({
  prompt: "the user message",
  options: {
    model: "claude-haiku-4-5",
  },
});

for await (const message of stream) {
  if (message.type === "assistant") {
    // message.message.content is an array of { type: "text", text: string }
  }
  if (message.type === "result") {
    // message.total_cost_usd is the cost
  }
}

Use EXACTLY this pattern. Do NOT use new Anthropic() or client.messages.create().
`;
```

---

## Problem 4: TanStack Start + "React is not defined"

**Severity:** Blocks all client-side hydration. The page loads server-rendered HTML but nothing is interactive.

**When it happened:** After scaffolding the TanStack Start app and running `vite dev`.

### The Symptom

Open the browser. See the server-rendered HTML. Open DevTools console:

```
Uncaught ReferenceError: React is not defined
  at client.tsx:3:27
```

The page renders but nothing hydrates. No click handlers, no state, no interactivity.

### What Went Wrong

TanStack Start generates a virtual client entry file internally. When you run the dev server, Vite serves this generated file at a URL like `/@id/__x00__virtual:/@tanstack/react-start/client-runtime`. The generated code looks like this:

```tsx
import { StrictMode, startTransition } from "react";
import { hydrateRoot } from "react-dom/client";

startTransition(() => {
  hydrateRoot(
    document,
    <StrictMode>
      <App />
    </StrictMode>
  );
});
```

The file uses JSX (`<StrictMode>`, `<App />`). Vite's default esbuild transform compiles JSX to `React.createElement()` calls -- the "classic" JSX runtime. But the file only imports `{ StrictMode, startTransition }` from React. There is no `import React from "react"` statement.

Result:

```javascript
// After esbuild classic transform:
import { StrictMode, startTransition } from "react";

startTransition(() => {
  hydrateRoot(
    document,
    React.createElement(StrictMode, null,  // <-- React is not defined!
      React.createElement(App, null)
    )
  );
});
```

The variable `React` was never assigned. The named imports (`StrictMode`, `startTransition`) work fine, but `React.createElement` requires the default import.

### What We Tried

1. **Adding `import React from "react"` to our files.** Does not help -- the problem is in the *virtual* entry file that TanStack Start generates. We cannot edit it.
2. **Using `@vitejs/plugin-react`.** This plugin adds the automatic JSX transform, but TanStack Start's `tanstackStart()` Vite plugin conflicts with it. You get duplicate React transformation errors.
3. **Searching the TanStack Start docs.** Nothing about this issue. The [build from scratch guide](https://tanstack.com/start/latest/docs/framework/react/build-from-scratch) does not mention it.

### The Fix

One line in `vite.config.ts`:

```typescript
import tailwindcss from '@tailwindcss/vite'
import { tanstackStart } from '@tanstack/react-start/plugin/vite'
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [tailwindcss(), tanstackStart()],
  esbuild: {
    jsx: 'automatic',  // <-- THIS FIXES IT
  },
})
```

Setting `esbuild.jsx` to `'automatic'` switches Vite's built-in esbuild JSX transform from the classic `React.createElement` pattern to the new automatic runtime. The compiled output becomes:

```javascript
import { jsx as _jsx } from "react/jsx-dev-runtime";
import { StrictMode, startTransition } from "react";

startTransition(() => {
  hydrateRoot(
    document,
    _jsx(StrictMode, {
      children: _jsx(App, {})
    })
  );
});
```

The `_jsx` function is imported from `react/jsx-dev-runtime` automatically. No `React` variable needed.

### Why This is Not in the Docs

TanStack Start moved from [Vinxi](https://github.com/nksaraf/vinxi) to native Vite in late 2024 (see the [LogRocket migration guide](https://blog.logrocket.com/migrating-tanstack-start-vinxi-vite/)). During this migration, the virtual client entry generation was rewritten. The assumption was that developers would use `@vitejs/plugin-react` (which sets up automatic JSX). But `@vitejs/plugin-react` conflicts with `tanstackStart()` because both try to transform JSX.

The fix is to set esbuild's JSX mode directly, bypassing the React plugin entirely. This is a Vite-level config, not a TanStack config, which is probably why it is not in TanStack's docs.

### How We Found It

We traced the error by fetching the virtual entry URL directly:

```bash
# In the browser, the error pointed to client.tsx
# We fetched the transformed version from Vite's dev server:
curl http://localhost:3000/@id/__x00__virtual:/@tanstack/react-start/client-runtime
```

This showed us the compiled JavaScript with `React.createElement` calls. Once we saw that, the fix was obvious: switch to automatic JSX.

> **Key insight:** When a framework generates virtual files, bugs in those files cannot be fixed by editing your own code. You need to change the build configuration so the *transform* produces correct output. In this case, the transform (esbuild classic JSX) was the problem, not the source code.

### Diagnostic Steps for Anyone Hitting This

```bash
# 1. Check if the error is in a virtual file
# Look at the error stack trace -- if the file path contains "virtual" or "@id", it's generated

# 2. Check your vite.config.ts for jsx settings
grep -r "jsx" vite.config.ts

# 3. Check if you have conflicting React plugins
grep -r "plugin-react" package.json

# 4. The fix
# Add to vite.config.ts defineConfig:
# esbuild: { jsx: 'automatic' }
```

---

## Problem 5: tldraw + SSR Framework = Crash

**Severity:** Server process crashes on startup. Nothing renders.

**When it happened:** Adding tldraw to the TanStack Start app.

### The Symptom

```
ReferenceError: window is not defined
    at /node_modules/@tldraw/tldraw/dist/cjs/index.js:42:15
```

Or:

```
ReferenceError: document is not defined
    at /node_modules/@tldraw/editor/dist/cjs/index.js:118:22
```

The server crashes immediately when importing tldraw.

### What Went Wrong

tldraw is a canvas library. It accesses browser APIs (`window`, `document`, `navigator`, `CanvasRenderingContext2D`) at **module import time** -- not just when components render. This means the mere act of `import { Tldraw } from '@tldraw/tldraw'` executes code that references `window`.

TanStack Start does server-side rendering. When the server processes your route components, it imports their dependencies. If a route component imports tldraw (directly or transitively), the server process tries to access `window` and crashes.

This is not a tldraw-specific problem. It affects any browser-only library with side effects at import time: Monaco Editor, Leaflet, Three.js, PDF.js, and dozens of others. It is a fundamental tension between SSR frameworks and browser-only code.

### What We Tried

**Attempt 1: Dynamic import in the component**

```tsx
// DOES NOT WORK
import { useState, useEffect } from 'react';

export function Canvas() {
  const [Tldraw, setTldraw] = useState(null);

  useEffect(() => {
    import('@tldraw/tldraw').then(mod => setTldraw(() => mod.Tldraw));
  }, []);

  if (!Tldraw) return <div>Loading...</div>;
  return <Tldraw />;
}
```

This helps but has issues: TypeScript cannot infer the Tldraw component type, and the CSS import (`@tldraw/tldraw/tldraw.css`) still needs to be handled.

**Attempt 2: `typeof window` guard**

```tsx
// DOES NOT WORK
if (typeof window !== 'undefined') {
  const { Tldraw } = await import('@tldraw/tldraw');
}
```

Top-level await is not supported in all module contexts, and this pattern is fragile in bundler environments.

**Attempt 3: `'use client'` directive alone**

```tsx
'use client'

import { Tldraw } from '@tldraw/tldraw';
// Still crashes -- 'use client' tells the framework this is a client component,
// but it does NOT prevent the server from importing the module during SSR.
```

The `'use client'` directive marks a component boundary, but it does not prevent SSR of that component. The server still renders it (to produce HTML), which means it still imports tldraw.

### The Fix: Two-File Pattern

The solution requires two separate files:

**File 1: `CanvasWrapper.tsx`** (imported by routes, always loads)

```tsx
import { lazy, Suspense } from 'react'

// React.lazy() ensures the import happens ONLY on the client
const CanvasContent = lazy(() => import('./CanvasContent'))

export function Canvas() {
  return (
    <Suspense fallback={
      <div style={{
        width: '100%',
        height: '100%',
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        color: '#999'
      }}>
        Loading tldraw...
      </div>
    }>
      <CanvasContent />
    </Suspense>
  )
}
```

**File 2: `CanvasContent.tsx`** (only loads on the client, has `'use client'`)

```tsx
'use client'

import { type Editor, Tldraw } from '@tldraw/tldraw'
import { useCallback } from 'react'
import '@tldraw/tldraw/tldraw.css'

export default function CanvasContent() {
  const handleMount = useCallback((editor: Editor) => {
    editor.user.updateUserPreferences({ colorScheme: 'dark' })
  }, [])

  return (
    <div style={{ width: '100%', height: '100%', position: 'absolute', inset: 0 }}>
      <Tldraw onMount={handleMount} />
    </div>
  )
}
```

**File 3: Route component** (imports only the wrapper)

```tsx
import { createFileRoute } from '@tanstack/react-router'
import { Suspense, useEffect, useState } from 'react'
import { Canvas } from '../components/CanvasWrapper'

export const Route = createFileRoute('/')({
  component: StudioHome,
})

function StudioHome() {
  const [isClient, setIsClient] = useState(false)
  useEffect(() => { setIsClient(true) }, [])

  return (
    <div className="flex flex-col h-screen">
      <main className="flex-1 relative">
        {isClient ? (
          <Suspense fallback={<LoadingPlaceholder />}>
            <Canvas />
          </Suspense>
        ) : (
          <LoadingPlaceholder />
        )}
      </main>
    </div>
  )
}
```

### Why This Works

The chain of imports on the **server**:

```
Route → CanvasWrapper.tsx → (lazy() does NOT import CanvasContent.tsx)
```

`React.lazy()` creates a lazy reference. It does not execute the `import()` until the component actually renders in the browser. On the server, the `<Suspense>` boundary catches the lazy component and renders the fallback instead. tldraw is never imported on the server.

The chain of imports on the **client**:

```
Route → CanvasWrapper.tsx → (lazy triggers) → CanvasContent.tsx → @tldraw/tldraw
```

The `isClient` state guard in the route component adds a belt-and-suspenders check: even if `React.lazy` behaves unexpectedly during hydration, the tldraw component is not rendered until `useEffect` fires (which only happens on the client).

### Cross-Framework Comparison

This pattern works across SSR frameworks. The equivalent in other frameworks:

| Framework | tldraw SSR Solution |
|-----------|-------------------|
| **TanStack Start** | `React.lazy()` + `'use client'` (as shown above) |
| **Next.js** | `dynamic(() => import('./Canvas'), { ssr: false })` |
| **Remix** | `React.lazy()` + `ClientOnly` wrapper from `remix-utils` |
| **SvelteKit** | `{#if browser}` conditional + dynamic `import()` in `onMount` |
| **Nuxt** | `<ClientOnly>` component wrapper |

The tldraw community [recommends the lazy-load pattern](https://tldraw.dev/quick-start) across all frameworks.

> **Key insight:** The two-file pattern is not just a workaround -- it is the correct architecture for any browser-only library in an SSR framework. File 1 (wrapper) lives in the SSR world. File 2 (implementation) lives in the browser world. The `lazy()` call is the bridge between them. Do not try to cram both worlds into one file.

---

## Problem 6: tldraw v3.10+ API Breaking Change -- `text` to `richText`

**Severity:** Shapes fail to create. Existing shapes in localStorage crash on reload.

**When it happened:** Creating shapes programmatically via the tldraw Editor API.

### The Symptom

```
ValidationError: At shape(type = geo).props.text: Unexpected property
```

Or, after "fixing" the code but not clearing the browser:

```
ValidationError: At shape(type = text).props.text: Unexpected property
```

The second error is the cached invalid shapes replaying from localStorage.

### What Changed

In [tldraw v3.10.0](https://tldraw.dev/releases/v3.10.0), the `text` property on most shapes was replaced with `richText`. This is a **breaking change** documented in the release notes as:

> "text property on most shapes replaced with richText"

The old API:

```typescript
// BROKEN in v3.10+
editor.createShape({
  type: 'geo',
  x: 100,
  y: 100,
  props: {
    w: 200,
    h: 100,
    geo: 'rectangle',
    text: 'Hello world',  // <-- no longer accepted
  },
})

editor.createShape({
  type: 'text',
  x: 100,
  y: 200,
  props: {
    text: 'Some text',  // <-- no longer accepted
  },
})
```

The new API requires a ProseMirror-style document structure:

```typescript
// WORKS in v3.10+
editor.createShape({
  type: 'geo',
  x: 100,
  y: 100,
  props: {
    w: 200,
    h: 100,
    geo: 'rectangle',
    richText: {
      type: 'doc',
      content: [
        {
          type: 'paragraph',
          content: [
            { type: 'text', text: 'Hello world' }
          ]
        }
      ]
    },
  },
})
```

### The richText Format

The `richText` property expects a [ProseMirror](https://prosemirror.net/) document structure. Under the hood, tldraw uses [TipTap](https://tiptap.dev/) (which wraps ProseMirror) for text editing. The format:

```typescript
interface RichTextDoc {
  type: 'doc';
  content: ParagraphNode[];
}

interface ParagraphNode {
  type: 'paragraph';
  content?: TextNode[];  // undefined = empty line
}

interface TextNode {
  type: 'text';
  text: string;
  marks?: Mark[];  // bold, italic, etc.
}

interface Mark {
  type: 'bold' | 'italic' | 'code' | 'link';
  attrs?: Record<string, unknown>;
}
```

### The Helper Function

tldraw exports a `toRichText()` helper, but if you need to handle multiline text or want to avoid the import, here is a standalone converter:

```typescript
/** Convert a plain string to tldraw v3.10+ richText format */
function rt(text: string) {
  return {
    type: 'doc' as const,
    content: text.split('\n').map((line) =>
      line
        ? { type: 'paragraph' as const, content: [{ type: 'text' as const, text: line }] }
        : { type: 'paragraph' as const }
    ),
  }
}
```

Usage:

```typescript
function box(
  editor: Editor,
  x: number,
  y: number,
  w: number,
  h: number,
  color = 'grey',
  fill = 'none',
  dash = 'solid',
  label = ''
) {
  const props: any = { w, h, geo: 'rectangle', color, fill, dash, size: 's' }
  if (label) props.richText = rt(label)
  editor.createShape({ type: 'geo', x, y, props })
}

function txt(editor: Editor, x: number, y: number, text: string, color = 'white') {
  editor.createShape({
    type: 'text',
    x,
    y,
    props: { richText: rt(text), color, size: 's' },
  })
}
```

### The localStorage Cache Trap

Even after fixing the code, tldraw caches shapes in `localStorage` under a persistence key. If you created invalid shapes before fixing the code, those invalid shapes are still in the cache. On page reload, tldraw tries to restore them and hits the validation error again.

**Fix:**

```javascript
// In browser console:
localStorage.clear()

// Or more targeted:
Object.keys(localStorage)
  .filter(k => k.startsWith('tldraw'))
  .forEach(k => localStorage.removeItem(k))
```

In production, you can also set `persistenceKey={undefined}` on the `<Tldraw>` component to disable localStorage persistence entirely:

```tsx
<Tldraw persistenceKey={undefined} onMount={handleMount} />
```

This is what we did during development -- no persistence while iterating on shapes.

### Shapes Affected

Both built-in shape types that accept text:

| Shape Type | Old Property | New Property | Notes |
|-----------|-------------|-------------|-------|
| `geo` | `props.text` | `props.richText` | Rectangles, ellipses, arrows, etc. with labels |
| `text` | `props.text` | `props.richText` | Standalone text shapes |
| `note` | `props.text` | `props.richText` | Sticky note shapes |
| `arrow` | `props.text` | `props.richText` | Arrow labels |

If you have custom shapes that extend these or use text properties, they need the same migration.

### Using the Official Helper

If you prefer to use tldraw's built-in helper:

```typescript
import { toRichText, renderPlaintextFromRichText } from '@tldraw/tldraw'

// Writing text to a shape
editor.createShape({
  type: 'text',
  x: 100,
  y: 100,
  props: { richText: toRichText('Hello world') },
})

// Reading text from a shape
const shape = editor.getShape(shapeId)
const plainText = renderPlaintextFromRichText(shape.props.richText)
```

> **Key insight:** When a canvas library switches from plain text to a rich text document model (ProseMirror/TipTap), the migration is not just a rename. The data structure changes from `string` to a nested object tree. This means every place you create shapes, read shape text, or serialize shapes needs to change. And because localStorage caches the old format, you get ghost errors that persist even after fixing the code.

---

## Problem 7: Vite HMR Breaks with Rapid File Rewrites

**Severity:** Dev server becomes unrecoverable. Requires full restart.

**When it happened:** Whenever Claude Code (the AI agent) made multiple rapid file edits.

### The Symptom

The browser console fills with:

```
[vite] hmr update /src/routes/index.tsx
TypeError: Cannot read properties of undefined (reading 'routesById')
    at router.ts:1847:32
```

Or:

```
[vite] hmr invalidate /src/routeTree.gen.ts
[vite] hmr update /src/routes/__root.tsx
Uncaught Error: Route "/" does not exist
```

The page goes blank. Refreshing does not help. Even hard-refresh (Cmd+Shift+R) does not help. The HMR state is corrupted in memory.

### What Went Wrong

AI coding agents rewrite files at machine speed. A typical agentic edit session might:

1. Write `CanvasWrapper.tsx` (new file)
2. Write `CanvasContent.tsx` (new file)
3. Rewrite `routes/index.tsx` (modify existing)
4. Rewrite `routes/index.tsx` again (fix an error)
5. Write `styles.css` (add tldraw import)

Steps 3 and 4 happen within milliseconds. Vite's file watcher sees the first change, starts HMR processing, then sees the second change before the first is complete. This is especially destructive with TanStack Router because:

1. TanStack Router watches route files and regenerates `routeTree.gen.ts`
2. `routeTree.gen.ts` is a generated file that maps route paths to route modules
3. While TanStack Router is regenerating the route tree, Vite is trying to HMR the old route tree
4. The old route tree references modules that may have moved or been renamed
5. `router.routesById` becomes `undefined` because the route tree is in an inconsistent state

### The Cascade

```
File write (by AI agent)
  → Vite file watcher triggers
    → TanStack Router code-gen starts regenerating routeTree.gen.ts
      → Vite HMR processes the first file change
        → Second file write happens (by AI agent, 50ms later)
          → Vite file watcher triggers again
            → routeTree.gen.ts is mid-write
              → HMR tries to hot-update with partial route tree
                → "Cannot read properties of undefined (reading 'routesById')"
                  → Client-side router state is now corrupt
                    → No recovery without full restart
```

### What We Tried

1. **Throttling file writes.** The AI agent does not have a "write slowly" mode.
2. **Disabling HMR.** Setting `server.hmr: false` in Vite config works but removes the main benefit of a dev server.
3. **Using `autoCodeSplitting: false`** in TanStack Router config. [Helps for some HMR issues](https://github.com/TanStack/router/issues/3561) but does not fix the rapid-rewrite race condition.
4. **Running `vite --debug hmr`** to log the exact sequence. Confirmed the race condition but did not help fix it.

### The (Non-)Fix

There is no real fix. This is a fundamental impedance mismatch between:
- AI agents that write files as fast as possible
- File watchers that process changes incrementally
- Code generators (TanStack Router) that need a stable filesystem snapshot

**The workflow that actually works:**

```bash
# 1. Let the AI agent finish all its edits
# 2. Kill the dev server
kill -9 $(lsof -ti:3000)

# 3. Clear Vite's cache
rm -rf node_modules/.vite

# 4. Restart the dev server
pnpm dev

# 5. Hard refresh the browser (Cmd+Shift+R)
```

We do this after every major AI agent editing session. It takes 5 seconds and avoids 10 minutes of debugging phantom HMR errors.

### Mitigation for CI/CD

In CI, this is not a problem because you run `vite build`, not `vite dev`. HMR is only a dev-time concern.

For development, consider these mitigations:

```typescript
// vite.config.ts — increase the HMR debounce
export default defineConfig({
  server: {
    watch: {
      // Increase polling interval to reduce race conditions
      // Only helps if the AI agent is slightly slower
      interval: 500,
    },
  },
})
```

Or, if you are frequently working with AI agents:

```bash
# Run the dev server in "restart on crash" mode
while true; do pnpm dev; sleep 1; done
```

> **Key insight:** Vite HMR assumes files change at human speed -- one edit at a time, seconds apart. AI agents violate this assumption. The result is not a bug in Vite or TanStack Router; it is an architectural mismatch. Plan your dev workflow around "edit, then restart" rather than "edit and rely on HMR."

---

## Patterns: What Breaks at the Intersections

Every problem we hit was at the intersection of two tools. No individual tool was broken. The bugs emerged from incompatible assumptions:

```
+-------------------+-------------------+----------------------------+
| Tool A            | Tool B            | What Broke                 |
+-------------------+-------------------+----------------------------+
| Biome v2          | AI Agent          | Agent wrote v1 config keys |
| Agent SDK         | Model API         | Short names resolve wrong  |
| Haiku model       | Agent SDK         | Familiar SDK != right SDK  |
| TanStack Start    | Vite esbuild      | Classic JSX + no React     |
| tldraw            | TanStack Start    | Browser-only + SSR         |
| tldraw v3.10+     | localStorage      | Old cached shapes + new API|
| Vite HMR          | AI Agent          | Machine-speed file writes  |
+-------------------+-------------------+----------------------------+
```

### The Meta-Pattern

When combining cutting-edge tools, the problems cluster at the boundaries:

1. **Config mismatches** -- tools evolve independently, breaking config compatibility
2. **Assumption violations** -- Tool A assumes X (human-speed edits, browser environment, specific JSX transform), Tool B violates that assumption
3. **Training data lag** -- AI agents know the training-data version of APIs, not the installed version
4. **Cache poisoning** -- state cached by one tool becomes invalid when another tool changes behavior

### A Rule of Thumb

If you are combining N tools, expect bugs proportional to N*(N-1)/2 -- the number of pairwise intersections. Three tools (tldraw + TanStack Start + AI agents) gives 3 potential intersection bugs. We hit all three and then some, because each tool has multiple surfaces (config, runtime, dev server, caching).

---

## The Complete Working Setup

For reference, here is the complete working configuration after solving all seven problems.

### vite.config.ts

```typescript
import tailwindcss from '@tailwindcss/vite'
import { tanstackStart } from '@tanstack/react-start/plugin/vite'
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [tailwindcss(), tanstackStart()],
  esbuild: {
    jsx: 'automatic',  // Fix: "React is not defined" in virtual entry
  },
})
```

### biome.json

```json
{
  "$schema": "https://biomejs.dev/schemas/2.4.8/schema.json",
  "files": {
    "includes": ["**/*.ts", "**/*.tsx", "**/*.js", "**/*.jsx", "**/*.json", "**/*.css", "**/*.md"]
  },
  "formatter": { "enabled": false },
  "linter": { "enabled": false },
  "css": { "parser": { "tailwindDirectives": true } }
}
```

### CanvasWrapper.tsx

```tsx
import { lazy, Suspense } from 'react'

const CanvasContent = lazy(() => import('./CanvasContent'))

export function Canvas() {
  return (
    <Suspense fallback={
      <div style={{
        width: '100%', height: '100%',
        display: 'flex', alignItems: 'center', justifyContent: 'center',
        color: '#999'
      }}>
        Loading tldraw...
      </div>
    }>
      <CanvasContent />
    </Suspense>
  )
}
```

### CanvasContent.tsx

```tsx
'use client'

import { type Editor, Tldraw } from '@tldraw/tldraw'
import { useCallback } from 'react'
import '@tldraw/tldraw/tldraw.css'

function rt(text: string) {
  return {
    type: 'doc' as const,
    content: text.split('\n').map((line) =>
      line
        ? { type: 'paragraph' as const, content: [{ type: 'text' as const, text: line }] }
        : { type: 'paragraph' as const }
    ),
  }
}

export default function CanvasContent() {
  const handleMount = useCallback((editor: Editor) => {
    editor.user.updateUserPreferences({ colorScheme: 'dark' })

    // Example: create shapes with richText (not text!)
    editor.createShape({
      type: 'geo',
      x: 100,
      y: 100,
      props: {
        w: 300,
        h: 200,
        geo: 'rectangle',
        color: 'blue',
        fill: 'semi',
        richText: rt('Hello from tldraw'),
      },
    })

    editor.createShape({
      type: 'text',
      x: 100,
      y: 350,
      props: {
        richText: rt('This uses richText, not text'),
        color: 'white',
        size: 's',
      },
    })

    editor.zoomToFit({ animation: { duration: 400 } })
  }, [])

  return (
    <div style={{ width: '100%', height: '100%', position: 'absolute', inset: 0 }}>
      <Tldraw persistenceKey={undefined} onMount={handleMount} />
    </div>
  )
}
```

### Agent Harness (using Agent SDK correctly)

```typescript
import { query } from "@anthropic-ai/claude-code";

function resolveModel(model: string): string {
  const map: Record<string, string> = {
    haiku: "claude-haiku-4-5",
    sonnet: "claude-sonnet-4-6",
    opus: "claude-opus-4-6",
  };
  return map[model] || model;
}

export async function runDesignAgent(
  brief: string,
  opts: { model: string }
) {
  const model = resolveModel(opts.model);
  const systemPrompt = buildSystemPrompt({ brief, patterns: [] });
  const fullPrompt = `${systemPrompt}\n\n---\n\n${brief}`;

  const stream = query({
    prompt: fullPrompt,
    options: { model },
  });

  let responseText = "";
  for await (const message of stream) {
    if (message.type === "assistant") {
      responseText += message.message.content
        .map((c: { type: string; text?: string }) =>
          c.type === "text" ? c.text : ""
        )
        .join("");
    }
    if (message.type === "result") {
      console.log(`Done. Cost: $${message.total_cost_usd?.toFixed(4)}`);
    }
  }

  return responseText;
}
```

---

## Comparison: SSR Frameworks + Canvas Libraries

If you are choosing a stack for a similar project (AI-powered visual tool with SSR), here is how the options compare:

### SSR Frameworks

| Framework | tldraw Integration | JSX Config | HMR Stability | AI Agent Compat | Learning Curve |
|-----------|-------------------|------------|---------------|-----------------|----------------|
| **TanStack Start** | Needs `React.lazy()` + `'use client'` + `esbuild.jsx: 'automatic'` | Manual config needed | Fragile with rapid writes | Good (Vite-based) | Medium |
| **Next.js** | `dynamic(() => import(...), { ssr: false })` — first-class support | Works out of box | More stable (Turbopack) | Good | Low |
| **Remix** | `React.lazy()` + `ClientOnly` from remix-utils | Works out of box | Moderate | Good (Vite-based) | Medium |
| **Astro** | `client:only="react"` directive — cleanest DX | N/A (islands) | Stable (no full-page HMR) | Good | Low |
| **SvelteKit** | `{#if browser}` conditional | N/A (Svelte) | Stable | Good (Vite-based) | Medium (Svelte) |

### Canvas / Drawing Libraries

| Library | SSR-Safe? | Text Model | React Support | AI Shape Generation | Bundle Size |
|---------|-----------|------------|---------------|-------------------|-------------|
| **tldraw** | No (needs lazy-load) | richText (ProseMirror) | First-class | Good (Editor API) | ~400KB |
| **Excalidraw** | No (needs lazy-load) | Plain string | First-class | Good (API) | ~300KB |
| **Konva / react-konva** | No (needs lazy-load) | Plain string | Good | Moderate | ~150KB |
| **Fabric.js** | No (needs lazy-load) | Plain string | Community wrapper | Moderate | ~300KB |
| **Rough.js + SVG** | Yes (SSR-safe) | N/A (SVG text) | Manual | Good (just SVG) | ~15KB |
| **Canvas API (raw)** | No | N/A | Manual | Poor (imperative) | 0KB |

### AI Agent SDKs

| SDK | What It Is | API Pattern | Built-in Tools | Cost Tracking | Best For |
|-----|-----------|-------------|----------------|---------------|----------|
| **Claude Agent SDK** (`@anthropic-ai/claude-agent-sdk`) | Full agent framework | `query({ prompt, options })` async generator | File read/write, bash, web search | Yes (`total_cost_usd`) | Autonomous agents with file/code access |
| **Anthropic SDK** (`@anthropic-ai/sdk`) | API client | `client.messages.create({...})` | None (you build them) | Manual (token counting) | Direct API calls, custom tool systems |
| **Vercel AI SDK** (`ai`) | Multi-provider abstraction | `generateText()`, `streamText()` | Via tool definitions | Via callbacks | Multi-provider apps, streaming UI |
| **LangChain** | Agent framework | `agent.invoke()` | Via tool classes | Via callbacks | Complex chains, RAG |
| **OpenAI Assistants** | Managed agents | `client.beta.threads.create()` | Code interpreter, retrieval | Via run object | OpenAI ecosystem |

> **Our recommendation:** For a visual tool with AI agents, use **Next.js** (most stable tldraw integration, best HMR) or **Astro** (cleanest client-only component model). We used TanStack Start because we wanted to learn it, but it required more manual configuration than necessary. For the AI agent, the Claude Agent SDK is the right choice if you need file and code manipulation -- the standard SDK if you just need text generation.

---

## Anti-Patterns

Mistakes we made or almost made, collected here so you do not repeat them.

| Don't | Do Instead | Why |
|-------|-----------|-----|
| Use short model names like `"haiku"` with the Agent SDK | Use full model IDs: `"claude-haiku-4-5"` | Short names resolve to outdated model versions that return 404 |
| Let a cheap model implement code with niche SDK APIs | Include exact code patterns in the prompt, or use a larger model | Haiku defaults to the most-familiar API pattern, which may be the wrong SDK |
| Import tldraw directly in a route component | Use the two-file `React.lazy()` + `'use client'` pattern | tldraw accesses `window` at import time, crashing SSR |
| Use `text` property on tldraw shapes (v3.10+) | Use `richText` with ProseMirror document format | The `text` property was removed; shapes fail validation |
| Fix tldraw shape code without clearing localStorage | Always clear localStorage (or disable persistence) when changing shape schemas | Cached invalid shapes replay on load, causing the same error |
| Rely on Vite HMR during AI agent editing sessions | Kill dev server and restart after agent finishes editing | Rapid file writes cause unrecoverable HMR state corruption |
| Use `@vitejs/plugin-react` with `tanstackStart()` | Use `esbuild: { jsx: 'automatic' }` directly | The two plugins conflict on JSX transformation |
| Use Biome v1 config key names (`include`) with Biome v2 | Use `includes` (v2 key name), verify against installed version | Biome v2 silently ignores `include`, applying rules to all files |
| Trust AI agent output for config files without verification | Always validate config syntax before committing | AI agents can write `undefined`, use wrong schema versions, or mix API versions |
| Put system prompt in a `systemPrompt` option on older SDK versions | Prepend system context to the `prompt` string | Older `@anthropic-ai/claude-code` versions may not support `systemPrompt` |
| Use `persistenceKey` during development with changing shape schemas | Set `persistenceKey={undefined}` while iterating | Avoids localStorage cache poisoning when shape props change |
| Assume `'use client'` prevents server-side import | Combine with `React.lazy()` to actually prevent import | `'use client'` marks a boundary but does not prevent SSR rendering |

---

## Small Examples

### Example 1: Minimal tldraw in TanStack Start

The absolute minimum to get tldraw rendering in a TanStack Start app:

```typescript
// vite.config.ts
import { tanstackStart } from '@tanstack/react-start/plugin/vite'
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [tanstackStart()],
  esbuild: { jsx: 'automatic' },
})
```

```tsx
// src/components/TldrawLazy.tsx
import { lazy, Suspense } from 'react'
const Inner = lazy(() => import('./TldrawInner'))
export const TldrawLazy = () => (
  <Suspense fallback={<div>Loading...</div>}>
    <Inner />
  </Suspense>
)
```

```tsx
// src/components/TldrawInner.tsx
'use client'
import { Tldraw } from '@tldraw/tldraw'
import '@tldraw/tldraw/tldraw.css'
export default function TldrawInner() {
  return <div style={{ width: '100vw', height: '100vh' }}><Tldraw /></div>
}
```

### Example 2: Creating Shapes with richText Programmatically

```typescript
import { type Editor } from '@tldraw/tldraw'

function richText(text: string) {
  return {
    type: 'doc' as const,
    content: text.split('\n').map(line =>
      line ? { type: 'paragraph' as const, content: [{ type: 'text' as const, text: line }] }
           : { type: 'paragraph' as const }
    ),
  }
}

function createLabeledBox(editor: Editor, x: number, y: number, label: string) {
  editor.createShape({
    type: 'geo',
    x, y,
    props: {
      w: 200, h: 100,
      geo: 'rectangle',
      color: 'blue',
      fill: 'semi',
      richText: richText(label),
    },
  })
}

function createTextNote(editor: Editor, x: number, y: number, text: string) {
  editor.createShape({
    type: 'text',
    x, y,
    props: {
      richText: richText(text),
      color: 'yellow',
      size: 'm',
    },
  })
}
```

### Example 3: Reading Shape Text Back

```typescript
import { renderPlaintextFromRichText } from '@tldraw/tldraw'

function getAllTextFromCanvas(editor: Editor): string[] {
  const shapes = editor.getCurrentPageShapes()
  const texts: string[] = []

  for (const shape of shapes) {
    const props = shape.props as any
    if (props.richText) {
      texts.push(renderPlaintextFromRichText(props.richText))
    }
  }

  return texts
}
```

### Example 4: Agent SDK Query with Error Handling

```typescript
import { query } from "@anthropic-ai/claude-code";

async function askAgent(prompt: string): Promise<string> {
  const stream = query({
    prompt,
    options: {
      model: "claude-haiku-4-5",
      maxTurns: 1,  // No tool use, just answer
    },
  });

  let result = "";
  for await (const msg of stream) {
    switch (msg.type) {
      case "assistant":
        result += msg.message.content
          .filter((c: any) => c.type === "text")
          .map((c: any) => c.text)
          .join("");
        break;
      case "result":
        if (msg.subtype === "error_max_turns") {
          throw new Error("Agent exceeded max turns");
        }
        break;
    }
  }
  return result;
}
```

### Example 5: Safe Model Resolution

```typescript
const MODEL_MAP: Record<string, string> = {
  haiku: "claude-haiku-4-5",
  sonnet: "claude-sonnet-4-6",
  opus: "claude-opus-4-6",
} as const;

function resolveModel(input: string): string {
  const resolved = MODEL_MAP[input.toLowerCase()];
  if (resolved) return resolved;

  // Validate it looks like a full model ID
  if (!input.startsWith("claude-")) {
    throw new Error(
      `Unknown model "${input}". Use a full model ID (e.g., "claude-haiku-4-5") ` +
      `or a shorthand: ${Object.keys(MODEL_MAP).join(", ")}`
    );
  }
  return input;
}
```

### Example 6: Detecting SSR vs Client in TanStack Start

```tsx
import { useState, useEffect, type ReactNode } from 'react'

function ClientOnly({ children, fallback }: { children: ReactNode; fallback?: ReactNode }) {
  const [mounted, setMounted] = useState(false)
  useEffect(() => setMounted(true), [])

  if (!mounted) return fallback ?? null
  return children
}

// Usage in a route:
function MyPage() {
  return (
    <div>
      <h1>Works on server and client</h1>
      <ClientOnly fallback={<div>Loading canvas...</div>}>
        <Canvas />
      </ClientOnly>
    </div>
  )
}
```

### Example 7: Biome Config Validation Script

```bash
#!/bin/bash
# validate-biome.sh — run before commits to catch config corruption

BIOME_JSON="biome.json"

# Check file exists and is not empty
if [ ! -s "$BIOME_JSON" ]; then
  echo "ERROR: $BIOME_JSON is empty or missing"
  exit 1
fi

# Check it's valid JSON (not "undefined" or other garbage)
if ! python3 -c "import json; json.load(open('$BIOME_JSON'))" 2>/dev/null; then
  echo "ERROR: $BIOME_JSON is not valid JSON"
  echo "Contents: $(head -1 $BIOME_JSON)"
  exit 1
fi

# Check for v1 key names in v2
if grep -q '"include"' "$BIOME_JSON" && ! grep -q '"includes"' "$BIOME_JSON"; then
  echo "WARNING: biome.json uses 'include' (v1) instead of 'includes' (v2)"
fi

# Check schema version matches installed version
SCHEMA_VER=$(grep -o 'schemas/[0-9.]*' "$BIOME_JSON" | head -1 | cut -d/ -f2)
INSTALLED_VER=$(node -e "console.log(require('@biomejs/biome/package.json').version)" 2>/dev/null)
if [ -n "$SCHEMA_VER" ] && [ -n "$INSTALLED_VER" ] && [ "$SCHEMA_VER" != "$INSTALLED_VER" ]; then
  echo "WARNING: Schema version ($SCHEMA_VER) != installed version ($INSTALLED_VER)"
fi

echo "biome.json looks valid"
```

### Example 8: Vite Config for Maximum HMR Stability

```typescript
import tailwindcss from '@tailwindcss/vite'
import { tanstackStart } from '@tanstack/react-start/plugin/vite'
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [tailwindcss(), tanstackStart()],
  esbuild: {
    jsx: 'automatic',
  },
  server: {
    watch: {
      // Increase poll interval when working with AI agents
      // that write files rapidly
      interval: 300,
      // Ignore generated files to reduce churn
      ignored: ['**/routeTree.gen.ts', '**/node_modules/**'],
    },
    hmr: {
      // Overlay helps see errors without opening DevTools
      overlay: true,
    },
  },
})
```

### Example 9: tldraw richText with Bold and Italic Marks

```typescript
function richTextWithMarks(segments: Array<{
  text: string;
  bold?: boolean;
  italic?: boolean;
  code?: boolean;
}>) {
  return {
    type: 'doc' as const,
    content: [{
      type: 'paragraph' as const,
      content: segments.map(seg => {
        const marks: Array<{ type: string }> = []
        if (seg.bold) marks.push({ type: 'bold' })
        if (seg.italic) marks.push({ type: 'italic' })
        if (seg.code) marks.push({ type: 'code' })

        return {
          type: 'text' as const,
          text: seg.text,
          ...(marks.length > 0 ? { marks } : {}),
        }
      }),
    }],
  }
}

// Usage:
editor.createShape({
  type: 'text',
  x: 100,
  y: 100,
  props: {
    richText: richTextWithMarks([
      { text: 'Bold text', bold: true },
      { text: ' and ' },
      { text: 'italic text', italic: true },
    ]),
    color: 'white',
    size: 'm',
  },
})
```

### Example 10: Complete Dev Server Restart Script

```bash
#!/bin/bash
# restart-dev.sh — nuclear option for when HMR is broken

PORT=${1:-3000}
APP_DIR=${2:-.}

echo "Killing processes on port $PORT..."
lsof -ti:$PORT | xargs kill -9 2>/dev/null

echo "Clearing Vite cache..."
rm -rf "$APP_DIR/node_modules/.vite"
rm -rf "$APP_DIR/.vinxi"  # TanStack Start legacy

echo "Clearing tldraw localStorage (reminder: do this in browser too)"
echo "  Run in browser console: localStorage.clear()"

echo "Restarting dev server..."
cd "$APP_DIR" && pnpm dev &

echo "Waiting for server..."
sleep 3
echo "Done. Hard refresh (Cmd+Shift+R) your browser."
```

---

## Comparison: Where Each Tool Excels and Fails

### tldraw

| Aspect | Assessment |
|--------|-----------|
| **Canvas DX** | Excellent. Best React canvas SDK available. Full shape system, selection, zoom, collaboration out of the box. |
| **SSR compatibility** | Poor. Requires workarounds in every SSR framework. |
| **Text handling** | Good after v3.10. richText (ProseMirror) is more powerful than plain strings, but migration is painful. |
| **Programmatic shape creation** | Excellent. The `Editor` API is comprehensive and well-typed. |
| **Documentation** | Good for basics, sparse for advanced (programmatic shape creation, custom shapes, richText format). |
| **Bundle size** | Large (~400KB). Acceptable for a full canvas app, problematic for a widget. |

### TanStack Start

| Aspect | Assessment |
|--------|-----------|
| **Type safety** | Excellent. Full end-to-end type safety for routes, loaders, actions. |
| **Vite integration** | Good but fragile. The virtual entry file JSX issue is a sharp edge. |
| **SSR** | Works well once configured. The `esbuild.jsx: 'automatic'` fix is not obvious. |
| **File-based routing** | Good. TanStack Router's code-gen is powerful but adds HMR fragility. |
| **Documentation** | Improving but incomplete. Missing common gotchas like the JSX config. |
| **Maturity** | Early. Moving fast, breaking things. The Vinxi-to-Vite migration changed internals. |

### Claude Code Agent SDK

| Aspect | Assessment |
|--------|-----------|
| **Agent capabilities** | Excellent. File read/write, bash execution, web search built in. |
| **API ergonomics** | Good. `query()` async generator is clean. |
| **Model management** | Rough edges. Short model names are unreliable. |
| **Cost tracking** | Excellent. `total_cost_usd` on every result message. |
| **Documentation** | Good. [TypeScript reference](https://platform.claude.com/docs/en/agent-sdk/typescript) is comprehensive. |
| **AI agent integration** | Unique. No other SDK gives you "Claude Code in a box" programmatically. |

### Biome

| Aspect | Assessment |
|--------|-----------|
| **Speed** | Excellent. Fastest JS/TS linter and formatter. |
| **v1 to v2 migration** | Painful. Key renames, glob engine changes, silent failures. |
| **AI agent compatibility** | Poor. Config format is not well-known to AI models (small training data footprint). |
| **Tailwind CSS support** | Good with `tailwindDirectives: true`. |
| **Error messages** | Acceptable. Config errors could be more specific about what is wrong. |

---

## Timeline of Discovery

For context, here is the order we hit these problems. The commit log tells the story:

```
ad91b0e chore: add husky, commitlint, CLAUDE.md
  ↓ (biome config corruption begins)
88c6a98 chore: update biome.json           # content: "undefined"
  ... 30+ biome update commits ...
6185ef6 chore: update biome.json           # finally valid

b0c893f feat: scaffold Frontasy Studio     # TanStack Start scaffolding
  ↓ "React is not defined" — fixed with esbuild.jsx: 'automatic'

a677f87 feat(cli): agent harness           # Agent SDK integration
796fb10 feat(cli): design + bank-add       # Haiku rewrites harness with wrong SDK
90818e7 fix(cli): restore agent harness    # Human rescue

05753fd feat: fix tldraw SSR rendering     # First SSR fix attempt
4743a5b fix(studio): fix tldraw SSR error  # Working two-file pattern
38e3d2c feat(studio): tldraw canvas        # richText format + localStorage fix
```

---

## Closing Thoughts

Seven problems. All at intersections. None of them are documented in any single tool's docs because they require two tools to manifest.

The meta-lesson: **when you combine cutting-edge tools, budget time for intersection debugging.** The individual tools are solid. The gaps are between them. A "simple" task like "render a tldraw canvas in a TanStack Start app" touches JSX compilation, SSR hydration, module import side effects, browser API access, and localStorage caching. Each of those is a potential failure point.

For AI-assisted development specifically: AI agents are excellent at writing code for well-established APIs. They struggle with niche SDKs (Claude Code Agent SDK), recent breaking changes (tldraw richText), and tool-specific config formats (Biome v2). The higher the tool's novelty, the more you need to hand-hold the agent with exact code patterns.

The best workflow we found: **let the AI agent draft, verify at every intersection, and keep a restart script handy for when HMR falls over.**

---

## References

### Official Documentation

- [tldraw Quick Start](https://tldraw.dev/quick-start) -- getting started guide, includes SSR notes
- [tldraw v3.10.0 Release Notes](https://tldraw.dev/releases/v3.10.0) -- documents the `text` to `richText` breaking change
- [tldraw Default Shapes](https://tldraw.dev/sdk-features/default-shapes) -- shape types and their properties
- [tldraw RichTextArea Reference](https://tldraw.dev/reference/tldraw/RichTextArea) -- rich text component API
- [tldraw Rich Text Custom Extension Example](https://tldraw.dev/examples/rich-text-custom-extension) -- extending rich text with TipTap
- [TanStack Start Build from Scratch](https://tanstack.com/start/latest/docs/framework/react/build-from-scratch) -- official setup guide (missing JSX config)
- [TanStack Router Vite Installation](https://tanstack.com/router/latest/docs/framework/react/routing/installation-with-esbuild) -- esbuild configuration reference
- [Claude Agent SDK Overview](https://platform.claude.com/docs/en/agent-sdk/overview) -- what the Agent SDK is and when to use it
- [Claude Agent SDK TypeScript Reference](https://platform.claude.com/docs/en/agent-sdk/typescript) -- complete API reference for `query()`, `Options`, message types
- [Claude Agent SDK Migration Guide](https://platform.claude.com/docs/en/agent-sdk/migration-guide) -- migrating from `@anthropic-ai/claude-code` to `@anthropic-ai/claude-agent-sdk`
- [Biome v2 Upgrade Guide](https://biomejs.dev/guides/upgrade-to-biome-v2/) -- `include` to `includes` migration, glob engine changes
- [Biome v2 Announcement](https://biomejs.dev/blog/biome-v2/) -- what changed and why
- [Vite Troubleshooting Guide](https://vite.dev/guide/troubleshooting) -- HMR debugging, common issues

### GitHub Issues and PRs

- [tldraw #5131: Replace text property with richText](https://github.com/tldraw/tldraw/pull/5131) -- the PR that made the breaking change
- [tldraw #4895: Rich text experiment](https://github.com/tldraw/tldraw/pull/4895) -- original rich text spike
- [tldraw #5657: Add toPlainText helper](https://github.com/tldraw/tldraw/issues/5657) -- community request for text extraction
- [tldraw #5715: Mass rich text updates example](https://github.com/tldraw/tldraw/pull/5715) -- bulk shape text updates
- [TanStack Router #902: Vite HMR breaks the router](https://github.com/TanStack/router/issues/902) -- HMR instability reports
- [TanStack Router #3561: AutoCodeSplitting stops HMR](https://github.com/TanStack/router/issues/3561) -- code splitting + HMR conflict
- [TanStack Router #3917: Vite HMR fails, app reloaded](https://github.com/TanStack/router/issues/3917) -- another HMR failure mode
- [TanStack Router #2793: Getting started guide missing import](https://github.com/TanStack/router/issues/2793) -- React import issue in docs
- [Biome #6272: biome migrate includes fail for subdirectories](https://github.com/biomejs/biome/issues/6272) -- migration tool bug
- [Biome #6713: Tool-specific includes stops working](https://github.com/biomejs/biome/issues/6713) -- includes behavior issue in v2

### Blog Posts and Articles

- [Migrating TanStack Start from Vinxi to Vite](https://blog.logrocket.com/migrating-tanstack-start-vinxi-vite/) -- LogRocket guide on the framework migration
- [What's New in tldraw -- March 2025](https://tldraw.dev/blog/whats-new-2025) -- recent tldraw updates including rich text improvements
- [Building Agents with the Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk) -- Anthropic engineering blog on agent patterns
- [Biome Roadmap 2025](https://biomejs.dev/blog/roadmap-2025/) -- Biome's direction and v2 planning
- [Stop "Window Is Not Defined" in Next.js](https://dev.to/devin-rosario/stop-window-is-not-defined-in-nextjs-2025-394j) -- general SSR + browser API guide

### NPM Packages

- [@anthropic-ai/claude-agent-sdk](https://www.npmjs.com/package/@anthropic-ai/claude-agent-sdk) -- Claude Agent SDK (new name)
- [@anthropic-ai/sdk](https://www.npmjs.com/package/@anthropic-ai/sdk) -- Standard Anthropic API client (different package!)
- [@tldraw/tldraw](https://www.npmjs.com/package/@tldraw/tldraw) -- tldraw canvas SDK
- [@biomejs/biome](https://www.npmjs.com/package/@biomejs/biome) -- Biome linter/formatter

### Related Technologies

- [ProseMirror](https://prosemirror.net/) -- the document model underlying tldraw's richText format
- [TipTap](https://tiptap.dev/) -- the headless editor framework tldraw uses for rich text editing
- [Vite esbuild options](https://vite.dev/config/shared-options#esbuild) -- where to configure `jsx: 'automatic'`
