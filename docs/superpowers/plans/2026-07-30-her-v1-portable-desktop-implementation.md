# “她”V1 Portable Cross-Platform Desktop Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an installation-free personal desktop companion for Apple Silicon macOS and x64 Windows 10/11 whose personality, conversations, memories, relationship state, model configuration, and backups travel together in one portable `Yours` directory.

**Architecture:** Electron owns the trusted main process and exposes a narrow typed preload API to a React renderer. Markdown, JSON, and append-only JSONL files are authoritative; a `better-sqlite3` FTS index is derived and rebuildable. Every user message is durably appended before any model request, and all generated interpretations retain source message IDs.

**Tech Stack:** Node.js 24 LTS for development, Electron 43.2.0, Electron Forge 7.11.2, React 19.2.8, TypeScript 7.0.2, Zod 4.4.3, better-sqlite3 13.0.1, Vitest, Playwright 1.61.1, npm.

## Global Constraints

- The repository contains documentation only at plan start; no product code exists.
- The first release targets Apple Silicon macOS and x64 Windows 10/11.
- End users must not install Node.js, a database, Docker, Nginx, or a local server.
- macOS and Windows executables differ, but both consume the same Vault format.
- The renderer has `nodeIntegration: false`, `contextIsolation: true`, and `sandbox: true`.
- The renderer never receives the API key and never reads arbitrary files directly.
- Remote model URLs require HTTPS; HTTP is allowed only for `127.0.0.1`, `localhost`, or `::1`.
- Real `config/model.json`, Vault data, backups, portable exports, API keys, model URLs, prompts, and conversations must never enter Git or logs.
- V1 stores the API key as plaintext in `config/model.json` by explicit user choice.
- Markdown, JSON, and JSONL are authoritative; `.cache/index.sqlite` is disposable and rebuildable.
- All authoritative paths are relative to the portable root and use UTF-8 with LF output; readers accept CRLF.
- JSONL history is append-only and rotates monthly or at 64 MB, whichever occurs first.
- JSON and Markdown replacement uses same-directory temporary files followed by atomic rename.
- Every structured record has a UUID, `schemaVersion`, and ISO-8601 timestamp.
- User input is committed before a model request begins.
- Generated facts and interpretations cite source message IDs and use `quote`, `explicitFact`, or `inference`.
- V1 permits at most 10 onboarding memory seeds, 5 active narratives, and 3 candidate thoughts per reflection.
- One Vault has one writer. A second process may open only read-only or exit.
- V1 has no encryption, cloud backend, account, sync, mobile client, background notification, automatic updater, app-store release, voice, avatar, external-app ingestion, or agent actions.
- Each task follows red-green-refactor TDD and ends with a focused commit.

## File Map

```text
package.json                         dependency pins and developer commands
package-lock.json                    reproducible npm dependency graph
forge.config.ts                      portable macOS/Windows packaging
vite.main.config.ts                  Electron main-process bundle
vite.preload.config.ts               isolated preload bundle
vite.renderer.config.ts              React renderer bundle
tsconfig.json                        strict shared TypeScript settings
vitest.config.ts                     unit/integration test configuration
playwright.config.ts                 Electron end-to-end configuration
.gitignore                           generated, private, and portable artifacts
.github/workflows/verify.yml         macOS/Windows verification and packaging
src/
  main/
    main.ts                           Electron lifecycle and composition root
    create-window.ts                  secure BrowserWindow creation
    ipc/register-ipc.ts               typed IPC registration
    portable/
      portable-root.ts               manifest discovery
      vault-layout.ts                 relative path construction
      vault-initializer.ts            new Vault creation
      vault-lock.ts                   single-writer lease
    storage/
      atomic-file.ts                  atomic JSON/Markdown replacement
      jsonl-store.ts                  durable append and rotation
      schemas.ts                      persisted-record Zod schemas
    persona/
      persona-service.ts              fixed/growing/current persona layers
      defaults.ts                     initial Chinese persona documents
    state/
      state-repository.ts              current focus and hanging events
    model/
      connection-policy.ts            HTTPS/loopback policy
      model-config-repository.ts       plaintext portable config
      openai-compatible-gateway.ts     private model HTTP client
    chat/
      conversation-repository.ts      raw conversation authority
      chat-coordinator.ts              raw-first request orchestration
    memory/
      memory-repository.ts            three memory streams and lifecycle
      memory-retriever.ts             ranked relevant memories
    index/
      search-index.ts                 derived FTS schema and cursor
      index-rebuilder.ts               full rebuild from authority
    context/
      context-builder.ts              bounded model context assembly
    reflection/
      reflection-parser.ts             structured output validation
      reflection-service.ts            derived memory/thought creation
      reflection-scheduler.ts          end/30-minute eligibility
    narrative/
      narrative-repository.ts          max-five Markdown narratives
      correction-service.ts            append-only correction history
    proactive/
      timing-gate.ts                   relevance and restraint rules
      proactive-recall-service.ts      at-most-one app-open recall
    maintenance/
      integrity-service.ts             startup checks and repair report
      migration-service.ts             versioned forward migration
      backup-service.ts                daily/weekly Vault backups
      portable-export-service.ts       complete portable archive
      restore-service.ts               verified transactional restore
  preload/
    preload.ts                         narrow contextBridge surface
  shared/
    domain.ts                          stable domain types
    ipc-contract.ts                    renderer/main request-response types
    errors.ts                          serializable application errors
  renderer/
    index.html
    index.tsx
    App.tsx                            four-screen shell
    api.ts                             typed preload adapter
    styles.css
    onboarding/OnboardingScreen.tsx
    chat/ChatScreen.tsx
    chat/MessageList.tsx
    memory/MemoryScreen.tsx
    settings/SettingsScreen.tsx
tests/
  unit/                                pure domain and policy tests
  integration/                         filesystem/model/index tests
  e2e/portable-flow.spec.ts            packaged Electron acceptance flow
  fixtures/                            redacted deterministic fixtures
scripts/
  prepare-portable-root.mjs            assemble local portable directory
  verify-package.mjs                   inspect packaged artifact contents
README.md                              developer and personal-use guide
```

---

### Task 1: Secure Electron shell and reproducible toolchain

**Files:**
- Create: `package.json`
- Create: `package-lock.json`
- Create: `forge.config.ts`
- Create: `vite.main.config.ts`
- Create: `vite.preload.config.ts`
- Create: `vite.renderer.config.ts`
- Create: `tsconfig.json`
- Create: `vitest.config.ts`
- Create: `.gitignore`
- Create: `src/main/create-window.ts`
- Create: `src/main/main.ts`
- Create: `src/preload/preload.ts`
- Create: `src/renderer/index.html`
- Create: `src/renderer/index.tsx`
- Create: `src/renderer/App.tsx`
- Test: `tests/unit/create-window.test.ts`

