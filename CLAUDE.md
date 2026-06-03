# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Payload CMS v3 plugin that adds appointment scheduling to any Payload app. It registers collections, a global, custom endpoints, and admin UI components into the host app's config via a plugin factory function.

> **Status**: WIP and currently testing — not yet production-ready.

## Commands

```bash
# Development (runs the dev/ Next.js app with Turbo)
pnpm dev

# Build the plugin for publishing
pnpm build          # copyfiles → tsc (types) → swc (JS)
pnpm build:types    # TypeScript declarations only
pnpm build:swc      # JS output only
pnpm clean          # Remove dist and .tsbuildinfo

# Linting
pnpm lint
pnpm lint:fix

# Testing
pnpm test           # integration then e2e
pnpm test:int       # Vitest (unit/integration)
pnpm test:e2e       # Playwright

# Type generation (from the dev app)
pnpm generate:types      # Regenerates dev/payload-types.ts
pnpm generate:importmap  # Regenerates import map for admin UI

# Run a single Vitest test file
pnpm test:int -- path/to/test.ts
```

The dev app requires a Postgres connection. Set `DATABASE_URI` in `dev/.env`.

## Architecture

### Monorepo Layout

```
src/          ← plugin source (published to npm as dist/)
dev/          ← Next.js + Payload app used for development and testing
```

`pnpm-workspace.yaml` links them; the dev app imports the plugin as `appointments-plugin` (aliased to the local `src/`).

### Plugin Entry Point

`src/index.ts` exports `appointmentsPlugin(config)` — a curried function that takes plugin config and returns a Payload config transformer. This is the standard Payload plugin pattern: `(pluginConfig) => (payloadConfig) => payloadConfig`.

Config options:
- `disabled` — bail out entirely, return config unchanged
- `seedData` — auto-seed sample data on `onInit`
- `showDashboardCards` — inject `BeforeDashboard` component
- `showNavItems` — inject `BeforeNavLinks` component

The plugin registers:
- **4 collections**: `appointments`, `guestCustomers`, `teamMembers`, `services`
- **1 global**: `openingTimes`
- **1 custom endpoint**: `GET /api/get-available-appointment-slots`
- **3 admin components**: `BeforeDashboard`, `BeforeNavLinks`, custom view at `/appointments/schedule`

### Data Model

**Appointments** — core collection. Two `appointmentType` values drive conditional field visibility throughout the admin UI:
- `"appointment"`: requires host (TeamMember), customer or guestCustomer (mutually exclusive), and services
- `"blockout"`: requires host and a title (lunch, PTO, etc.)

The `end` field is never set manually — it is calculated by the `setEndDateTime` field hook from the sum of selected service durations. `adminTitle` is a hidden field calculated by `addAdminTitle` and used as the collection's `useAsTitle`.

**TeamMembers** — staff who can be assigned appointments. Only members with `takingAppointments: true` appear in the host selector (`filterOptions`).

**Services** — bookable services with `duration` (minutes) and optional `paidService`/`price`.

**GuestCustomers** — lightweight customer records for non-registered users (email, name, optional phone).

**OpeningTimes** (global) — per-day config (Monday–Sunday), each with `isOpen`, `opening`, and `closing` time fields. Used by the availability endpoint.

### Hook Lifecycle for Appointments

```
beforeValidate (collection level): validateCustomerOrGuest
  → ensures exactly one of customer or guestCustomer is set

beforeValidate (field level, on `end`): setEndDateTime
  → fetches services, sums durations, sets end = start + totalDuration

beforeValidate (field level, on `adminTitle`): addAdminTitle
  → generates display title from customer name or blockout title

afterChange (collection level): sendCustomerEmail
  → sends appointment created/updated email via payload.sendEmail()
```

### Availability Endpoint

`GET /api/get-available-appointment-slots?day=YYYY-MM-DD&services=id1,id2`

Logic in `src/endpoints/getAppointmentsForDayAndHost.ts`:
1. Fetches services by ID, sums their durations → `slotInterval`
2. Reads OpeningTimes global for the day of week
3. Generates all slots from opening to closing at `slotInterval` intervals
4. Filters out slots already booked
5. Returns `{ availableSlots, filteredSlots }`

### Admin Calendar View

`src/views/AppointmentsList/` is a server component that fetches appointments and team members, then renders `CalendarClient` (a `'use client'` component). The calendar uses `react-big-calendar` with team members as resources (columns in day/week view). SCSS overrides live alongside the component.

Component reference strings used in the plugin config (e.g., `'appointments-plugin/BeforeDashboard'`) map to the `exports` in `package.json`.

### Build Pipeline

During development, `package.json` exports point directly to `src/*.ts` files (no build needed to run `pnpm dev`). For publishing, `publishConfig.exports` redirects to `dist/`. The build:
1. `copyfiles` — copies CSS/SCSS/fonts/images from `src/` to `dist/`
2. `tsc` — emits `.d.ts` declaration files only (`emitDeclarationOnly`)
3. `swc` — transpiles TS/TSX to ESM JS (configured in `.swcrc`)

### Formatting and Linting

- **Prettier**: single quotes, trailing commas everywhere, 100-char line width (`.prettierrc.json`)
- **ESLint**: `@payloadcms/eslint-config` base, `eslint-config-next` for the dev app, TypeScript project service enabled

## Key Conventions

- **Date handling**: Use `moment.js` throughout. Store as ISO 8601 (`YYYY-MM-DDTHH:mm:ss.SSSZ`). Never use native `Date` for calculations.
- **Access control**: `anyone` (from `src/access/anyone.ts`) for public-facing operations (appointment creation, guest customer creation); `authenticated` for everything else.
- **Conditional fields**: Field visibility in the admin UI is controlled by `admin.condition` callbacks checking `siblingData.appointmentType`. Follow this pattern for any new appointment fields.
- **Collection slugs**: camelCase (`teamMembers`, `guestCustomers`, `openingTimes`).
- **No build step for dev**: The dev app imports directly from `src/` via workspace linking — running `pnpm dev` does not require `pnpm build` first.
