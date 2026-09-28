# BusGo — Intercity Transport Booking App

A multi-role intercity bus booking platform for Ethiopia. Passengers search routes, pick a seat, pay via
mobile money or card, receive a QR ticket, and track their bus live. Companion driver, admin, and
station-agent surfaces cover trip operations and fleet dispatch.

Built with React 19, Vite 8, TypeScript, and Tailwind CSS v4. This is a UI prototype: all data is
in-memory mock data and every flow is driven by local React state. There is no backend, no router
library, and no test suite.

## Quick start

The project uses [mise](https://mise.jdx.dev) to pin the toolchain (Node 22, pnpm 10.34.3).

```bash
mise install          # install pinned Node + pnpm
pnpm install
pnpm dev              # http://localhost:8443
```

Inside Figma Make the dev server is already running on `$PORT` (default `8443`) and the app is
reachable from the preview panel — no need to start it manually.

### Scripts

| Script            | What it does                                          |
| ----------------- | ----------------------------------------------------- |
| `pnpm dev`        | Vite dev server on `0.0.0.0:$PORT` (default `8443`)    |
| `pnpm build`      | Production build into `dist/`                          |
| `pnpm preview`    | Serve the built `dist/` output                         |
| `pnpm format`     | Format the repo with `oxfmt`                           |

There is no lint script and no test runner configured. `tsconfig.json` is present but not wired to a
script, so type checking is run manually with `npx tsc --noEmit` — note that it currently fails with
one pre-existing error (see [Known gaps](#known-gaps)).

### Environment variables

| Variable             | Used by             | Default                            |
| -------------------- | ------------------- | ---------------------------------- |
| `PORT`               | `vite.config.ts`    | `8443`                             |
| `FIGMA_PUBLIC_URL`   | `vite.config.ts`    | `/` (used as the build `base`)     |

## Preview harness

`src/App.tsx` is a prototype shell rather than a plain app render. It renders a sidebar with:

- an **app mode switcher** (Passenger / Driver / Admin Dashboard),
- a **language toggle** (English / Afaan Oromo),
- a **direct screen navigator** that jumps to any passenger screen without walking the flow,
- a **phone frame** (320 × 740 viewport) containing the active screen, plus a 640px-wide panel for
  the admin dashboard.

The bottom tab bar is hidden on `splash`, `auth`, `driver`, and `admin`.

## Architecture

Navigation is a hand-rolled state machine, not a routing library. `App.tsx` holds a `Screen` union
and a `go()` setter, and `renderScreen()` switches over it. Shared state lives in the same component:

```
src/App.tsx     screen state, selected bus/seat, search query, app mode, language,
                bus list mutations (add bus, mark "no car")
src/main.tsx    React entrypoint, mounts App into #root
src/index.css   Tailwind v4 import, @theme design tokens, global utilities
src/data.ts     types, mock data, i18n strings, route/format helpers
```

### Screens (`src/screens/`)

Twenty components, each a default export receiving props and callbacks:

`Splash`, `Auth`, `Home`, `BusList`, `BusDetail`, `SeatSelection`, `BookingSummary`, `Payment`,
`PaymentSuccess`, `Ticket`, `LiveTracking`, `Queue`, `MyBookings`, `Notifications`, `Profile`,
`Settings`, `LanguageSelect`, `DriverApp`, `AdminDashboard`, `StationDashboard`

### Shared components (`src/components/`)

- `Icons.tsx` — ~30 inline SVG icons (`HomeIcon`, `BusIcon`, `QRIcon`, `TrendUpIcon`, …), all
  accepting a `size` and `className`.
- `StatusBadge.tsx` — pill badge for the eight statuses used across the app (`confirmed`, `pending`,
  `cancelled`, `completed`, `on_time`, `delayed`, `active`, `inactive`).

### Data and i18n (`src/data.ts`)

- `Lang = "en" | "om"` and a `t` dictionary of English / Afaan Oromo strings. Screens take `lang` as
  a prop and read `t.<key>[lang]`.
- `Bus`, `StationAgent` interfaces plus mock collections: `BUSES`, `BOOKINGS`, `NOTIFICATIONS`,
  `ADMIN_STATS`, `INITIAL_STATION_AGENTS`, `CITIES` (20 Ethiopian cities).
- Helpers: `getRouteDistance()` (direct table, then haversine over `CITY_COORDS` with a 1.3 road
  factor), `formatDuration()` (assumes ~55 km/h), `getStationName()` (routes Addis Ababa departures to
  the correct terminal — Kality, Lamberet, or Autobis Tera).

## Feature tour

**Passenger.** Search by origin/destination/date/passengers, browse and sort available buses, inspect
bus detail (amenities, rating, station terminal, driver), pick a seat on an interactive grid, review
the booking summary, pay, then view a QR ticket. Also: live tracking with map and ETA, station queue
position, booking history, notifications, profile, and settings.

**Driver.** Today's trip, passenger manifest with boarded/waiting status, simulated QR ticket
verification, and start/end trip controls.

**Admin.** Wide desktop dashboard with stat cards and tabs for station agents, users, buses, trips,
bookings, routes, payments, and reports.

**Station agent.** Per-station console for registering departing cars, toggling whether a car actually
exists, and marking routes as having no service.

### Dispatch model

Seat availability is not a simple countdown. It reflects a chain of custody:

1. Station agents register cars departing from their assigned station (`StationDashboard`).
2. Passengers at a station can report whether a car is genuinely present and boarding.
3. A station report of "no car" sets the bus to `cancelled` and records the reporter, station,
   timestamp, and note on `bus.reportedByPassenger`.
4. `BusList` surfaces the note and an "alert recorded" confirmation back to the passenger.

## Design system

Tokens are declared once in the `@theme` block of `src/index.css` (Tailwind v4 CSS-first config — no
`tailwind.config.js` or PostCSS setup):

- **Color** — `navy` (950 → 50) for surfaces and text, `brand` (blue 600 → 50) for primary actions,
  plus `amber`, `emerald`, and `rose` for status accents.
- **Type** — Plus Jakarta Sans (display), DM Sans (body), JetBrains Mono (codes, times, plates).
- **Custom utilities** — `.map-bg` (grid map placeholder), `.qr-pattern`, `.route-line`, `.live-dot`
  (pulse animation), `.seat-available` / `.seat-selected` / `.seat-booked` / `.seat-driver`,
  `.screen-enter` (screen transition), `.admin-table`.
- Scrollbars are hidden by default and appear at 4px on hover — the app is a fixed-height phone frame,
  so scrollbars are treated as visual noise.

Add new theme values to the `@theme` block rather than a separate config file. Put `@font-face` rules
and any global CSS in `src/index.css`, after the `@import` statements.

## Figma Make integration

`vite.config.ts` registers platform plugins alongside React and Tailwind:

- `figmaSiteConfiguration()` — applies `.figma/make/site.json` (title, description, favicon, Open Graph,
  Google Analytics, custom script slots, `robots.txt`, bypass links) to the generated HTML shell.
  Placeholders in `index.html` use `<!-- figma:lang -->`, `<!-- figma:title -->`, `<!-- figma:head-start -->`,
  `<!-- figma:head-end -->`, `<!-- figma:body-start -->`, `<!-- figma:body-end -->`.
- `figmaErrorOverlayReplay()` — dev-only. Replays the last build error to sockets that connect after
  Vite already delivered it, so the preview iframe does not show a stale blank screen.
- `figmaReactRefreshBoundaryFallback()` — dev-only. Forces a full reload when a module stops defining
  a React Refresh boundary (typically after moving a component and replacing the old file with a
  re-export).
- `figmaMakeKitPlugin()` — dev-only. Serves `/.figma/make/kit.html`, which exposes any
  `src/**/*.stories.*` modules on `window.__FIGMA__.stories` for the design surface to mount.

Building with `--mode development` produces inline sourcemaps and disables minification (used by the
platform's cached preview builds).

## Project layout

```
├── .figma/make/            platform config (site.json, dev/build scripts)
├── src/
│   ├── components/         Icons.tsx, StatusBadge.tsx
│   ├── screens/            20 screen components
│   ├── App.tsx             preview shell + navigation state machine
│   ├── data.ts             types, mock data, i18n, helpers
│   ├── index.css           Tailwind import, @theme tokens, utilities
│   └── main.tsx            React entrypoint
├── AGENTS.md               repo conventions for agents
├── CLAUDE.md               points at AGENTS.md
└── vite.config.ts          React + Tailwind + Figma Make plugins, @ alias
```

## Conventions

- Export components as **default** exports.
- The `@` alias resolves to `src` (configured in both `vite.config.ts` and `tsconfig.json`).
- Use double quotes for strings containing apostrophes (`"We're here to help"`) or escape them in
  single-quoted strings — an unescaped apostrophe breaks the build.
- Close all JSX tags and keep braces balanced.
- `AGENTS.md` holds the fuller list of conventions and the canonical project structure.

## Known gaps

- All state is in-memory; reloading resets to the splash screen.
- `npx tsc --noEmit` reports one error: `App.tsx` passes `onAddBus` and `onMarkNoCar` to `<BusList />`,
  but `BusList` currently accepts neither in its props. `pnpm build` succeeds because Vite does not
  type check, so this is easy to miss.
- `StationDashboard` is fully implemented and `BusList` already accepts an `onOpenStationDashboard`
  prop, but neither is wired up in `App.tsx` — the `station_dashboard` screen is in the `Screen` union
  with no case in `renderScreen()`, and `App` does not pass the prop. Wiring it requires lifting the
  station-agent state (`INITIAL_STATION_AGENTS`, `onAddBus`, `onToggleCarExist`, `onSetRouteStatus`)
  out of the data module and into `App.tsx`.
- Payloads are simulated: payment always succeeds, QR scans resolve to random results, and "live"
  tracking is a static map illustration.

## Notes

- Design patterns come from a Figma Make scaffold. `AGENTS.md` and `CLAUDE.md` are generated alongside
  the project; keep them in sync when adding files or changing conventions.
- `.figma/` and `dist/` are platform/build output — `.figma` is watched-ignored by the dev server and
  `dist/` is gitignored.
#   T r a n s p o r t _ b o o k i n g  
 