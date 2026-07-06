# Copilot instructions for PocketCal

## Build, lint, and test commands

Use npm scripts from `package.json`:

- `npm ci` — install dependencies from lockfile.
- `npm run dev` — start Vite dev server.
- `npm run build` — TypeScript project build (`tsc -b`) plus Vite production bundle.
- `npm run lint` — run ESLint across the repo.
- `npm run preview` — serve the built app locally.

Automated tests are not currently configured in this repository (no `test` script and no test files), so there is no single-test command yet.

## High-level architecture

- PocketCal is a client-side React + TypeScript + Vite app. App bootstrap is `src/main.tsx` -> `src/App.tsx`.
- Shared application state lives in a single Zustand store (`src/store.ts`), including:
  - calendar settings,
  - event groups and date ranges,
  - UI mode flags (help modal, embed mode, pro status),
  - URL serialization/deserialization logic.
- The main UI is split into:
  - `src/components/Sidebar.tsx` for group management and calendar settings,
  - `src/components/Calendar.tsx` for 12-month rendering and date interactions.
- Calendar data is link-backed (README: “no accounts, just a link”):
  - state is encoded into the URL hash by `generateShareableUrl` in `store.ts`,
  - decoded on load by `getAppStateFromUrl` in `store.ts`.
- Share surfaces:
  - embed mode (`?embed=true`) is handled in `App.tsx` with read-only calendar behavior and embed-specific styling in `src/App.css`,
  - ICS export is generated in `src/ics.ts` and triggered from `src/components/EmbedModal.tsx`.
- “Pro” license validation uses a Netlify serverless function:
  - frontend calls `/.netlify/functions/validate-license` from `validateLicenseKey` in `store.ts`,
  - backend proxy is `netlify/functions/validate-license.js` (Lemon Squeezy API).

## Key repository-specific conventions

- **URL hash schema is compact and intentionally sparse.** `store.ts` uses compressed keys (`s`, `w`, `t`, `g`, `f`) and omits defaults before compression. If you add shareable state, update both `generateShareableUrl` and `getAppStateFromUrl`.
- **Date ranges are stored as ISO date strings** (`yyyy-MM-dd`) in each event group (`DateRange.start/end`), and many operations depend on `date-fns` `parseISO`/`formatISO`.
- **Event groups are color-index driven.** Colors must come from `GROUP_COLORS`; share payloads store color indexes (`c`), not raw hex strings.
- **Group limits are plan-dependent.** Free/pro limits are enforced through `getMaxGroups(isProUser)`; keep this in sync with help/pro UI text (`HelpModal`, `LicenseModal`, `Sidebar`).
- **Theme is controlled by `data-theme` on `<html>`.** `App.tsx` applies `"light" | "dark" | "system"` and `src/index.css` is built around CSS custom properties for both themes.
- **Embed mode must stay read-only.** Calendar interaction guards use `isEmbedMode` checks in `Calendar.tsx`; embed visuals are styled via `.app-container.embed-mode` rules in `App.css`.
- **Pro state is cached in localStorage.** Existing keys are `pocketcal_license`, `pocketcal_pro_validated`, and `pocketcal_theme`; preserve these unless migration is intentional.