**Interfaces:**
- Produces: `secureWebPreferences(preloadPath: string): WebPreferences`
- Produces: npm commands `start`, `typecheck`, `test`, `package:mac`, and `package:win`
- Consumes: no application interfaces

- [ ] **Step 1: Create the pinned package manifest**

Create `package.json` with private publication, Node 24, Electron Forge commands, React, Zod, and `better-sqlite3`. Pin exact versions rather than ranges. Include `@electron-forge/plugin-auto-unpack-natives` so the SQLite native module is unpacked from ASAR.

```json
{
  "name": "yours",
  "productName": "她",
  "version": "0.1.0",
  "private": true,
  "main": ".vite/build/main.js",
  "engines": { "node": ">=24 <25" },
  "scripts": {
    "start": "electron-forge start",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "package:mac": "electron-forge package --platform=darwin --arch=arm64",
    "package:win": "electron-forge package --platform=win32 --arch=x64",
    "verify": "npm run typecheck && npm test"
  },
  "dependencies": {
    "archiver": "8.0.0",
    "better-sqlite3": "13.0.1",
    "react": "19.2.8",
    "react-dom": "19.2.8",
    "zod": "4.4.3"
  },
  "devDependencies": {
    "@electron-forge/cli": "7.11.2",
    "@electron-forge/plugin-auto-unpack-natives": "7.11.2",
    "@electron-forge/plugin-fuses": "7.11.2",
    "@electron-forge/plugin-vite": "7.11.2",
    "@electron-forge/shared-types": "7.11.2",
    "@electron/fuses": "1.8.0",
    "@playwright/test": "1.61.1",
    "@testing-library/dom": "10.4.1",
    "@testing-library/react": "16.3.2",
    "@types/archiver": "6.0.3",
    "@types/better-sqlite3": "7.6.13",
    "@types/node": "24.0.15",
    "@types/react": "19.2.2",
    "@types/react-dom": "19.2.2",
    "@vitejs/plugin-react": "5.0.0",
    "electron": "43.2.0",
    "jsdom": "29.1.1",
    "typescript": "7.0.2",
    "vite": "7.1.0",
    "vitest": "4.0.17"
  }
}
```

- [ ] **Step 2: Install exactly and generate the lockfile**

Run:

```bash
npm install --save-exact
```

Expected: `package-lock.json` is created, `npm ls --depth=0` exits 0, and no real Vault or model configuration is created.

- [ ] **Step 3: Write the failing secure-window test**

```ts
import { describe, expect, it } from "vitest";
import { secureWebPreferences } from "../../src/main/create-window";

describe("secureWebPreferences", () => {
  it("isolates the renderer from Node and the API key", () => {
    expect(secureWebPreferences("/tmp/preload.js")).toMatchObject({
      preload: "/tmp/preload.js",
      nodeIntegration: false,
      contextIsolation: true,
      sandbox: true
    });
  });
});
```

- [ ] **Step 4: Run the test and verify red**

Run: `npm test -- tests/unit/create-window.test.ts`

Expected: FAIL because `src/main/create-window.ts` does not exist.

- [ ] **Step 5: Implement the secure shell**

```ts
import type { WebPreferences } from "electron";

export function secureWebPreferences(preloadPath: string): WebPreferences {
  return {
    preload: preloadPath,
    nodeIntegration: false,
    contextIsolation: true,
    sandbox: true
  };
}
```

Create `BrowserWindow` only in `src/main/main.ts`, deny unexpected navigation and new windows, and load the Forge Vite URL in development or packaged renderer file in production.

- [ ] **Step 6: Add strict TypeScript, Vite, Forge, Fuse, and ignore configuration**

Set `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, and `useUnknownInCatchVariables`. Configure Forge with ASAR, native-module unpacking, Vite entry points, and fuses that disable `RunAsNode` and enable cookie encryption. Ignore:

```gitignore
node_modules/
.vite/
out/
coverage/
test-results/
.DS_Store
config/model.json
Vault/
portable-exports/
*.yours-portable.zip
```

Configure Vitest with Node as the default environment and `environmentMatchGlobs` so `tests/unit/**/*screen.test.tsx` uses `jsdom`. Keep filesystem and main-process tests in Node.

- [ ] **Step 7: Verify and commit**

Run:

```bash
npm run verify
npm run start
```

Expected: tests and typecheck pass; a window titled “她” opens without renderer Node access.

```bash
git add package.json package-lock.json forge.config.ts vite.*.config.ts tsconfig.json vitest.config.ts .gitignore src tests/unit/create-window.test.ts
git commit -m "build: scaffold secure Electron desktop shell"
```

---

### Task 2: Versioned domain records and IPC contract

**Files:**
- Create: `src/shared/domain.ts`
- Create: `src/shared/errors.ts`
- Create: `src/shared/ipc-contract.ts`
- Create: `src/main/storage/schemas.ts`
- Test: `tests/unit/domain-schemas.test.ts`

**Interfaces:**
- Produces: `MessageRecord`, `MemoryRecord`, `NarrativeDocument`, `CorrectionRecord`, `ThoughtRecord`, `ModelConfig`, `VaultManifest`
- Produces: Zod schemas with `schemaVersion: 1`
- Produces: `YoursAPI` renderer contract
- Consumes: Zod 4.4.3

- [ ] **Step 1: Write failing schema tests**

Test that a valid user message parses, an inference without `sourceMessageIds` fails, and a record without `schemaVersion` fails.

```ts
expect(() => memoryRecordSchema.parse({
  id: crypto.randomUUID(),
  schemaVersion: 1,
  createdAt: new Date().toISOString(),
  type: "understanding",
  content: "你可能更担心走错方向",
  sourceNature: "inference",
  sourceMessageIds: []
})).toThrow();
```

- [ ] **Step 2: Run the test and verify red**

Run: `npm test -- tests/unit/domain-schemas.test.ts`

Expected: FAIL because the schemas are undefined.

- [ ] **Step 3: Define exact discriminated records**

Use string UUIDs and ISO timestamps. Define:

```ts
export type SourceNature = "quote" | "explicitFact" | "inference";
export type MemoryType = "event" | "relationship" | "understanding";
export type RecordStatus = "active" | "deep" | "sealed";

