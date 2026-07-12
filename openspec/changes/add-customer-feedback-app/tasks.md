# Tasks: add-customer-feedback-app

10 tasks across 3 waves. Wave 3 is a stretch goal per the change request — Waves 1-2 alone satisfy the spec's core requirements.

## Wave 1 — Scaffold + object/view/nav + install verified

- [ ] 1.1 Scaffold the `customer-feedback` app package
  - **Change:** add-customer-feedback-app
  - **Spec:** `openspec/changes/add-customer-feedback-app/specs/customer-feedback-app.md`
  - **Spec wave slice:** n/a
  - **Stories:** CFA-01
  - **Owner profile:** backend
  - **Dependencies:** none
  - **Definition of done:** `packages/twenty-apps/customer-feedback/` exists with `package.json`, `tsconfig.json`, `tsconfig.spec.json`, `.oxlintrc.json`, `.gitignore`, `.nvmrc`, `.yarnrc.yml`, `vitest.config.ts`, `src/application-config.ts` (`defineApplication`, new valid UUID v4 `universalIdentifier`), `src/roles/default-role.ts` (`defineRole`) — all mirroring `packages/twenty-apps/examples/hello-world` scaffolding exactly.
  - **Verification gate:** cmd:cd packages/twenty-apps/customer-feedback && yarn twenty dev:typecheck

- [ ] 1.2 Define the `Feedback` custom object
  - **Change:** add-customer-feedback-app
  - **Spec:** `openspec/changes/add-customer-feedback-app/specs/customer-feedback-app.md`
  - **Spec wave slice:** n/a
  - **Stories:** CFA-01
  - **Owner profile:** backend
  - **Dependencies:** 1.1
  - **Definition of done:** `src/objects/feedback.object.ts` defines `Feedback` (`nameSingular: feedback`, `namePlural: feedbacks`) with fields `comment` (TEXT, label identifier), `rating` (NUMBER), `source` (SELECT: web/email/call/other with `{id,value,label,position,color}` options per the `rich-app` `PostCardStatus` pattern), `contactEmail` (EMAILS), `receivedAt` (DATE_TIME) — per design.md AD-2 data model table.
  - **Verification gate:** cmd:cd packages/twenty-apps/customer-feedback && yarn twenty dev:typecheck

- [ ] 1.3 Define the index view and navigation menu item for `Feedback`
  - **Change:** add-customer-feedback-app
  - **Spec:** `openspec/changes/add-customer-feedback-app/specs/customer-feedback-app.md`
  - **Spec wave slice:** n/a
  - **Stories:** CFA-01
  - **Owner profile:** backend
  - **Dependencies:** 1.2
  - **Definition of done:** `src/views/all-feedback.view.ts` (`defineView`, `ViewKey.INDEX`, columns for all 5 `Feedback` fields) and `src/navigation-menu-items/feedback.navigation-menu-item.ts` (`defineNavigationMenuItem`, `type: NavigationMenuItemType.VIEW`, pointing at the view) exist — avoids the two LLMS.md pitfalls (view without nav item, object without index view).
  - **Verification gate:** cmd:cd packages/twenty-apps/customer-feedback && yarn twenty dev:typecheck

- [ ] 1.4 Verify install against the local dev workspace
  - **Change:** add-customer-feedback-app
  - **Spec:** `openspec/changes/add-customer-feedback-app/specs/customer-feedback-app.md`
  - **Spec wave slice:** n/a
  - **Stories:** CFA-01
  - **Owner profile:** backend
  - **Dependencies:** 1.3
  - **Definition of done:** `src/__tests__/setup-test.ts` and `src/__tests__/app-install.integration-test.ts` exist (adapted from `hello-world`, asserting the app appears in `findManyApplications` after build+deploy+install); `yarn twenty plan` against the local dev server shows the expected object/view/nav-item creation with zero unexpected diffs, and `yarn twenty apply` succeeds.
  - **Verification gate:** cmd:cd packages/twenty-apps/customer-feedback && yarn test

## Wave 2 — Logic functions + tests

- [ ] 2.1 Implement the `feedback-ingest` HTTP-route logic function
  - **Change:** add-customer-feedback-app
  - **Spec:** `openspec/changes/add-customer-feedback-app/specs/customer-feedback-app.md`
  - **Spec wave slice:** n/a
  - **Stories:** CFA-02
  - **Owner profile:** backend
  - **Dependencies:** 1.2, 1.4
  - **Definition of done:** `src/logic-functions/feedback-ingest.function.ts` defines a `defineLogicFunction` with `httpRouteTriggerSettings: { path: '/feedback-ingest', httpMethod: 'POST', isAuthRequired: false }` per design.md AD-4; handler manually validates `rating` (numeric 0-10), `source` (one of web/email/call/other), `contactEmail` (email-shaped string), optional `comment`/`receivedAt`, throws a descriptive `Error` on invalid input, and otherwise calls `CoreApiClient.mutation({ createFeedback: {...} })` building the `EMAILS` composite as `{ primaryEmail: contactEmail, additionalEmails: null }`.
  - **Verification gate:** cmd:cd packages/twenty-apps/customer-feedback && yarn twenty dev:function:exec -n feedback-ingest -p '{"rating":8,"comment":"Great!","source":"web","contactEmail":"a@b.com"}'

