# Clipboard Mirror Policy Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development on capable harnesses. In pi runtimes, or when you need a separate session fallback, use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add persistent `piVim.clipboardMirror` settings so users can choose whether pi-vim mirrors all register writes, yanks only, or no writes to the OS clipboard.

**Architecture:** Keep register writes synchronous and centralize mirroring in `ModalEditor.writeToRegister(text, source)`. Add a focused `clipboard-policy.ts` module for settings reads, policy resolution, diagnostics, and test seams; `index.ts` only applies the resolved policy to editor behavior.

**Tech Stack:** TypeScript, Pi `ExtensionAPI`, Pi `SettingsManager`, Node test runner, Biome, ESLint, `lefthook`.

---

## File structure

- `clipboard-policy.ts`
  - New focused module for `ClipboardMirrorPolicy`, `RegisterWriteSource`, policy resolution, safe diagnostics, merged settings reads, and the settings-reader test seam.
- `index.ts`
  - Imports the policy module.
  - Stores the resolved policy on `ModalEditor`.
  - Keeps platform clipboard backend ownership unchanged.
  - Keeps `writeToRegister(text, source)` as the only mirror decision point.
- `test/clipboard-policy.test.ts`
  - New focused tests for pure resolver behavior and settings merge behavior.
- `test/clipboard-policy-editor.test.ts`
  - New focused editor integration tests for mirror policy behavior using the existing editor harness.
- `test/modal-editor.test.ts`
  - Adds only extension-startup settings tests near existing extension lifecycle tests.
- `README.md`
  - Adds high-up settings documentation near install/setup.
  - Updates the register/clipboard policy section.

---

## Chunk 1: Policy resolver and settings reader

### Task 1: Add failing pure policy tests

**Files:**
- Create: `test/clipboard-policy.test.ts`

- [ ] **Step 1: Create focused policy test file**

Create `test/clipboard-policy.test.ts`:

```ts
import assert from "node:assert/strict";
import { describe, it } from "node:test";

import {
  DEFAULT_CLIPBOARD_MIRROR_POLICY,
  readPiVimClipboardMirrorSetting,
  resolveClipboardMirrorPolicy,
} from "../clipboard-policy.js";

describe("clipboard mirror policy resolver", () => {
  it("defaults missing clipboard mirror policy to all", () => {
    assert.deepEqual(resolveClipboardMirrorPolicy(undefined), {
      policy: DEFAULT_CLIPBOARD_MIRROR_POLICY,
    });
  });

  it("accepts all supported clipboard mirror policy values", () => {
    assert.deepEqual(resolveClipboardMirrorPolicy("all"), { policy: "all" });
    assert.deepEqual(resolveClipboardMirrorPolicy("yank"), { policy: "yank" });
    assert.deepEqual(resolveClipboardMirrorPolicy("never"), { policy: "never" });
  });

  it("normalizes clipboard mirror policy casing and whitespace", () => {
    assert.deepEqual(resolveClipboardMirrorPolicy("YANK"), { policy: "yank" });
    assert.deepEqual(resolveClipboardMirrorPolicy(" never "), { policy: "never" });
  });

  it("falls back to all and reports invalid clipboard mirror strings", () => {
    const result = resolveClipboardMirrorPolicy("delete");

    assert.equal(result.policy, "all");
    assert.match(result.warning ?? "", /delete/);
    assert.match(result.warning ?? "", /all, yank, never/);
  });

  it("falls back to all and reports non-string clipboard mirror values safely", () => {
    const result = resolveClipboardMirrorPolicy({ mode: "yank" });

    assert.equal(result.policy, "all");
    assert.match(result.warning ?? "", /object/);
    assert.match(result.warning ?? "", /all, yank, never/);
  });
});

describe("piVim clipboard mirror settings reader", () => {
  it("returns undefined when global and project settings are missing", () => {
    assert.equal(readPiVimClipboardMirrorSetting(undefined, undefined), undefined);
    assert.equal(readPiVimClipboardMirrorSetting(null, null), undefined);
    assert.equal(readPiVimClipboardMirrorSetting("bad", 42), undefined);
  });

  it("reads global piVim clipboardMirror when project setting is missing", () => {
    assert.equal(
      readPiVimClipboardMirrorSetting(
        { piVim: { clipboardMirror: "yank" } },
        {},
      ),
      "yank",
    );
  });

  it("lets project piVim clipboardMirror override global", () => {
    assert.equal(
      readPiVimClipboardMirrorSetting(
        { piVim: { clipboardMirror: "never" } },
        { piVim: { clipboardMirror: "all" } },
      ),
      "all",
    );
  });

  it("treats invalid project clipboardMirror as an override instead of falling back to global", () => {
    assert.equal(
      readPiVimClipboardMirrorSetting(
        { piVim: { clipboardMirror: "yank" } },
        { piVim: { clipboardMirror: null } },
      ),
      null,
    );
  });
});
```

