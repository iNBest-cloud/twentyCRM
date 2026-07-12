# Proposal: add-customer-feedback-app

## Problem

iNBest is evaluating this Twenty fork as the base for a heavily customized CRM. Before committing to that strategy, we need a hands-on proof that the official Apps/SDK framework (`packages/twenty-sdk`, scaffolded via `create-twenty-app`, examples in `packages/twenty-apps/examples/hello-world` and `packages/twenty-apps/fixtures/rich-app`) can deliver a real, end-to-end feature — custom data model, external ingestion, and reactive backend logic — without touching `packages/twenty-server/src/engine` or any other core module.

Twenty ships standard CRM objects (Company, Person, Opportunity, Task, ...) but has no way to capture structured, NPS-style customer feedback, nor to ingest it from an external source (e.g. a web form).

## Solution

Build a workspace-installable app, `customer-feedback`, under `packages/twenty-apps/customer-feedback/`, following the `hello-world` example layout and using only the public `defineApplication` / `defineObject` / `defineView` / `defineNavigationMenuItem` / `defineLogicFunction` / `defineFrontComponent` / `defineRole` surface of `twenty-sdk/define`. The app will:

1. Define a custom `Feedback` object (rating, comment, source, contact email, received-at) with a default index view and a left-sidebar navigation item — giving it a full CRUD UI for free, as Twenty does for every custom object.
2. Ingest feedback from an external source via an HTTP-route-triggered logic function (`POST /feedback-ingest`) that validates the payload and creates a `Feedback` record through the generated `CoreApiClient`.
3. React to low ratings via a database-event-triggered logic function (`feedback.created`) that creates a follow-up `Task` record when `rating <= 6`.
4. (Stretch, Wave 3) Surface a `defineFrontComponent` widget summarizing the average feedback rating.

The app installs and runs against the local dev workspace via the `twenty-sdk` CLI (`yarn twenty plan|apply`, `yarn twenty dev:function:exec`), exactly like `hello-world`.

## Scope

- New standalone package `packages/twenty-apps/customer-feedback/` (own `package.json`, `tsconfig.json`, `.oxlintrc.json`, vitest config — mirrors `hello-world` scaffolding).
- One custom object (`Feedback`), one view, one navigation menu item, one default role.
- Two logic functions: HTTP-route ingestion, database-event low-rating follow-up.
- Stretch: one front component (average-rating widget).
- Local dev-workspace install/verification only.

## Out of scope

- Production deployment (`lambda` driver, `app:publish` to a registry).
- Email/Slack notifications on low ratings.
- AI agents/skills (`defineAgent`, `defineSkill`).
- Any edit to `packages/twenty-server/src/engine`, existing modules, or standard objects' core definitions.
- Upstream contribution back to `twentyhq/twenty`.
- Authenticating/hardening the ingestion webhook beyond payload validation (see `design.md` discovered limitations).

## Success criteria

- `packages/twenty-apps/customer-feedback` type-checks and builds via the SDK CLI (`yarn twenty dev:typecheck`, `yarn twenty dev:build`).
- `yarn twenty apply` against a local dev workspace creates the `Feedback` object, its view, and nav item with no errors; `Feedback` is visible and usable (table view, filters) in the running app.
- `yarn twenty dev:function:exec -n feedback-ingest -p '<valid JSON>'` creates a `Feedback` record; an invalid payload is rejected with a descriptive error.
- `yarn twenty dev:function:exec -n flag-low-rating-feedback -p '<payload with rating<=6>'` (or a real `feedback.created` event from the ingestion step) results in a follow-up `Task` record being created.
- No file under `packages/twenty-server/src/engine` or any other existing module is modified.
- Test suite for the app passes (`yarn test` inside the app package), within the repo's test-proportionality caps.