export interface MessageRecord {
  id: string;
  schemaVersion: 1;
  conversationId: string;
  createdAt: string;
  role: "user" | "assistant";
  content: string;
  delivery: "committed" | "model-failed" | "complete";
}
```

Define memory confirmation as `"unconfirmed" | "confirmed" | "rejected"` and require non-empty source IDs for all derived records.

- [ ] **Step 4: Define the typed preload surface**

```ts
export interface YoursAPI {
  bootstrap(): Promise<BootstrapResult>;
  selectPortableRoot(): Promise<BootstrapResult>;
  saveModelConfig(input: ModelConfigInput): Promise<ModelConnectionResult>;
  sendMessage(input: SendMessageInput): Promise<SendMessageResult>;
  listMemories(input: ListMemoriesInput): Promise<MemoryRecord[]>;
  correctMemory(input: CorrectMemoryInput): Promise<void>;
  runMaintenance(input: MaintenanceInput): Promise<MaintenanceResult>;
}
```

Do not expose raw filesystem methods or a `getModelConfig()` method that returns the API key.

- [ ] **Step 5: Run verification and commit**

Run: `npm run verify`

Expected: schema tests and typecheck pass.

```bash
git add src/shared src/main/storage/schemas.ts tests/unit/domain-schemas.test.ts
git commit -m "feat: define versioned portable domain contracts"
```

---

### Task 3: Portable-root discovery and Vault initialization

**Files:**
- Create: `src/main/portable/portable-root.ts`
- Create: `src/main/portable/vault-layout.ts`
- Create: `src/main/portable/vault-initializer.ts`
- Test: `tests/integration/portable-root.test.ts`
- Test: `tests/integration/vault-initializer.test.ts`

**Interfaces:**
- Produces: `findPortableRoot(startPath: string): Promise<string | null>`
- Produces: `vaultLayout(root: string): VaultLayout`
- Produces: `initializePortableRoot(root: string, now: Date): Promise<VaultLayout>`
- Consumes: `VaultManifest` and persisted schemas from Task 2

- [ ] **Step 1: Write failing discovery and initialization tests**

Use `mkdtemp` and create a nested fake executable path. Assert upward discovery finds `yours.manifest.json`, initialization creates every authority directory, and no path stored in the manifest is absolute.

- [ ] **Step 2: Verify red**

Run: `npm test -- tests/integration/portable-root.test.ts tests/integration/vault-initializer.test.ts`

Expected: FAIL because portable services do not exist.

- [ ] **Step 3: Implement deterministic layout**

```ts
export interface VaultLayout {
  root: string;
  manifest: string;
  configDir: string;
  vaultDir: string;
  personaDir: string;
  conversationsDir: string;
  memoriesDir: string;
  narrativesDir: string;
  correctionsDir: string;
  thoughtsDir: string;
  stateDir: string;
  attachmentsDir: string;
  backupsDir: string;
  cacheDir: string;
}
```

Construct all fields from the discovered root. Reject a root without read/write access, and reject manifests whose relative data path escapes the root after `resolve()`.

- [ ] **Step 4: Implement manifest creation**

Write `yours.manifest.json` with:

```json
{
  "schemaVersion": 1,
  "kind": "yours-portable-root",
  "vaultPath": "Vault",
  "configPath": "config/model.json",
  "createdAt": "2026-07-30T00:00:00.000Z"
}
```

Create directories idempotently. A second initialization must preserve existing authority files.

- [ ] **Step 5: Verify and commit**

Run: `npm run verify`

```bash
git add src/main/portable tests/integration/portable-root.test.ts tests/integration/vault-initializer.test.ts
git commit -m "feat: discover and initialize portable Vault roots"
```

---

### Task 4: Crash-safe authority writes and single-writer lock

**Files:**
- Create: `src/main/storage/atomic-file.ts`
- Create: `src/main/storage/jsonl-store.ts`
- Create: `src/main/portable/vault-lock.ts`
- Test: `tests/integration/atomic-file.test.ts`
- Test: `tests/integration/jsonl-store.test.ts`
- Test: `tests/integration/vault-lock.test.ts`

**Interfaces:**
- Produces: `atomicWriteText(path: string, content: string): Promise<void>`
- Produces: `JsonlStore<T>.append(record: T): Promise<JsonlPosition>`
- Produces: `JsonlStore<T>.scan(): AsyncIterable<T>`
- Produces: `acquireVaultLock(vaultDir: string, identity: ProcessIdentity): Promise<VaultLease>`
- Consumes: `VaultLayout`

- [ ] **Step 1: Write failing durability tests**

Test LF normalization, atomic replacement without a remaining `.tmp`, monthly naming, 64 MB rotation via an injected size provider, recovery of only an incomplete final JSONL line, refusal to repair a corrupt middle line, and a second live writer receiving `VAULT_LOCKED`.

- [ ] **Step 2: Verify red**

Run: `npm test -- tests/integration/atomic-file.test.ts tests/integration/jsonl-store.test.ts tests/integration/vault-lock.test.ts`

- [ ] **Step 3: Implement atomic replacement**

Write to `.<basename>.<uuid>.tmp` in the same directory, open with mode `0o600`, write normalized UTF-8, call `sync()`, close, rename, then best-effort sync the parent directory. On failure, remove only the known temporary file.

- [ ] **Step 4: Implement append-only JSONL**

Serialize one compact JSON object plus `\n`, validate before append, open with `"a"`, write, sync, and close. Return:

```ts
export interface JsonlPosition {
  file: string;
  byteOffset: number;
  byteLength: number;
}
```

Before truncating an incomplete final line, copy the original file to `Vault/backups/recovery/<timestamp>/`.

- [ ] **Step 5: Implement the lease**

Store `.yours.lock` with `pid`, `hostname`, `instanceId`, and `acquiredAt`. Treat it as stale only when hostname equals the current host and the PID no longer exists. Never automatically break a lock created on another host.

- [ ] **Step 6: Verify and commit**

Run: `npm run verify`

```bash
git add src/main/storage src/main/portable/vault-lock.ts tests/integration
git commit -m "feat: add crash-safe portable authority storage"
```

---

### Task 5: Layered persona authority

**Files:**
- Create: `src/main/persona/defaults.ts`
- Create: `src/main/persona/persona-service.ts`
- Create: `src/main/state/state-repository.ts`
- Create: `src/main/persona/templates/constitution.md`
- Create: `src/main/persona/templates/boundaries.md`
- Create: `src/main/persona/templates/expression.md`
- Create: `src/main/persona/templates/relationship.md`
- Test: `tests/integration/persona-service.test.ts`
- Test: `tests/integration/state-repository.test.ts`

**Interfaces:**
- Produces: `PersonaService.readLayers(): Promise<PersonaLayers>`
- Produces: `PersonaService.updateFixedLayer(layer, content, actor): Promise<void>`
- Produces: `PersonaService.applyGrowth(candidate): Promise<PersonaEvolutionRecord>`
- Produces: `StateRepository.readCurrent(): Promise<CurrentState>`
- Produces: `StateRepository.replaceCurrent(candidate): Promise<void>`
- Produces: `StateRepository.upsertHangingEvent(event): Promise<void>`
- Consumes: atomic file and JSONL services from Task 4

- [ ] **Step 1: Write failing persona tests**

Assert initialization creates four readable Markdown files plus valid `state/current.json` and `state/hanging-events.json`; model actor cannot change `constitution` or `boundaries`; user actor can; every expression/relationship change appends old/new hashes and source message IDs to `evolution.jsonl`; a hanging event keeps its source IDs and can be resolved without deleting history.

- [ ] **Step 2: Verify red**

Run: `npm test -- tests/integration/persona-service.test.ts tests/integration/state-repository.test.ts`

- [ ] **Step 3: Add restrained initial Chinese persona**

The templates must state that she is newly acquainted, does not flatter, distinguishes memory from inference, may remain silent, and never claims a shared experience without a source. Do not prefill claims about the user.

- [ ] **Step 4: Implement layer permissions and growth limits**

Only `"user"` may update fixed layers. Model candidates may update `"expression"` or `"relationship"` only when they include at least one source message ID and remain below a 4,000-character replacement limit.

- [ ] **Step 5: Implement short-term state authority**

Store `current.json` through atomic replacement and hanging-event lifecycle in `hanging-events.json` plus append-only events. Current state may summarize recent hours/days but must not overwrite long-term persona or source conversations.

- [ ] **Step 6: Verify and commit**

Run: `npm run verify`

```bash
git add src/main/persona src/main/state tests/integration/persona-service.test.ts tests/integration/state-repository.test.ts
git commit -m "feat: establish layered and traceable persona"
```

---

### Task 6: Plaintext model configuration and secure connection policy

**Files:**
- Create: `src/main/model/connection-policy.ts`
- Create: `src/main/model/model-config-repository.ts`
- Create: `src/main/model/openai-compatible-gateway.ts`
- Test: `tests/unit/connection-policy.test.ts`
- Test: `tests/integration/model-config-repository.test.ts`
- Test: `tests/integration/openai-compatible-gateway.test.ts`

**Interfaces:**
- Produces: `assertAllowedModelURL(value: string): URL`
- Produces: `ModelConfigRepository.save(input): Promise<void>`
- Produces: `ModelConfigRepository.loadForMainProcess(): Promise<ModelConfig | null>`
- Produces: `ModelGateway.complete(request, signal): Promise<ModelReply>`
- Consumes: `config/model.json` path and atomic JSON writer

- [ ] **Step 1: Write failing policy tests**

Accept `https://model.example/v1`, `http://127.0.0.1:8000/v1`, `http://localhost:8000/v1`, and `http://[::1]:8000/v1`. Reject remote HTTP, credentials in URLs, query strings, fragments, and non-HTTP schemes.

