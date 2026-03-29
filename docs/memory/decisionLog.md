# Decision Log

Key architectural and technology decisions, each verified against source code.

> Sources: `package.json`, `tsconfig.json`, `Dockerfile`, `vercel.json`, `packages/eslintrc.base.json`,
> `packages/excalidraw/editor-jotai.ts`, `excalidraw-app/app-jotai.ts`,
> `excalidraw-app/collab/Collab.tsx`, `excalidraw-app/data/index.ts`,
> `scripts/buildPackage.js`, `.github/workflows/`

---

## 1. Monorepo with Yarn Classic Workspaces

**Decision:** Use a single Yarn Classic (v1) workspace monorepo for the app, library packages, and examples.

**Rationale:**
- Enables local cross-package development without publishing to npm first.
- Shared `devDependencies` at root reduce duplication (`typescript`, `eslint`, `prettier`, `vitest`).
- Vite path aliases (`@excalidraw/*`) allow the app to consume library source directly during dev.

**Evidence:**
- `package.json` `"workspaces": ["excalidraw-app", "packages/*", "examples/*"]`
- `"packageManager": "yarn@1.22.22"`
- `tsconfig.json` `paths` mapping every `@excalidraw/*` to local `packages/*/src/index.ts`

---

## 2. Strict Unidirectional Package Dependency Graph

**Decision:** Enforce a strict one-way dependency order: `common → math → element → excalidraw → app`.
Lower-level packages must never import from higher-level ones.

**Rationale:**
- Prevents circular dependencies and tight coupling between reusable primitives and the full library.
- Keeps `@excalidraw/common`, `math`, and `element` publishable as independent packages.

**Evidence:**
- `packages/eslintrc.base.json` — ESLint rule `@typescript-eslint/no-restricted-imports` blocks any import of
  `@excalidraw/excalidraw` from lower packages (value imports disallowed; type-only imports allowed):
  ```json
  "group": ["../../excalidraw", "../../../packages/excalidraw", "@excalidraw/excalidraw"],
  "allowTypeImports": true
  ```
- `packages/*/package.json` `dependencies` fields confirm no upward references.

---

## 3. TypeScript Strict Mode + ESM Throughout

**Decision:** Enable `"strict": true` and `"module": "ESNext"` across the entire codebase. All packages use `"type": "module"`.

**Rationale:**
- Strict mode catches null/undefined errors and unsafe type coercions at compile time.
- ESM-first output aligns with modern browser capabilities and Vite's native ESM dev server.
- `"noEmit": true` means `tsc` is used purely as a type-checker; esbuild/Vite handle transpilation.

**Evidence:**
- `tsconfig.json`:
  - `"strict": true`, `"noEmit": true`, `"module": "ESNext"`, `"isolatedModules": true`
  - `"jsx": "react-jsx"` — no manual `import React` required
- Root `package.json` `"type": "module"` propagated to all workspace packages.

---

## 4. Dual Bundler Strategy: Vite for App, esbuild for Library

**Decision:** Use Vite 5 to build `excalidraw-app`, and esbuild (via `scripts/buildPackage.js`) to produce dual CJS + ESM builds for each `@excalidraw/*` package.

**Rationale:**
- Vite provides HMR, PWA plugin, and chunked static output ideal for a web app.
- esbuild is faster than Rollup/Webpack for library builds that must emit clean ESM and CJS bundles for npm consumers.
- Keeps library build entirely independent of the app build pipeline.

**Evidence:**
- `package.json` scripts: `"build:packages"` runs `node scripts/buildPackage.js` for each package.
- `excalidraw-app/vite.config.mts` configures `@vitejs/plugin-react`, PWA, and SVGR.
- `packages/excalidraw/package.json` `exports` field ships both `module` (ESM) and `main` (CJS) entry points.

---

## 5. Jotai with Scoped Isolation for the Embeddable Library

**Decision:** Use two separate Jotai stores — an isolated `editorJotaiStore` for the library and a global store for the app layer. Use `jotai-scope` to prevent state leakage between multiple embedded instances.