- [ ] **Step 2: Run tests and verify they fail for missing module**

Run:

```bash
node --import tsx/esm --test test/clipboard-policy.test.ts
```

Expected: FAIL with a module-not-found error for `../clipboard-policy.js`.

### Task 2: Implement the focused policy module

**Files:**
- Create: `clipboard-policy.ts`

- [ ] **Step 1: Create the policy module with constants and types**

Create `clipboard-policy.ts`:

```ts
import { SettingsManager } from "@mariozechner/pi-coding-agent";

const CLIPBOARD_MIRROR_POLICY_VALUES = ["all", "yank", "never"] as const;

export type ClipboardMirrorPolicy = typeof CLIPBOARD_MIRROR_POLICY_VALUES[number];
export type RegisterWriteSource = "mutation" | "yank";

export const DEFAULT_CLIPBOARD_MIRROR_POLICY: ClipboardMirrorPolicy = "all";

type ClipboardMirrorPolicyResult = {
  policy: ClipboardMirrorPolicy;
  warning?: string;
};

export type PiVimSettings = {
  clipboardMirror?: unknown;
};

type PiVimSettingsContainer = {
  piVim?: unknown;
};

type PiVimSettingsReader = (cwd: string) => PiVimSettings;
```

- [ ] **Step 2: Add policy validation and safe diagnostics**

Append:

```ts
function isClipboardMirrorPolicy(value: string): value is ClipboardMirrorPolicy {
  return (CLIPBOARD_MIRROR_POLICY_VALUES as readonly string[]).includes(value);
}

function formatInvalidSettingValue(value: unknown): string {
  if (typeof value === "string") return JSON.stringify(value);
  try {
    const json = JSON.stringify(value);
    if (json !== undefined) return `${typeof value} ${json}`;
  } catch {
    // Fall through to type-only formatting.
  }
  return typeof value;
}

export function resolveClipboardMirrorPolicy(value: unknown): ClipboardMirrorPolicyResult {
  if (value === undefined) return { policy: DEFAULT_CLIPBOARD_MIRROR_POLICY };

  if (typeof value === "string") {
    const normalized = value.trim().toLowerCase();
    if (isClipboardMirrorPolicy(normalized)) return { policy: normalized };
  }

  return {
    policy: DEFAULT_CLIPBOARD_MIRROR_POLICY,
    warning: `Unsupported piVim.clipboardMirror ${formatInvalidSettingValue(value)}; expected one of all, yank, never. Using all.`,
  };
}
```

- [ ] **Step 3: Add robust settings object extraction**

Append:

```ts
function getPiVimSettings(settings: unknown): PiVimSettings {
  if (typeof settings !== "object" || settings === null) return {};
  const piVim = (settings as PiVimSettingsContainer).piVim;
  return typeof piVim === "object" && piVim !== null ? piVim as PiVimSettings : {};
}

function hasClipboardMirrorSetting(settings: PiVimSettings): boolean {
  return Object.prototype.hasOwnProperty.call(settings, "clipboardMirror");
}

export function readPiVimClipboardMirrorSetting(
  globalSettings: unknown,
  projectSettings: unknown,
): unknown {
  const globalPiVim = getPiVimSettings(globalSettings);
  const projectPiVim = getPiVimSettings(projectSettings);
  return hasClipboardMirrorSetting(projectPiVim)
    ? projectPiVim.clipboardMirror
    : globalPiVim.clipboardMirror;
}
```

- [ ] **Step 4: Add disk-backed settings reader and test seam**

Append:

```ts
function readPiVimSettingsFromDisk(cwd: string): PiVimSettings {
  const settingsManager = SettingsManager.create(cwd);
  const clipboardMirror = readPiVimClipboardMirrorSetting(
    settingsManager.getGlobalSettings(),
    settingsManager.getProjectSettings(),
  );
  return { clipboardMirror };
}

let piVimSettingsReader: PiVimSettingsReader = readPiVimSettingsFromDisk;

export function readPiVimSettings(cwd: string): PiVimSettings {
  return piVimSettingsReader(cwd);
}

export function setPiVimSettingsReaderForTests(reader: PiVimSettingsReader): () => void {
  const previous = piVimSettingsReader;
  piVimSettingsReader = reader;
  return () => {
    piVimSettingsReader = previous;
  };
}
```

- [ ] **Step 5: Run policy tests and verify they pass**

Run:

```bash
node --import tsx/esm --test test/clipboard-policy.test.ts
```

Expected: PASS for all resolver and settings reader tests.

- [ ] **Step 6: Commit resolver and settings reader**

Run:

```bash
git add clipboard-policy.ts test/clipboard-policy.test.ts
git commit -m "feat(clipboard): resolve mirror policy settings"
```

---

## Chunk 2: Editor mirror policy behavior

### Task 3: Add failing editor policy tests

**Files:**
- Create: `test/clipboard-policy-editor.test.ts`

- [ ] **Step 1: Create focused editor policy behavior tests**

Create `test/clipboard-policy-editor.test.ts`:

```ts
import assert from "node:assert/strict";
import { describe, it } from "node:test";

import { createEditorWithSpy, sendKeys } from "./harness.js";

function nextImmediate(): Promise<void> {
  return new Promise<void>((resolve) => setImmediate(resolve));
}

describe("clipboard mirror policy", () => {
  it("all clipboard mirror policy mirrors mutation and yank writes", async () => {
  const { editor, clipboardWrites } = createEditorWithSpy("foo bar");
  editor.setClipboardMirrorPolicy("all");

  sendKeys(editor, ["d", "w", "y", "w"]);
  await nextImmediate();

  assert.equal(editor.getRegister(), "bar");
  assert.deepEqual(clipboardWrites, ["foo ", "bar"]);
});

it("all clipboard mirror policy mirrors change writes", async () => {
  const { editor, clipboardWrites } = createEditorWithSpy("foo bar");
  editor.setClipboardMirrorPolicy("all");

  sendKeys(editor, ["c", "w"]);
  await nextImmediate();

  assert.equal(editor.getRegister(), "foo ");
  assert.deepEqual(clipboardWrites, ["foo "]);
});

it("yank clipboard mirror policy skips delete writes but updates the register", () => {
  const { editor, clipboardWrites } = createEditorWithSpy("foo bar");
  editor.setClipboardMirrorPolicy("yank");

  sendKeys(editor, ["d", "w"]);

  assert.equal(editor.getRegister(), "foo ");
  assert.deepEqual(clipboardWrites, []);
});

it("yank clipboard mirror policy skips change writes but updates the register", () => {
  const { editor, clipboardWrites } = createEditorWithSpy("foo bar");
  editor.setClipboardMirrorPolicy("yank");

  sendKeys(editor, ["c", "w"]);

  assert.equal(editor.getRegister(), "foo ");
  assert.deepEqual(clipboardWrites, []);
});

it("yank clipboard mirror policy skips mutation writes", async () => {
  const { editor, clipboardWrites } = createEditorWithSpy("foo bar");
  editor.setClipboardMirrorPolicy("yank");

  sendKeys(editor, ["d", "w", "y", "w", "c", "w"]);
  await nextImmediate();

  assert.equal(editor.getRegister(), "bar");
  assert.deepEqual(clipboardWrites, ["bar"]);
});

it("never clipboard mirror policy keeps mutation and yank writes internal", () => {
  const { editor, clipboardWrites } = createEditorWithSpy("foo bar");
  editor.setClipboardMirrorPolicy("never");

  sendKeys(editor, ["y", "y"]);

  assert.equal(editor.getRegister(), "foo bar\n");
  assert.deepEqual(clipboardWrites, []);

  sendKeys(editor, ["d", "w"]);

  assert.equal(editor.getRegister(), "foo ");
  assert.deepEqual(clipboardWrites, []);
});

it("never clipboard mirror policy keeps change writes internal", () => {
  const { editor, clipboardWrites } = createEditorWithSpy("foo bar");
  editor.setClipboardMirrorPolicy("never");

  sendKeys(editor, ["c", "w"]);

  assert.equal(editor.getRegister(), "foo ");
  assert.deepEqual(clipboardWrites, []);
});

  for (const policy of ["all", "yank", "never"] as const) {
    it(`${policy} clipboard mirror policy keeps p reading OS clipboard`, () => {
    const { editor } = createEditorWithSpy("ab");
    editor.setClipboardMirrorPolicy(policy);
    editor.setRegister("shadow");
    editor.setClipboardReadFn(() => "SYS");

    sendKeys(editor, ["p"]);

    assert.equal(editor.getText(), "aSYSb");
    assert.equal(editor.getRegister(), "shadow");
  });

    it(`${policy} clipboard mirror policy keeps P reading OS clipboard`, () => {
    const { editor } = createEditorWithSpy("ab");
    editor.setClipboardMirrorPolicy(policy);
    editor.setRegister("shadow");
    editor.setClipboardReadFn(() => "SYS");

    sendKeys(editor, ["P"]);

    assert.equal(editor.getText(), "SYSab");
    assert.equal(editor.getRegister(), "shadow");
    });
  }
});
```