- [ ] **Step 2: Write failing repository and gateway tests**

Assert the repository writes mode `0600`, never returns the key through renderer DTOs, the gateway sends `Authorization: Bearer …`, uses `/chat/completions`, times out, validates response shape, and redacts the key and prompt from errors.

- [ ] **Step 3: Verify red**

Run: `npm test -- tests/unit/connection-policy.test.ts tests/integration/model-config-repository.test.ts tests/integration/openai-compatible-gateway.test.ts`

- [ ] **Step 4: Implement configuration and gateway**

Use injected `fetch` and clock for tests. Set `Content-Type: application/json`, a 60-second abort timeout, and no request/response body logging. Return only:

```ts
export interface ModelReply {
  content: string;
  model: string;
  finishReason: string | null;
}
```

- [ ] **Step 5: Verify and commit**

Run: `npm run verify`

```bash
git add src/main/model tests/unit/connection-policy.test.ts tests/integration/model-*.test.ts
git commit -m "feat: connect portable config to HTTPS model gateway"
```

---

### Task 7: Raw-first conversation authority and chat coordinator

**Files:**
- Create: `src/main/chat/conversation-repository.ts`
- Create: `src/main/chat/chat-coordinator.ts`
- Test: `tests/integration/conversation-repository.test.ts`
- Test: `tests/integration/chat-coordinator.test.ts`

**Interfaces:**
- Produces: `ConversationRepository.appendUserMessage(input): Promise<MessageRecord>`
- Produces: `ConversationRepository.appendAssistantMessage(input): Promise<MessageRecord>`
- Produces: `ChatCoordinator.send(input): Promise<SendMessageResult>`
- Consumes: JSONL store and `ModelGateway`

- [ ] **Step 1: Write failing raw-first tests**

Use a gateway spy. Assert the user record exists and is fsynced before the gateway is called; model failure updates delivery through a new append-only status event rather than rewriting the message; retry reuses the same user message ID; assistant content is committed after success.

- [ ] **Step 2: Verify red**

Run: `npm test -- tests/integration/conversation-repository.test.ts tests/integration/chat-coordinator.test.ts`

- [ ] **Step 3: Implement conversation events**

Store message records and delivery-status events in the monthly conversation JSONL. Reconstruct current delivery state by folding events. Reject blank input and input above 32,000 Unicode code points.

- [ ] **Step 4: Implement coordinator idempotency**

`send()` accepts a renderer-generated `requestId`. Persist `requestId → userMessageId`; repeated requests return or retry the existing message rather than appending another user message.

- [ ] **Step 5: Verify and commit**

Run: `npm run verify`

```bash
git add src/main/chat tests/integration/conversation-repository.test.ts tests/integration/chat-coordinator.test.ts
git commit -m "feat: persist conversations before model requests"
```

---

### Task 8: Typed preload, IPC registration, and onboarding screen

**Files:**
- Create: `src/main/ipc/register-ipc.ts`
- Modify: `src/main/main.ts`
- Modify: `src/preload/preload.ts`
- Create: `src/renderer/api.ts`
- Modify: `src/renderer/App.tsx`
- Create: `src/renderer/onboarding/OnboardingScreen.tsx`
- Create: `src/renderer/styles.css`
- Test: `tests/unit/ipc-contract.test.ts`
- Test: `tests/unit/onboarding-screen.test.tsx`