- [ ] 2.2 Implement the `flag-low-rating-feedback` database-event logic function
  - **Change:** add-customer-feedback-app
  - **Spec:** `openspec/changes/add-customer-feedback-app/specs/customer-feedback-app.md`
  - **Spec wave slice:** n/a
  - **Stories:** CFA-03
  - **Owner profile:** backend
  - **Dependencies:** 1.2
  - **Definition of done:** `src/logic-functions/flag-low-rating-feedback.function.ts` defines a `defineLogicFunction` with `databaseEventTriggerSettings: { eventName: 'feedback.created' }` per design.md AD-3; handler inspects the event payload's `rating`, and when `<= 6` calls `CoreApiClient.mutation({ createTask: {...} })` with a title referencing `contactEmail` and `comment`; no-ops when `rating > 6`. Required `createTask` args confirmed against the generated client per design.md's `needs-verification` note before finalizing.
  - **Verification gate:** cmd:cd packages/twenty-apps/customer-feedback && yarn twenty dev:function:exec -n flag-low-rating-feedback -p '{"rating":4,"comment":"Not happy","contactEmail":"a@b.com"}'

- [ ] 2.3 Unit tests for `feedback-ingest` payload validation
  - **Change:** add-customer-feedback-app
  - **Spec:** `openspec/changes/add-customer-feedback-app/specs/customer-feedback-app.md`
  - **Spec wave slice:** n/a
  - **Stories:** CFA-02
  - **Owner profile:** backend
  - **Dependencies:** 2.1
  - **Definition of done:** `src/__tests__/feedback-ingest.test.ts` has exactly 3 tests per design.md's Testing strategy: (1) valid payload builds the expected `createFeedback` mutation args, (2) out-of-range/missing `rating` throws, (3) malformed `contactEmail` throws. No additional cases — respects the per-task cap.
  - **Verification gate:** cmd:cd packages/twenty-apps/customer-feedback && yarn test src/__tests__/feedback-ingest.test.ts

- [ ] 2.4 Manual end-to-end verification of the ingestion → follow-up flow
  - **Change:** add-customer-feedback-app
  - **Spec:** `openspec/changes/add-customer-feedback-app/specs/customer-feedback-app.md`
  - **Spec wave slice:** n/a
  - **Stories:** CFA-02, CFA-03
  - **Owner profile:** backend
  - **Dependencies:** 2.1, 2.2
  - **Definition of done:** confirms the two logic functions compose correctly against a real `feedback.created` event (not just direct `dev:function:exec` invocation), across the ✅ manual-test gate.
  - **Verification gate:** manual: 1) Run `yarn twenty apply` then `yarn twenty dev` against the local workspace 2) POST a low-rating payload (rating<=6) to the deployed `/feedback-ingest` route 3) Open the workspace UI, confirm the new Feedback record appears in the index view 4) Confirm a new Task record was created referencing the feedback's contactEmail/comment 5) Repeat with rating>6 and confirm no Task is created

## Wave 3 — Average-rating front component (stretch)

- [ ] 3.1 Implement the `feedback-average-rating` front component
  - **Change:** add-customer-feedback-app
  - **Spec:** `openspec/changes/add-customer-feedback-app/specs/customer-feedback-app.md`
  - **Spec wave slice:** n/a
  - **Stories:** CFA-04
  - **Owner profile:** frontend
  - **Dependencies:** 1.2
  - **Definition of done:** `src/front-components/feedback-average-rating.tsx` defines a `defineFrontComponent` (per `hello-world/front-components/hello-world.tsx` pattern) that queries `findManyFeedbacks` via `CoreApiClient`, computes the average `rating` client-side, and renders it in a fixed-size, non-scrolling widget (per LLMS.md pitfall on scroll-vs-fixed-size components). Documented as PoC-scale only (no pagination/aggregate-query optimization) per design.md Testing strategy note.
  - **mockup_ref:** n/a (design-pending — internal PoC widget, no Figma mockup; layout follows the plain inline-style convention used by `hello-world`'s own front component)
  - **Verification gate:** cmd:cd packages/twenty-apps/customer-feedback && yarn twenty dev:typecheck

- [ ] 3.2 Manual verification of the widget in a running workspace
  - **Change:** add-customer-feedback-app
  - **Spec:** `openspec/changes/add-customer-feedback-app/specs/customer-feedback-app.md`
  - **Spec wave slice:** n/a
  - **Stories:** CFA-04
  - **Owner profile:** frontend
  - **Dependencies:** 3.1
  - **Definition of done:** widget renders correctly against real `Feedback` data in the local dev workspace, across the ✅ manual-test gate.
  - **mockup_ref:** n/a (design-pending, see 3.1)
  - **Verification gate:** manual: 1) Run `yarn twenty apply` then `yarn twenty dev` 2) Create at least 2 Feedback records with different ratings via the index view 3) Add the widget to a page layout / dashboard per the CLI's dev instructions 4) Confirm the displayed average matches the manually computed average of the created records