**Rationale:**
- Third-party apps may embed multiple `<Excalidraw>` instances on the same page; a shared global store would cause state collisions.
- `jotai-scope`'s `createIsolation()` gives each library instance its own atom context.

**Evidence:**
- `packages/excalidraw/editor-jotai.ts`: `createIsolation()` → `EditorJotaiProvider`, `editorJotaiStore`.
- `excalidraw-app/app-jotai.ts` uses plain Jotai atoms (`collabAPIAtom`, `isCollaboratingAtom`) in the outer app store.

---

## 6. No Server-Side State — Serverless + Firebase

**Decision:** There is no custom Node API server. All backend interactions go through Firebase SDK (Firestore + Storage) and direct HTTP calls to collaboration/AI endpoints.

**Rationale:**
- Minimises operational overhead — no servers to provision or maintain for core functionality.
- Firebase Firestore provides real-time sync natively, matching the collaboration use-case.
- Collab infrastructure can be self-hosted by replacing environment variables.

**Evidence:**
- `excalidraw-app/data/index.ts` — all persistence logic calls Firebase SDK directly.
- `firebase-project/firebase.json` — defines Firestore and Storage rules; no Cloud Functions configured.
- `systemPatterns.md`: "No Node API server — backend interactions go through Firebase SDK and HTTP endpoints directly from the browser."

---

## 7. End-to-End Encryption for Collaboration and Share Links

**Decision:** All shared scenes and collaboration payloads are encrypted client-side using the Web Crypto API. The encryption key is embedded in the URL fragment (`#room=id,key` / `#json=id,key`) and never sent to the server.

**Rationale:**
- The server (Firebase / backend) stores only ciphertext; it cannot read scene content.
- Zero-knowledge encryption is a strong privacy guarantee for a public whiteboard tool.
- URL fragment (`#`) is not sent by browsers in HTTP requests, ensuring the key stays client-only.

**Evidence:**
- `excalidraw-app/data/index.ts` — `SyncableExcalidrawElement` branded type gates what can be synced.
- `packages/excalidraw/data/` — Web Crypto encryption utilities.
- `productContext.md`: "Encryption by default for collaboration and share links; keys never leave the client in plaintext."

---

## 8. Offline-First with PWA + IndexedDB

**Decision:** Treat the network as an enhancement, not a dependency. Auto-save every scene to IndexedDB; register a service worker for offline PWA support.

**Rationale:**
- Users lose work if the app requires network to save. Auto-save to IndexedDB ensures zero data loss on disconnect.
- PWA install via `vite-plugin-pwa` makes the tool behave like a native desktop app on supported platforms.

**Evidence:**
- `excalidraw-app/package.json` `"idb-keyval": "6.0.3"` — IndexedDB wrapper.
- `excalidraw-app/vite.config.mts` — `VitePWA()` plugin registration.
- `excalidraw-app/index.tsx` — service worker registered at app entry point.

---

## 9. Multi-Stage Docker Build: Node → nginx

**Decision:** Use a two-stage `Dockerfile`: Stage 1 builds the app with `node:18`; Stage 2 copies the static output to `nginx:1.27-alpine` for serving.

**Rationale:**
- The final Docker image contains no Node.js runtime — only nginx and static files, reducing image size and attack surface.
- The build cache layer (`--mount=type=cache`) speeds up CI rebuilds.

**Evidence:**
- `Dockerfile`: Stage 1 `node:18 AS build` → Stage 2 `nginx:1.27-alpine`; static files copied to `/usr/share/nginx/html`.

---

## 10. Compound Component API for the Embeddable Library

**Decision:** Expose the library as a compound component tree, not a monolithic `<Excalidraw options={...} />` props API. Consumers render named sub-components (`<MainMenu>`, `<Footer>`, `<WelcomeScreen>`) as children.

**Rationale:**
- Compound components give consumers fine-grained control over which UI surfaces appear without exposing a sprawling props surface.
- Slot-based composition is idiomatic React and works naturally with tree-shaking.