**Interfaces:**
- Produces: `window.yours: YoursAPI`
- Produces: IPC channels `bootstrap`, `selectPortableRoot`, `saveModelConfig`
- Consumes: portable-root, Vault initializer, lease, persona, and model services

- [ ] **Step 1: Write failing IPC and onboarding tests**

Assert unknown keys are stripped by Zod before main-process handlers run, thrown errors become serializable error codes, API key inputs are never echoed in results, onboarding limits seeds to 10, and a successful connection advances to chat.

- [ ] **Step 2: Verify red**

Run: `npm test -- tests/unit/ipc-contract.test.ts tests/unit/onboarding-screen.test.tsx`

- [ ] **Step 3: Implement the narrow bridge**

Expose explicit functions with `contextBridge.exposeInMainWorld`. Do not expose `ipcRenderer`, event names, file paths outside the chosen root, or model secrets.

- [ ] **Step 4: Implement onboarding**

Screens:

1. discover/select/create portable root;
2. explain plaintext local storage;
3. enter base URL, model ID, and API key, then test connection;
4. set mutual names;
5. optionally add 0–10 memory seeds.

Persist seeds as user-provided explicit facts with source kind `onboardingSeed`, not as personality conclusions.

- [ ] **Step 5: Verify and commit**

Run: `npm run verify`

```bash
git add src/main/ipc src/main/main.ts src/preload src/renderer tests/unit
git commit -m "feat: add secure onboarding and typed desktop bridge"
```

---

### Task 9: Conversation UI with safe retry

**Files:**
- Create: `src/renderer/chat/ChatScreen.tsx`
- Create: `src/renderer/chat/MessageList.tsx`
- Modify: `src/renderer/App.tsx`
- Modify: `src/renderer/styles.css`
- Modify: `src/main/ipc/register-ipc.ts`
- Test: `tests/unit/chat-screen.test.tsx`

**Interfaces:**
- Produces: accessible chat composer, message stream, failure state, retry action
- Produces: explicit `endConversation(conversationId)` event for reflection eligibility
- Consumes: `YoursAPI.sendMessage`

- [ ] **Step 1: Write failing renderer tests**

Assert Enter sends, Shift+Enter inserts a newline, the composer clears only after raw commit succeeds, double-click does not duplicate a request ID, failed model calls retain the user bubble, retry sends the same request ID, and “先到这里” appends one conversation-ended event.

- [ ] **Step 2: Verify red**

Run: `npm test -- tests/unit/chat-screen.test.tsx`

- [ ] **Step 3: Implement the minimum chat screen**

Keep a neutral system light/dark background. Show user and assistant messages, a pending indicator, and inline failure text with “重新回应”. Do not display memory scores, tokens, prompts, or model internals.

Add a quiet “先到这里” action in the conversation menu. It marks the current conversation ended, creates a fresh conversation ID for the next user message, and makes the ended conversation eligible for reflection without showing a summary.

- [ ] **Step 4: Wire conversation hydration**

Add `listConversation(conversationId)` to the IPC contract and repository. On relaunch, fold message and status events and render the same ordered conversation.

- [ ] **Step 5: Verify and commit**

Run: `npm run verify`

```bash
git add src/renderer/chat src/renderer/App.tsx src/renderer/styles.css src/main/ipc src/shared tests/unit/chat-screen.test.tsx
git commit -m "feat: deliver durable text conversation experience"
```

---

### Task 10: Traceable three-stream memory authority

**Files:**
- Create: `src/main/memory/memory-repository.ts`
- Test: `tests/integration/memory-repository.test.ts`

**Interfaces:**
- Produces: `MemoryRepository.appendCandidate(memory): Promise<MemoryRecord>`
- Produces: `MemoryRepository.list(query): Promise<MemoryRecord[]>`
- Produces: `MemoryRepository.changeStatus(input): Promise<void>`
- Consumes: event, relationship, and understanding JSONL streams

- [ ] **Step 1: Write failing memory tests**

Assert records route to the correct file; derived memories require existing source messages; rejected understanding remains in history but disappears from active results; sealing excludes proactive and retrieval results; missing source messages fail closed.

- [ ] **Step 2: Verify red**

Run: `npm test -- tests/integration/memory-repository.test.ts`

- [ ] **Step 3: Implement append-only lifecycle events**

Never rewrite a memory record. Append `memoryStatusChanged`, `memoryConfirmed`, and `memoryEvidenceAdded` events, then fold them when reading.

- [ ] **Step 4: Implement source validation**

Before append, load each cited message and verify it exists. For `quote`, require the quoted span to occur in the source content. For `inference`, preserve confidence from `0` to `1` and confirmation `"unconfirmed"`.

- [ ] **Step 5: Verify and commit**

Run: `npm run verify`

```bash
git add src/main/memory tests/integration/memory-repository.test.ts
git commit -m "feat: add source-traceable active memories"
```

---

### Task 11: Disposable SQLite FTS index and bounded retrieval

**Files:**
- Create: `src/main/index/search-index.ts`
- Create: `src/main/index/index-rebuilder.ts`
- Create: `src/main/memory/memory-retriever.ts`
- Create: `scripts/verify-package.mjs`
- Test: `tests/integration/search-index.test.ts`
- Test: `tests/integration/index-rebuilder.test.ts`
- Test: `tests/unit/memory-retriever.test.ts`

**Interfaces:**
- Produces: `SearchIndex.upsert(document): void`
- Produces: `SearchIndex.search(query, limit): SearchHit[]`
- Produces: `IndexRebuilder.rebuild(): Promise<IndexBuildReport>`
- Produces: `MemoryRetriever.retrieve(input): Promise<RetrievedContext>`
- Consumes: all authority readers, never writes authority

- [ ] **Step 1: Write failing index tests**

Assert FTS finds Chinese text, sealed/rejected memory is filtered, deleting `index.sqlite` followed by rebuild yields equivalent results, the stored cursor includes source path/byte offset/hash, and a corrupt database is replaced without authority loss.

- [ ] **Step 2: Verify red**

Run: `npm test -- tests/integration/search-index.test.ts tests/integration/index-rebuilder.test.ts tests/unit/memory-retriever.test.ts`

- [ ] **Step 3: Implement derived schema**

Create FTS5 content plus metadata tables:

```sql
CREATE VIRTUAL TABLE documents_fts USING fts5(id UNINDEXED, kind UNINDEXED, content);
CREATE TABLE source_cursor(path TEXT PRIMARY KEY, byte_offset INTEGER NOT NULL, sha256 TEXT NOT NULL);
CREATE TABLE index_meta(key TEXT PRIMARY KEY, value TEXT NOT NULL);
```

Store no API configuration. Use transactions and close the database before portable copying.

- [ ] **Step 4: Implement retrieval**

Rank lexical relevance, recency, importance, confirmation, and source quality. Return at most 12 memories and at most 5 relevant active narratives; never return sealed or rejected content.

- [ ] **Step 5: Verify native packaging and commit**

Run:

```bash
npm run verify
npm run package:mac
node scripts/verify-package.mjs out
```

Expected: packaged app contains unpacked `better-sqlite3` native binary and opens its index.

```bash
git add src/main/index src/main/memory/memory-retriever.ts tests scripts/verify-package.mjs
git commit -m "feat: add rebuildable local memory search"
```

---

### Task 12: Bounded personality and memory context assembly

**Files:**
- Create: `src/main/context/context-builder.ts`
- Modify: `src/main/chat/chat-coordinator.ts`
- Test: `tests/unit/context-builder.test.ts`
- Test: `tests/integration/chat-context.test.ts`

**Interfaces:**
- Produces: `ContextBuilder.build(input): Promise<ModelContext>`
- Consumes: persona layers, current state, hanging events, retrieved memories, narratives, and current conversation

- [ ] **Step 1: Write failing context tests**

Assert ordering is fixed persona → boundaries → growing relationship → current state → relevant authority → current conversation. Assert API key and base URL never appear, sealed memories are absent, “你说过” entries contain source IDs, and total context obeys an injected character budget.

- [ ] **Step 2: Verify red**

Run: `npm test -- tests/unit/context-builder.test.ts tests/integration/chat-context.test.ts`

- [ ] **Step 3: Implement deterministic budgeting**

Reserve current conversation first, then fixed persona, then boundaries. Fit relevant memory/narrative items by rank. Truncate only at record boundaries; do not cut a quote into a misleading fragment.

- [ ] **Step 4: Add response instructions**

The system context requires natural Chinese, uncertainty for inference, no database language, no fabricated shared history, and no claim of updated memory in ordinary replies.

- [ ] **Step 5: Verify and commit**

Run: `npm run verify`

```bash
git add src/main/context src/main/chat/chat-coordinator.ts tests/unit/context-builder.test.ts tests/integration/chat-context.test.ts
git commit -m "feat: assemble bounded continuous companion context"
```

---

### Task 13: Reflection parsing, idempotency, and scheduling

**Files:**
- Create: `src/main/reflection/reflection-parser.ts`
- Create: `src/main/reflection/reflection-service.ts`
- Create: `src/main/reflection/reflection-scheduler.ts`
- Test: `tests/unit/reflection-parser.test.ts`
- Test: `tests/integration/reflection-service.test.ts`
- Test: `tests/unit/reflection-scheduler.test.ts`

**Interfaces:**
- Produces: `parseReflection(value: unknown): ReflectionResult`
- Produces: `ReflectionService.reflect(conversationId): Promise<ReflectionReport>`
- Produces: `ReflectionScheduler.eligible(now): Promise<string[]>`
- Consumes: model gateway, conversation authority, memory authority, thoughts JSONL, reflection cursor

- [ ] **Step 1: Write failing parser tests**

Accept 0–3 thoughts and valid memory candidates. Reject unknown source IDs, more than 3 thoughts, confidence outside `[0,1]`, unsupported source nature, and narrative claims without evidence.

- [ ] **Step 2: Write failing scheduling tests**

Assert explicit conversation end is eligible, 29:59 idle is not, 30:00 idle is, ordinary greeting-only conversation is not, and the same committed message range is reflected exactly once.

- [ ] **Step 3: Verify red**

Run: `npm test -- tests/unit/reflection-*.test.ts tests/integration/reflection-service.test.ts`

- [ ] **Step 4: Implement reflection**

Save the reflection cursor only after all accepted append-only outputs commit. On partial failure, record a transaction ID on each output so retry detects and skips already committed records.

- [ ] **Step 5: Implement scheduler lifecycle**

Use an in-process timer while the app is open and run an eligibility scan on startup. Do not create OS background tasks or notifications.

- [ ] **Step 6: Verify and commit**

Run: `npm run verify`

```bash
git add src/main/reflection tests/unit/reflection-*.test.ts tests/integration/reflection-service.test.ts
git commit -m "feat: reflect once on meaningful finished conversations"
```

---

### Task 14: Narrative limits and append-only correction

**Files:**
- Create: `src/main/narrative/narrative-repository.ts`
- Create: `src/main/narrative/correction-service.ts`
- Test: `tests/integration/narrative-repository.test.ts`
- Test: `tests/integration/correction-service.test.ts`

**Interfaces:**
- Produces: `NarrativeRepository.propose(candidate): Promise<NarrativeDecision>`
- Produces: `NarrativeRepository.listActive(): Promise<NarrativeDocument[]>`
- Produces: `CorrectionService.correct(input): Promise<CorrectionRecord>`
- Consumes: atomic Markdown writer, memory repository, corrections JSONL

- [ ] **Step 1: Write failing narrative tests**

Assert creation requires two distinct conversations or explicit importance/high impact, no more than 5 narratives can be active, Markdown front matter cites memories/messages, and a one-off conversation cannot create a grand interpretation.

- [ ] **Step 2: Write failing correction tests**

Assert correction preserves the old interpretation, appends user wording and source message, marks the old memory rejected, creates a cautious replacement only when supplied, and retrieval stops using the old claim.

- [ ] **Step 3: Verify red**

Run: `npm test -- tests/integration/narrative-repository.test.ts tests/integration/correction-service.test.ts`

- [ ] **Step 4: Implement narrative documents**

Each Markdown document contains machine-readable front matter and human-readable sections: 过去、现在、变化、她的理解、反证、开放问题、来源. Parse and validate front matter before atomic replacement.

- [ ] **Step 5: Implement correction transaction**

Order writes as correction record → memory status event → optional new candidate → index update. If index update fails, authority remains valid and startup rebuild repairs it.

- [ ] **Step 6: Verify and commit**

Run: `npm run verify`

```bash
git add src/main/narrative tests/integration/narrative-repository.test.ts tests/integration/correction-service.test.ts
git commit -m "feat: maintain restrained narratives and real corrections"
```

---

### Task 15: Restrained proactive recall

