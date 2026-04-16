# CODEBUDDY.md This file provides guidance to CodeBuddy when working with code in this repository.

## Commands

### Build

```bash
pnpm install   # Install dependencies
pnpm build     # Build with Vite, then copy LICENSE, README.md, icon.png, package.json into ./dist/
```

There is no `dev` script defined. For local development, load the plugin manually in Logseq by pointing it to the project root (which serves `index.html` as the entry point) or run `npx vite` to start the Vite dev server.

### Lint

No lint script is configured in `package.json`. ESLint and `@typescript-eslint` are installed as devDependencies. Run manually:

```bash
npx eslint src/
```

### Test

No test framework is configured. There are no tests in this project.

## Architecture

This is a **Logseq plugin** written in TypeScript, bundled with Vite. The entire plugin logic lives in a single file: `src/main.ts`.

### Entry Point & Plugin Lifecycle

`index.html` is the plugin entry point declared in `package.json` (`"main": "index.html"`). It loads `src/main.ts` as an ES module. At the bottom of `main.ts`, `logseq.ready(main)` bootstraps the plugin after the Logseq SDK is ready.

### Core Dependency

- **`@logseq/libs` (v0.0.17)**: The Logseq Plugin SDK. All interaction with Logseq (reading/writing blocks, registering commands, opening the sidebar, accessing user configs) goes through the `logseq` global provided by this library.
- **`date-fns`**: Used solely for `format()` to produce today's date string matching the user's `preferredDateFormat` from Logseq settings.

### Settings System

The plugin defines a versioned settings object (`settingsVersion: "v2"`) with defaults for `showToolbarIcon`, `keyBindings`, and `disabled`. On startup, `initSettings()` checks if stored settings match the current version and resets them if not. A single user-facing setting is registered via `logseq.useSettingsSchema()`:

- **`putBlockRefAsChild`** (boolean): Controls comment insertion behavior — whether the block reference is added as a child of a new empty block, or the comment is added as a child of the block reference itself.

### Command Registration (`main()`)

The `main()` function registers two commands through three surfaces each:

1. **"Comment block"** → `handler`
   - Slash command (`/Comment block`)
   - Block context menu item
   - Command palette with global shortcut (`mod+shift+i`)

2. **"Embed Comment blocks To Children"** → `handleEmbed`
   - Slash command
   - Block context menu item
   - Command palette (no shortcut)

### `handler` — Comment Block Logic

This is the core workflow. Given a block (from event `e.uuid` or the currently focused block):

1. Ensures the block has an `id` property (upserts if missing).
2. Finds the page the block belongs to.
3. Looks for a top-level `[[Comments]]` block on that page; creates one (collapsed, appended as the last sibling) if absent.
4. Under `[[Comments]]`, looks for a child block matching today's date as `[[<todayTitle>]]`; creates one (collapsed) if absent.
5. Depending on the `putBlockRefAsChild` setting:
   - **Enabled**: Inserts an empty block under today's date, adds the original block's reference `((<uuid>))` as its child, opens the empty block in the right sidebar, and starts editing it.
   - **Disabled** (default): Inserts a `((<uuid>))` block ref under today's date (reuses if it already exists), then inserts an empty child block under that ref for the user to type in, opens the ref block in the right sidebar.

The resulting block tree on the page looks like:

```
[[Comments]]       (collapsed)
  └── [[2024-01-15]]   (collapsed, today's date)
        └── ((<uuid>))   (block reference)
              └── <user comment>
```

### `handleEmbed` — Embed Comments to Children

Traverses the `[[Comments]]` block tree (3 levels deep: date → block ref → comment children). For each comment child block found under a matching block ref `((<uuid>))`, it ensures the child has an `id` property, then inserts a block reference `((<child-uuid>))` as a child of the original block. This effectively "pulls" all comment history back into the original block as embedded references.

### Build Output

Vite builds to `./dist/` (default). The build script also copies `LICENSE`, `README.md`, `icon.png`, and `package.json` into `dist/` so the folder is a self-contained distributable plugin package. Vite config sets `base: './'` for relative asset paths and splits `@logseq/libs` into a separate chunk (`logseq`).