- [ ] **Step 2: Run tests and verify they fail for missing editor API**

Run:

```bash
node --import tsx/esm --test test/clipboard-policy-editor.test.ts
```

Expected: FAIL because `setClipboardMirrorPolicy()` is not implemented.

### Task 4: Implement editor policy gating

**Files:**
- Modify: `index.ts`

- [ ] **Step 1: Import policy types and constants**

Add near existing local imports:

```ts
import {
  DEFAULT_CLIPBOARD_MIRROR_POLICY,
  type ClipboardMirrorPolicy,
  type RegisterWriteSource,
} from "./clipboard-policy.js";
```

- [ ] **Step 2: Add policy state and test seam to `ModalEditor`**

Near `private unnamedRegister`, add:

```ts
private clipboardMirrorPolicy: ClipboardMirrorPolicy = DEFAULT_CLIPBOARD_MIRROR_POLICY;
```

Near other test seams, add:

```ts
setClipboardMirrorPolicy(policy: ClipboardMirrorPolicy): void {
  this.clipboardMirrorPolicy = policy;
}

getClipboardMirrorPolicy(): ClipboardMirrorPolicy {
  return this.clipboardMirrorPolicy;
}
```

- [ ] **Step 3: Replace `writeToRegister(text)` with policy-aware source handling**

Replace the method with:

```ts
private shouldMirrorRegisterWrite(source: RegisterWriteSource): boolean {
  if (this.clipboardMirrorPolicy === "never") return false;
  if (this.clipboardMirrorPolicy === "yank") return source === "yank";
  return true;
}

private writeToRegister(text: string, source: RegisterWriteSource = "mutation"): void {
  this.unnamedRegister = text;
  if (!text) return;
  if (!this.shouldMirrorRegisterWrite(source)) return;

  this.clipboardMirror.mirror(text);
}
```

- [ ] **Step 4: Mark yank paths as `source = "yank"`**

Update only register writes inside these yank-specific helpers:

- `yankLineRange()`
- `yankRange()`
- `yankRangeByAbsolute()`

Use these replacement shapes:

```ts
this.writeToRegister(this.getLinewisePayload(startLine, endLine), "yank");
this.writeToRegister(line.slice(start, end), "yank");
this.writeToRegister(text.slice(start, end), "yank");
```

Do not change delete/change calls; they should keep the default `mutation` source.

- [ ] **Step 5: Run policy tests and verify they pass**

Run:

```bash
node --import tsx/esm --test test/clipboard-policy-editor.test.ts
```

Expected: PASS for editor behavior tests.

- [ ] **Step 6: Commit editor policy behavior**

Run:

```bash
git add index.ts test/clipboard-policy-editor.test.ts
git commit -m "feat(clipboard): apply mirror policy to register writes"
```