**Files:**
- Create: `src/main/proactive/timing-gate.ts`
- Create: `src/main/proactive/proactive-recall-service.ts`
- Modify: `src/main/ipc/register-ipc.ts`
- Test: `tests/unit/timing-gate.test.ts`
- Test: `tests/integration/proactive-recall-service.test.ts`

**Interfaces:**
- Produces: `TimingGate.evaluate(thought, context): TimingDecision`
- Produces: `ProactiveRecallService.onAppOpen(): Promise<ThoughtRecord | null>`
- Consumes: unspoken thoughts, sealed topics, current vulnerability flags, and expiry

- [ ] **Step 1: Write failing restraint tests**

Reject stale, sealed, already expressed, evidence-poor, non-novel, and vulnerability-burdening thoughts. Accept a relevant unresolved event with explicit evidence. When several pass, return only the highest importance/recency thought.

- [ ] **Step 2: Verify red**

Run: `npm test -- tests/unit/timing-gate.test.ts tests/integration/proactive-recall-service.test.ts`

- [ ] **Step 3: Implement pure timing policy**

Return structured reasons:

```ts
type TimingDecision =
  | { allowed: true; score: number }
  | { allowed: false; reason: "stale" | "sealed" | "weak-evidence" | "burdensome" | "already-expressed" | "not-novel" };
```

- [ ] **Step 4: Implement app-open consumption**

Append `thoughtExpressed` only after the renderer acknowledges that the proactive message was displayed. A crash before acknowledgement may display it again; it must never silently disappear.

- [ ] **Step 5: Verify and commit**

Run: `npm run verify`

```bash
git add src/main/proactive src/main/ipc tests/unit/timing-gate.test.ts tests/integration/proactive-recall-service.test.ts
git commit -m "feat: add one restrained app-open recollection"
```

---

### Task 16: Memory, relationship, persona, and model settings screens

**Files:**
- Create: `src/renderer/memory/MemoryScreen.tsx`
- Create: `src/renderer/settings/SettingsScreen.tsx`
- Modify: `src/renderer/App.tsx`
- Modify: `src/renderer/styles.css`
- Modify: `src/main/ipc/register-ipc.ts`
- Test: `tests/unit/memory-screen.test.tsx`
- Test: `tests/unit/settings-screen.test.tsx`

**Interfaces:**
- Produces: four-screen V1 navigation
- Consumes: list/correct/seal/unseal memory APIs, persona fixed-layer update, model connection test, maintenance APIs

- [ ] **Step 1: Write failing UI tests**

Assert memory filters cover event/relationship/understanding; source links open the original conversation excerpt; correction is conversational text, not a yes/no score; sealing removes an item from active view; fixed persona editing warns and saves only on explicit confirmation; API key is never populated back into the form.

- [ ] **Step 2: Verify red**

Run: `npm test -- tests/unit/memory-screen.test.tsx tests/unit/settings-screen.test.tsx`

- [ ] **Step 3: Implement memory and relationship screen**

Show content, source nature, confirmation state, confidence language, sources, evidence, counterevidence, status, and correction history. Never render internal numeric relevance scores as a portrait of the user.

- [ ] **Step 4: Implement settings**

Include model replacement/test, current portable-root path, fixed persona editor, integrity check, index rebuild, backup, portable export, and restore entry points. Leave encryption, sync, update, and notification controls absent.

- [ ] **Step 5: Verify and commit**

Run: `npm run verify`

```bash
git add src/renderer src/main/ipc tests/unit/memory-screen.test.tsx tests/unit/settings-screen.test.tsx
git commit -m "feat: expose memory sources and portable maintenance"
```

---

### Task 17: Integrity, migrations, backups, portable export, and restore

**Files:**
- Create: `src/main/maintenance/integrity-service.ts`
- Create: `src/main/maintenance/migration-service.ts`
- Create: `src/main/maintenance/backup-service.ts`
- Create: `src/main/maintenance/portable-export-service.ts`
- Create: `src/main/maintenance/restore-service.ts`
- Modify: `src/main/main.ts`
- Modify: `src/main/storage/atomic-file.ts`
- Modify: `src/main/storage/jsonl-store.ts`
- Test: `tests/integration/integrity-service.test.ts`
- Test: `tests/integration/migration-service.test.ts`
- Test: `tests/integration/backup-service.test.ts`
- Test: `tests/integration/portable-export-service.test.ts`
- Test: `tests/integration/restore-service.test.ts`

**Interfaces:**
- Produces: `IntegrityService.check(mode): Promise<IntegrityReport>`
- Produces: `MigrationService.migrate(targetVersion): Promise<MigrationReport>`
- Produces: `BackupService.run(now): Promise<BackupReport>`
- Produces: `PortableExportService.create(destination): Promise<ExportReport>`
- Produces: `RestoreService.restore(source): Promise<RestoreReport>`
- Produces: `AuthorityChangeNotifier` that coalesces successful authority writes and schedules at most one backup check
- Consumes: writer lease, all authority readers, SHA-256 file manifest

- [ ] **Step 1: Write failing integrity and migration tests**

Assert startup detects malformed middle JSONL, missing sources, mismatched manifest version, stale index cursor, and newer unsupported data. Final-line damage is recoverable with a copy; middle damage produces read-only mode. A migration creates a pre-migration backup and is idempotent.

- [ ] **Step 2: Write failing backup/export tests**

Assert the first authority change of a day creates one backup, unchanged repeated runs do not, retention keeps 7 daily and 4 weekly snapshots, automatic backups omit `config/model.json`, and complete export contains manifest, config, Vault, checksums, and present platform packages.

- [ ] **Step 3: Write failing restore tests**

Reject checksum mismatch and path traversal. Before restore, create a recovery backup. Inject failure halfway and assert the current Vault remains unchanged. On success, rebuild the index and preserve relative paths.

- [ ] **Step 4: Verify red**

Run: `npm test -- tests/integration/*service.test.ts`

- [ ] **Step 5: Implement integrity and migration transactions**

Represent problems as:

```ts
export interface IntegrityIssue {
  severity: "info" | "repairable" | "read-only";
  code: string;
  relativePath: string;
  detail: string;
}
```

Never place private record contents in `detail`. Use staging directories inside the portable root so rename remains on the same volume.

- [ ] **Step 6: Implement backup retention**

Create `Vault/backups/.staging/<uuid>`, copy authority files excluding `.cache` and nested backups, generate SHA-256 `files.json`, verify, then rename to `daily/<date>` or `weekly/<iso-week>`.