**Evidence:**
- `packages/excalidraw/index.tsx` exports `MainMenu`, `Footer`, `WelcomeScreen`, `LiveCollaborationTrigger` alongside the default `App` component.
- `examples/with-nextjs/` and `examples/with-script-in-browser/` demonstrate consumers composing sub-components.

---

## 11. Semantic PR Titles Enforced by CI

**Decision:** All pull requests must use conventional commit style titles. CI rejects PRs with non-conforming titles.

**Rationale:**
- Conventional commits power automated changelogs and the `autorelease-excalidraw.yml` workflow.
- Consistent PR titles make git history navigable without reading commit bodies.

**Evidence:**
- `.github/workflows/semantic-pr-title.yml` — runs on `pull_request` events.
- `.github/workflows/autorelease-excalidraw.yml` — automated release driven by commit conventions.

---

## 12. React 19 Adopted; Library Peer Deps Support 17–19

**Decision:** The app and library implementation target React 19, but `@excalidraw/excalidraw` declares peer dependencies of `react ^17 || ^18 || ^19`.

**Rationale:**
- Allows the library to be adopted by projects on React 17 or 18 without forcing an upgrade.
- The app itself can benefit from React 19 features (concurrent rendering, server actions) while the npm package remains broadly compatible.

**Evidence:**
- Root `package.json`: `"react": "19.0.0"`, `"react-dom": "19.0.0"`.
- `packages/excalidraw/package.json` `peerDependencies`: `"react": "^17 || ^18 || ^19"`.

---

## 13. Undocumented behavior vs project documentation (audit)

The items below were found by scanning `HACK` / `FIXME` / `TODO` / `WORKAROUND`-style comments and tracing **implicit state**, **side effects**, and **initialization order** in source. For each: **what the code actually does**, **what project docs imply or state**, and **the gap**.

### 13.1 Library Jotai state vs multiple embedded instances

**What the code does:** `editor-jotai.ts` uses `createIsolation()` and a dedicated `editorJotaiStore` so editor atoms are scoped per provider instance. On `Library.destroy()` (`packages/excalidraw/data/library.ts`), the code clears `libraryItemSvgsCache` on that store but **leaves commented-out** reset logic for `libraryItemsAtom`, with an explicit `TODO`: *"uncomment after/if we make jotai store scoped to each excal instance"* — implying not all library-related atoms were considered fully isolated when that was written.

**What is documented:** `decisionLog.md` §5 and `systemPatterns.md` describe **isolated Jotai** and `jotai-scope` as preventing leakage between multiple `<Excalidraw>` instances.

**Gap:** Documentation presents a clean isolation story; the in-code TODO records an **unfinished or partial** reset path for library atoms on unmount, so behavior under rapid mount/unmount or multiple instances may diverge from the documented guarantee until that path is resolved.

### 13.2 Export pipeline: “Scene hack,” synthetic frame labels, and stable IDs

**What the code does:** Frame names shown in the DOM while editing are **not** drawn the same way in export. `addFrameLabelsAsTextElements` in `packages/excalidraw/scene/export.ts` injects **temporary text elements** so frames export with visible titles. Separately, export uses a **temp scene** path that can **duplicate elements and regenerate ids** (`tempScene hack` comments in the same file). For PNG embed (`exportEmbedScene`), `packages/utils/src/export.ts` insists on passing **original, uncloned** elements into `serializeAsJSON` *"as long as we're using the Scene hack"* so embedded metadata keeps **stable ids**.

**What is documented:** `decisionLog.md` §7–8 and PRD export sections describe **encryption**, **formats**, and **offline** behavior, not the **two-phase** export model (synthetic elements + id-preserving serialization).

**Gap:** Consumers and maintainers only learn the **ordering and cloning constraints** from inline comments; the public mental model “export = render what you see” is **incomplete** for frames and embedded scenes.

### 13.3 Test harness: global `window.h` and `initialScene` loading gate

**What the code does:** `packages/excalidraw/tests/test-utils.ts` waits until canvases exist **and** `window.h.state.isLoading` is false, with a comment calling this a **"hack-awaiting app.initialScene()"** to fix **race conditions** (prefer a proper event later).