---

## Chunk 3: Extension settings integration, docs, and gates

### Task 5: Add failing extension-startup settings tests

**Files:**
- Modify: `test/modal-editor.test.ts`

- [ ] **Step 1: Extend extension test context with `cwd`, `hasUI`, and captured notifications**

In `installExtensionWithEditorFactory()`, extend `InstalledExtension` and the test context so settings tests can inspect warning text and severity:

```ts
type NotificationCall = { message: string; type: string };

type InstalledExtension = {
  editorFactory: EditorFactory;
  readonly notificationCalls: number;
  readonly notifications: NotificationCall[];
  readonly shutdownCalls: number;
  emitShutdown(): Promise<void>;
  readonly sessionShutdownHandlerCount: number;
  readonly sessionEndHandlerCount: number;
};

let notificationCalls = 0;
let shutdownCalls = 0;
const notifications: NotificationCall[] = [];
const ctx = {
  cwd: process.cwd(),
  hasUI: true,
  ui: {
    theme: stubTheme,
    setEditorComponent(factory: EditorFactory): void {
      editorFactory = factory;
    },
    notify(message: string, type: string): void {
      notificationCalls++;
      notifications.push({ message, type });
    },
  },
  shutdown(): void {
    shutdownCalls++;
  },
};
```

In the returned object from `installExtensionWithEditorFactory()`, add:

```ts
get notifications() {
  return notifications;
},
```

- [ ] **Step 2: Add settings-reader test seam import**

Add `setPiVimSettingsReaderForTests` to the `../clipboard-policy.js` import or create that import if one does not exist:

```ts
import { setPiVimSettingsReaderForTests } from "../clipboard-policy.js";
```

- [ ] **Step 3: Add extension-startup tests**

Create `describe("clipboard mirror policy settings", ...)` before `describe("cursor shape lifecycle", ...)` in `test/modal-editor.test.ts`:

```ts
describe("clipboard mirror policy settings", () => {
  it("applies piVim clipboardMirror settings to installed editors", async () => {
  const restore = setPiVimSettingsReaderForTests(() => ({ clipboardMirror: "never" }));
  try {
    const extension = await installExtensionWithEditorFactory();
    const editor = extension.editorFactory(stubTui, stubTheme, stubKeybindings);

    assert.equal(editor.getClipboardMirrorPolicy(), "never");
    assert.equal(extension.notificationCalls, 0);
  } finally {
    restore();
  }
});

  it("warns and falls back when piVim clipboardMirror is invalid", async () => {
  const restore = setPiVimSettingsReaderForTests(() => ({ clipboardMirror: "delete" }));
  try {
    const extension = await installExtensionWithEditorFactory();
    const editor = extension.editorFactory(stubTui, stubTheme, stubKeybindings);

    assert.equal(editor.getClipboardMirrorPolicy(), "all");
    assert.equal(extension.notificationCalls, 1);
    assert.equal(extension.notifications[0]?.type, "warning");
    assert.match(extension.notifications[0]?.message ?? "", /delete/);
    assert.match(extension.notifications[0]?.message ?? "", /all, yank, never/);
  } finally {
    restore();
  }
  });
});
```

- [ ] **Step 4: Run tests and verify they fail for missing startup integration**

Run:

```bash
node --import tsx/esm --test --test-name-pattern "clipboard mirror policy settings" 'test/**/*.test.ts'
```

Expected: FAIL because startup settings application is not implemented.

### Task 6: Implement extension startup settings integration

**Files:**
- Modify: `index.ts`

- [ ] **Step 1: Import settings reader and resolver**

Extend the existing `./clipboard-policy.js` import:

```ts
import {
  DEFAULT_CLIPBOARD_MIRROR_POLICY,
  readPiVimSettings,
  resolveClipboardMirrorPolicy,
  type ClipboardMirrorPolicy,
  type RegisterWriteSource,
} from "./clipboard-policy.js";
```

- [ ] **Step 2: Apply resolved policy during `session_start`**

At the start of the `session_start` handler:

```ts
const piVimSettings = readPiVimSettings(ctx.cwd);
const clipboardMirrorPolicy = resolveClipboardMirrorPolicy(piVimSettings.clipboardMirror);
if (clipboardMirrorPolicy.warning && ctx.hasUI) {
  ctx.ui.notify(clipboardMirrorPolicy.warning, "warning");
}
```

Inside the editor factory, before `setQuitFn()`:

```ts
editor.setClipboardMirrorPolicy(clipboardMirrorPolicy.policy);
```

- [ ] **Step 3: Run startup settings tests and verify they pass**

Run:

```bash
node --import tsx/esm --test --test-name-pattern "clipboard mirror policy settings" 'test/**/*.test.ts'
```

Expected: PASS for resolver and extension-startup settings tests.

- [ ] **Step 4: Commit startup settings integration**

Run:

```bash
git add index.ts test/modal-editor.test.ts
git commit -m "feat(clipboard): load mirror policy from settings"
```

### Task 7: Update README configuration docs

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add high-up configure section after install**

Insert after `Restart Pi after install.`:

````md
## configure

pi-vim reads persistent Pi settings from `~/.pi/agent/settings.json` and project `.pi/settings.json`.

Clipboard write mirroring is controlled by `piVim.clipboardMirror`:

```json
{
  "piVim": {
    "clipboardMirror": "all"
  }
}
```

| value | behavior |
|-------|----------|
| `all` | Mirror every unnamed-register write (default/current behavior) |
| `yank` | Mirror yanks only; deletes/changes update only pi-vim's internal register |
| `never` | Never mirror register writes to the OS clipboard |

The setting controls write mirroring only. `p` / `P` keep the paste policy documented below.
````

- [ ] **Step 2: Update edit operator wording**

Replace:

```md
All operators write to the unnamed register and mirror to the system clipboard
(best-effort; clipboard failure never breaks editing).
```

with:

```md
Register-writing edits write to the unnamed register. With the default clipboard mirror policy, they also mirror to the system clipboard best-effort (clipboard failure never breaks editing).
```

- [ ] **Step 3: Update register/clipboard policy bullets**

Replace the mirroring bullets with:

```md
- `piVim.clipboardMirror = "all"` is the default: every unnamed-register write mirrors to the OS clipboard best-effort.
- `piVim.clipboardMirror = "yank"` mirrors yanks only; deletes and changes update only pi-vim's internal shadow.
- `piVim.clipboardMirror = "never"` disables write mirroring while keeping internal register writes synchronous.
- Rapid mirrored writes coalesce: only the latest pending value is guaranteed to be mirrored.
```

Keep the existing `p` / `P` bullets unchanged.

- [ ] **Step 4: Run a focused docs sanity check**

Run:

```bash
rg -n "vim-clipboard-mirror|piVim.clipboardMirror|register and clipboard policy|## configure" README.md
```

Expected: no `vim-clipboard-mirror` output; `piVim.clipboardMirror`, `register and clipboard policy`, and `## configure` appear.

- [ ] **Step 5: Commit docs**

Run:

```bash
git add README.md
git commit -m "docs(clipboard): document mirror policy settings"
```

### Task 8: Run full verification gates

**Files:**
- Verify only; no planned file edits.

- [ ] **Step 1: Run lint**

Run:

```bash
npm run lint
```

Expected: exit 0; Biome reports checked files and ESLint reports no errors.

- [ ] **Step 2: Run typecheck**

Run:

```bash
npm run typecheck
```

Expected: exit 0.

- [ ] **Step 3: Run full check**

Run:

```bash
npm run check
```

Expected: exit 0; test summary reports 0 failures.

- [ ] **Step 4: Run package check**

Run:

```bash
npm run pack:check
```

Expected: exit 0; `pack:check passed`.

- [ ] **Step 5: Run lefthook pre-commit**

Run:

```bash
npx lefthook run pre-commit --all-files
```

Expected: exit 0; lint and typecheck pass.

- [ ] **Step 6: Run lefthook pre-push**

Run:

```bash
npx lefthook run pre-push --all-files
```

Expected: exit 0; check and pack pass.

- [ ] **Step 7: Commit any final test/doc fixes**

If any gate required fixes, commit only the touched files:

```bash
git add <fixed-files>
git commit -m "fix(clipboard): address mirror policy review"
```

Expected: no commit is needed when all gates pass without changes.