- [ ] **Step 7: Wire automatic backup triggering**

Notify only after an authoritative JSONL append or atomic authority replacement succeeds. Debounce changes for 5 seconds, then call `BackupService.run(now)`. A backup error is reported in maintenance status but never rolls back the already committed conversation.

- [ ] **Step 8: Implement complete portable archive**

Close SQLite, copy the portable manifest, model config, Vault authority, and any present Mac/Windows program packages to staging, generate checksums, and archive without following symlinks. Display a plaintext-data warning before IPC invokes this method.

- [ ] **Step 9: Implement transactional restore**

Validate all relative paths stay under extraction root, verify checksums, acquire exclusive lease, create recovery backup, stage restored authority, atomically swap Vault directories, then rebuild the index.

- [ ] **Step 10: Verify and commit**

Run: `npm run verify`

```bash
git add src/main/maintenance tests/integration
git commit -m "feat: make portable continuity verifiable and recoverable"
```

---

### Task 18: Cross-platform packaging and end-to-end acceptance

**Files:**
- Create: `scripts/prepare-portable-root.mjs`
- Modify: `scripts/verify-package.mjs`
- Create: `playwright.config.ts`
- Create: `tests/e2e/portable-flow.spec.ts`
- Create: `.github/workflows/verify.yml`
- Modify: `README.md`
- Modify: `docs/版本规划.md`

**Interfaces:**
- Produces: `out/portable/Yours-mac-arm64.zip`
- Produces: `out/portable/Yours-win-x64.zip`
- Produces: CI verification on macOS and Windows
- Consumes: complete V1 application

- [ ] **Step 1: Write the failing Electron acceptance test**

Launch with an isolated `YOURS_TEST_PORTABLE_ROOT` and a local fake HTTPS-compatible gateway. Cover:

1. create portable root;
2. configure the fake model;
3. send a message and receive a reply;
4. close and relaunch;
5. verify the conversation persists;
6. create an evidence-linked memory;
7. correct it and verify old understanding is inactive;
8. delete the SQLite index and verify rebuild;
9. generate and validate a portable export.

- [ ] **Step 2: Run and verify red**

Run: `npx playwright test tests/e2e/portable-flow.spec.ts`

Expected: FAIL until packaging/test hooks and the complete IPC flow are wired.

- [ ] **Step 3: Implement deterministic portable assembly**

`prepare-portable-root.mjs` takes `--platform darwin-arm64|win32-x64`, copies the packaged app into the canonical directory, creates only a placeholder `config/model.example.json`, creates no real API key, and produces a ZIP that preserves macOS executable permissions.

- [ ] **Step 4: Add package verification**

`verify-package.mjs` must fail if:

- a real `config/model.json` exists;
- files contain `Authorization: Bearer`;
- Vault conversation/memory JSONL is present;
- the expected executable is absent;
- `better-sqlite3` native binding is absent;
- renderer source maps or development server URLs are shipped.

- [ ] **Step 5: Add the build matrix**

Use `macos-15` for `darwin/arm64` and `windows-2025` for `win32/x64`. Each job runs:

```text
npm ci
npm run typecheck
npm test
npx playwright test
electron-forge package for the host target
node scripts/verify-package.mjs out
node scripts/prepare-portable-root.mjs --platform <target>
```

Upload artifacts only; do not publish a release or auto-update feed.

- [ ] **Step 6: Document personal use**

README must cover:

- development prerequisites and exact commands;
- how to download/unzip the matching platform artifact;
- the plaintext API key warning;
- closing the App before copying;
- copying the whole `Yours` directory between Mac and Windows;
- replacing only program files during upgrades;
- integrity check, index rebuild, backup, export, and restore;
- first-run macOS/Windows security prompts for unsigned self-use builds.

- [ ] **Step 7: Run the full acceptance suite**

Run on macOS:

```bash
npm ci
npm run verify
npx playwright test
npm run package:mac
node scripts/verify-package.mjs out
node scripts/prepare-portable-root.mjs --platform darwin-arm64
```

Run the corresponding commands in the Windows CI job with `package:win` and `win32-x64`.

Expected: all checks pass and both portable archives are produced without private data.

- [ ] **Step 8: Update project status and commit**

Set the roadmap state to:

```text
V1 跨平台便携桌面版：已实现，进入本人连续使用两到四周的验收期
```

```bash
git add scripts playwright.config.ts tests/e2e .github/workflows/verify.yml README.md docs/版本规划.md
git commit -m "build: verify portable macOS and Windows releases"
```

---

## Final Verification Gate

Before calling V1 implemented:

- [ ] `npm ci` succeeds from a clean checkout.
- [ ] `npm run typecheck` passes with no suppressed TypeScript errors.
- [ ] `npm test` passes.
- [ ] Playwright Electron acceptance passes on macOS and Windows.
- [ ] Packaged apps run without a separately installed Node.js or database.
- [ ] A user message remains after forced model failure and relaunch.
- [ ] Every generated memory used in context resolves to existing source messages.
- [ ] A rejected understanding is visible in history but absent from current retrieval.
- [ ] Deleting `.cache/index.sqlite` and rebuilding changes no authority file.
- [ ] Mac → Windows → Mac Vault round-trip preserves authority-file SHA-256 hashes, excluding expected lock/cache/backup changes.
- [ ] Automatic backup excludes the API key.
- [ ] Complete portable export includes the plaintext model configuration only after the explicit warning.
- [ ] Restore failure leaves the current Vault untouched.
- [ ] Repository and packaged example artifacts contain no real API key, model URL, prompt, conversation, memory, or backup.
- [ ] No cloud backend, account, sync, mobile, auto-update, notification, voice, avatar, external ingestion, or agent action slipped into V1.

## Primary References

- Product behavior: `docs/superpowers/specs/2026-07-29-her-v1-design.md`
- Portable architecture: `docs/superpowers/specs/2026-07-29-her-v1-portable-desktop-design.md`
- Electron 43.2.0 release: <https://releases.electronjs.org/release/v43.2.0>
- Electron Forge Vite TypeScript template: <https://www.electronforge.io/templates/vite>
- Electron Forge TypeScript configuration: <https://www.electronforge.io/config/typescript-configuration>
- Electron security checklist: <https://www.electronjs.org/docs/latest/tutorial/security>
- Electron packaging: <https://www.electronjs.org/docs/latest/tutorial/application-distribution>
- Playwright Electron automation: <https://playwright.dev/docs/api/class-electron>
- better-sqlite3: <https://www.npmjs.com/package/better-sqlite3>