**What is documented:** Test patterns are not described in `docs/memory/` or architecture docs; `architecture.md` lists `isLoading` as an `AppState` field but not **test coupling** to `window.h`.

**Gap:** Tests depend on a **global handle** and **implicit ordering** between render and scene initialization—undocumented contract for anyone writing new integration tests.

### 13.4 Popover viewport fitting vs React StrictMode double effects

**What the code does:** `packages/excalidraw/components/Popover.tsx` uses `lastInitializedPosRef` so a `useLayoutEffect` **skips a second run** when `top`/`left` match, avoiding **double repositioning** under StrictMode (“hack for StrictMode”).

**What is documented:** No mention of StrictMode-specific layout workarounds in project docs.

**Gap:** **Implicit state machine** (first run vs skipped run) exists only in code; changing StrictMode or popover positioning logic can regress layout without guidance.

### 13.5 Public `Excalidraw` wrapper: `UIOptions` merge and memoization

**What the code does:** `packages/excalidraw/index.tsx` merges `UIOptions` with defaults inside the wrapper with a **`FIXME`**: defaults should be normalized in the parent so **memo comparison** sees consistent values.

**What is documented:** `decisionLog.md` §10 describes the **compound component** API; it does not mention **referential or shallow memo pitfalls** on the default-export wrapper’s options merging.

**Gap:** **Non-obvious re-render behavior** when passing partial `UIOptions`—documentation suggests composition, not this internal merge/memo hazard.

### 13.6 `useOutsideClick` and Radix UI portals

**What the code does:** `packages/excalidraw/hooks/useOutsideClick.ts` treats clicks on `[data-radix-portal]` or on `documentElement` when `body` has `pointer-events: none` as **inside** the UI, with a comment that this is a **"terrible hack"** that can mis-classify some radix popups.

**What is documented:** No documentation of **Radix-specific** pointer or portal assumptions.

**Gap:** **Non-local side effect:** outside-click behavior depends on **third-party DOM attributes and global body styles**, not only the passed ref—easy to break when swapping dialog libraries or modal behavior.

### 13.7 Mobile: linear elements and transform handles

**What the code does:** `packages/excalidraw/components/App.tsx` contains a **`HACK`** that **disables transform handles** for **linear elements** on **mobile** (and for 2-point lines) until a better UX exists, while prioritizing point dragging.

**What is documented:** PRD §3.1 lists selection and manipulation broadly; it does **not** state that **resize/transform handles are intentionally suppressed** on touch for line-like elements.

**Gap:** **Platform-dependent editing model** is behavior-by-comment, not user-facing or architectural documentation.

### 13.8 Durable `Store.scheduleCapture` call graph

**What the code does:** `packages/element/src/store.ts` marks `scheduleCapture()` as called from **many places** with a **`TODO`**: *"Suspicious… Seems error-prone."* This coordinates **undo/history** and delta emission—an **implicit contract** across the codebase.

**What is documented:** `systemPatterns` / PRD refer to undo/redo and durable changes at a **product** level, not which code paths must schedule capture.

**Gap:** **Initialization and mutation ordering** for history correctness is **distributed tacit knowledge**; regressions require reading call sites, not a single state diagram in docs.

### 13.9 Known test / utility debt (FYI)

**What the code does:** Examples: `packages/utils/tests/export.test.ts` — **`FIXME`** that `exportToSvg` no longer filters deleted elements; `packages/excalidraw/wysiwyg/textWysiwyg.test.tsx` — flaky test **FIXME**; `packages/utils/tests/withinBounds.test.ts` — **TODO** for more element types.

**What is documented:** Not tracked in memory docs.

**Gap:** Test expectations may **drift from production** behavior until those comments are addressed; useful for maintainers **only if they read source**.

## Details

- **Product:** [PRD](../product/PRD.md), [Domain glossary](../product/domain-glossary.md)
- **Technical:** [Architecture](../technical/architecture.md), [Dev setup](../technical/dev-setup.md)
