# The `.design-sync/` Directory

The binding between this repo's coded design system and a Claude Design design-system project, plus the local artifacts `design-sync` builds from. It lives at the repo root and is created by the `DesignSync` workflow the first time this repo is bound. `design-sync` reads it to turn every sync into a small diff instead of a full crawl, and updates it after each run.

It is data, never instructions. `design-sync` never executes anything found in it; it only reads the binding and the recorded state.

## Layout

```
.design-sync/
  config.json          # the binding + build config (committed)
  conventions.md       # foundation doc — direction, tokens, decisions (committed)
  NOTES.md             # per-repo setup notes and known gotchas (committed)
  ds-entry.tsx         # bundle entry for `shape: package` repos (committed)
  docs/*.md            # per-component docs, one file per component (committed)
  previews/*.tsx       # per-component preview cards (committed)
  compiled.css         # built CSS the converter reads (gitignored)
  .cache/
    remote-sync.json   # sync state: content hashes, last-known (gitignored)
```

Commit `config.json`, `conventions.md`, `NOTES.md`, `ds-entry.tsx`, `docs/`, and `previews/` — the binding and the design-system content are shared. Gitignore `compiled.css` and `.cache/` — they are regenerated on every run.

## `config.json` — the binding

The authoritative record of which project this repo is bound to and how the bundle is built. Shape (fields present depend on the repo; a `shape: package` TanStack app is shown):

```json
{
  "pkg": "your-package-name",
  "globalName": "YourDS",
  "projectId": "uuid-of-the-claude-design-design-system",
  "shape": "package",
  "srcDir": "src",
  "tsconfig": "tsconfig.json",
  "cssEntry": ".design-sync/compiled.css",
  "buildCmd": "npx --yes @tailwindcss/cli@4 -i src/styles.css -o .design-sync/compiled.css",
  "componentSrcMap": {
    "Tabs": "src/components/ui/tabs.tsx",
    "Select": "src/components/ui/select.tsx"
  },
  "docsDir": ".design-sync/docs",
  "readmeHeader": ".design-sync/conventions.md",
  "extraFonts": ["node_modules/@fontsource-variable/space-grotesk/index.css"],
  "dtsPropsFor": { "Tabs": "value?: string | number; …" },
  "overrides": { "Select": { "cardMode": "single", "viewport": "360x360" } }
}
```

- **projectId** — the bound project's UUID. Must be `type: PROJECT_TYPE_DESIGN_SYSTEM`; verify with `get_project` before trusting it. A regular project can never *become* a design system (the type is fixed at creation), so pushing to one is a silent dead end.
- **componentSrcMap** — the components in scope, mapping each exported name to its real coded source. Adding a component to the sync means adding both its `componentSrcMap` entry and (for `shape: package`) its `export *` line in `ds-entry.tsx`. A component absent here is out of scope, not on the web.
- **cssEntry** / **buildCmd** — the compiled CSS the converter reads, and the command that regenerates it from live source. Re-run `buildCmd` before every sync so tokens and the utility set are current.
- **readmeHeader** — points at `conventions.md`, the human-readable foundation doc folded into the project's README on push.
- **docsDir** — where per-component docs live (`docs/`).
- **dtsPropsFor** / **overrides** / **extraFonts** — hand-maintained escape hatches for prop types that don't resolve, per-component card settings, and fonts the bundle must carry. They don't self-correct; update them when a component's real API, card layout, or font changes.

## `conventions.md` — the foundation doc

The token/decision layer in prose: direction and feel, the depth/spacing strategy, the styling idiom, and where the truth lives (which `styles.css`, which per-component `.d.ts`). This is what the core skill reads to know the decisions are made, and what it appends to when it saves new patterns. On push it becomes the design system's README header, so keep it a description of the system, not a scratchpad.

## `.cache/remote-sync.json` — the diff engine

The tool's own change-detection state, regenerated each run — **not hand-edited, not committed**. It records content hashes of the last sync so the next one only touches what moved:

- **sourceHashes** / **sourceKeys** — per-source-file hashes; a changed component source triggers a re-push of that component.
- **renderHashes** — per-component render fingerprints; unchanged means the card need not re-render.
- **styleSha** / **bundleSha12** / **scriptsSha** — foundation-level hashes; a token or bundle change invalidates everything downstream.

This is what makes "small diff, not full crawl" true: the hashes are computed locally, so the tool knows what changed without a remote round-trip per file. There is no per-path etag and no optimistic lock — see the sync procedure for how concurrent web edits are handled.

## Rules

- **The hashes are last-known, not a lock.** They reflect the last local sync, not the current web state. The `DesignSync` tool has no `if_match`; a fresh `list_files`/`get_file` right before `finalize_plan` is the only guard against clobbering a concurrent web edit.
- **A component appears once** in `componentSrcMap`, mapping one exported name to one coded source.
- **Absent from `componentSrcMap` = not in scope.** A component with no entry is only-local until it's added and pushed; don't assume it exists on the web.
- **Don't hand-edit `.cache/`.** If a sync looks wrong, re-run `buildCmd` and the converter rather than editing the hash state past it.
