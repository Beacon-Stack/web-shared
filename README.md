# web-shared

Shared React/TypeScript components used by every Beacon service's frontend (Pilot, Prism, Haul, Pulse).

## Why this exists

Before this directory, each service either had a duplicated local copy of components like `Modal.tsx` and `ConfirmDialog.tsx` (Pilot/Prism kept drifting toward incidental divergence), or simply didn't have them at all (Haul and Pulse used raw `window.confirm()`, leaking the browser's ugly native dialog into the UX). The "dead torrent delete shows a browser prompt" bug was the symptom that surfaced this class.

Now there's one canonical source. If you fix a bug or tweak the styling in `Modal.tsx`, every service picks it up on its next frontend build. No manual sync, no drift.

## How it works

Each service's `vite.config.ts` adds an alias:

```ts
resolve: {
  alias: {
    "@": path.resolve(__dirname, "./src"),
    "@beacon-shared": path.resolve(__dirname, "../../../web-shared"),
  },
},
```

And each service's `tsconfig.app.json` mirrors it for type resolution:

```json
"paths": {
  "@": ["./src/*"],
  "@beacon-shared/*": ["../../../web-shared/*"]
}
```

Then any component in the service imports like:

```tsx
import Modal from "@beacon-shared/Modal";
import { useConfirm, ConfirmProvider } from "@beacon-shared/ConfirmDialog";
```

## Contents

- `Modal.tsx` — Generic modal shell (backdrop, Escape, click-outside, width/height controls).
- `ConfirmDialog.tsx` — `ConfirmProvider` + `useConfirm()` hook. Replaces `window.confirm()` everywhere.
- `UpdateCheck.tsx` — "Check for updates" and "Release notes" buttons and windows for a System page. Calls the app's `GET /api/v1/system/updates` (backed by `internal/updatecheck`, identical in each app) and shows the newer commits and the commands to update. Props: `app`, `service`, `image`.
- `Markdown.tsx` — Safe renderer for release-notes markdown (headings, lists, bold, code, https links, `<details>` folds). Builds React elements; never sets innerHTML.
- `UpdatedNotice.tsx` — "App was updated, reload" notice for a page left open across an update. Mount once in the app shell (the Shell `overlay` slot). Props: `app`, `versionPath` (default `/system/status`).
- `SystemInfo.tsx` — the About card at the top of every System page: version, build time, start time, uptime, database, Go version, API docs link and the update check. Reads `GET /api/v1/system/status` (same shape in all four apps). Props: `app`, `service`, `image`, optional `extraRows`.
- `SystemCard.tsx` — the frame every System page section sits in (`title`, `subtitle`, `action`), plus `SystemRow` for label/value lines.
- `TasksCard.tsx` — scheduled tasks from `GET /api/v1/tasks` with a Run now button each (Pilot, Prism).
- `HealthCard.tsx` — the Health card: overall line, one line per check or problem, Check now. Each app maps its own health data to `overall` + `lines`.
- `Discover.tsx` + `discoverState.ts` — the Discover page in Pilot and Prism: search box, the five lists (Trending, Popular, Top rated, New on streaming, Coming soon), genre / streaming service / rating filters, grid, loading, empty, paging and "TMDB isn't set up". State lives in the URL (`useDiscoverState`), so Back from a title returns to the same list. `discoverRegion()` picks the country from the browser locale. Each app supplies its own result card and calls `/api/v1/discover/{browse,search,providers}`.
- `Notifications.tsx` — the Notifications settings page, the same in Pulse, Pilot, Prism and Haul. Talks to `/api/v1/notifications/*` (`pulse/pkg/notify/notifyapi`): channel list with on/off, Send test, edit and remove; an add/edit form built from each channel type's settings; which events (and, in Pulse, which apps); a banner saying whether the app sends through Pulse; recently sent.

**System pages** are laid out the same in every app: `/settings/system` (sidebar "System", server icon) and `/settings/system/logs` ("Logs"); page description "What's running and how it's doing."; then About (`SystemInfo`), Health, then the app's own sections (Tasks, Haul's Maintenance).

## Rules for adding components

1. **Only add things that ≥3 services need** (or will need within the next few commits). This isn't a dumping ground for "it might be useful someday".
2. **No service-specific deps.** Pure React + TypeScript + CSS custom properties. No imports from `@/` paths.
3. **Props must be stable.** Changes here break every service simultaneously — treat it like a public API.
4. **Test before committing.** Run `npm run build` in at least one service that imports the component.
5. **Don't create parallel versions.** If you need a variant, add a prop or make two components, but never fork.

## Gotchas

- The alias targets a relative path (`../../../web-shared`). That assumes a service lives at `<service>/web/ui/`. If the directory layout moves, update the aliases.
- Because this isn't an npm package, there's no versioning. Breaking changes ship to every service at once. Make them rarely and intentionally.
- Vitest (used by Pilot's test suite) picks up the alias from `vite.config.ts` automatically. No extra test-config needed.
