# Design: add-customer-feedback-app

## Grounding

Read before writing this design (per `forge:propose`):
- `packages/twenty-apps/examples/hello-world/src/**` (application-config, object, field, view, nav item, logic functions, front component, role, integration test)
- `packages/twenty-apps/fixtures/rich-app/src/**` (SELECT-with-options object, cross-object `RELATION` field, `databaseEventTriggerSettings` usage, standard-object field extension)
- `packages/twenty-sdk/src/sdk/define/index.ts` (actual exported `define*` surface)
- `packages/twenty-shared/src/application/logicFunctionManifestType.ts`, `.../types/FieldMetadataType.ts`, `.../types/composite-types/emails.composite-type.ts`
- `packages/twenty-sdk/src/cli/commands/{dev,app}/*.ts` (actual CLI verbs)

Every API shape below is taken from one of those files — no invented API.

## Architecture decisions

### AD-1: Standalone app package, additive only
`packages/twenty-apps/customer-feedback/` is a self-contained package (own `package.json`, `tsconfig.json`, `.oxlintrc.json`, `vitest.config.ts`) scaffolded identically to `hello-world`. Nothing under `packages/twenty-server/src/engine` or any existing module is touched — the app talks to the running server only through the generated `CoreApiClient` / `MetadataApiClient` (`twenty-client-sdk`) and the CLI's dev-sync (`plan`/`apply`), never by editing engine code.

### AD-2: Field type choices (discovered constraints)
Available field types: `packages/twenty-sdk/src/sdk/define/fields/field-type.ts` re-exports `FieldMetadataType` (`ACTOR, ADDRESS, ARRAY, BOOLEAN, CURRENCY, DATE, DATE_TIME, EMAILS, FILES, FULL_NAME, LINKS, MORPH_RELATION, MULTI_SELECT, NUMBER, NUMERIC, PHONES, POSITION, RATING, RAW_JSON, RELATION, RICH_TEXT, SELECT, TEXT, TS_VECTOR, UUID`).

| Field | Type chosen | Alternative considered | Why chosen |
|---|---|---|---|
| `comment` | `TEXT` | `RICH_TEXT` (composite, `RichTextMetadata`) | Simpler payload shape for the HTTP ingestion handler; no markdown-body composite to assemble. Also serves as the object's `labelIdentifierFieldMetadataUniversalIdentifier` (mirrors `hello-world`'s `example-object.ts` using its `name` TEXT field as label). |
| `rating` | `NUMBER` | `SELECT` (0-10, 11 options, `RATING` built-in 1-5 stars) | `SELECT` would need 11 hand-authored options (see `rich-app/post-card.object.ts` pattern); `RATING` is a fixed 1-5 star widget, doesn't fit NPS 0-10. Plain `NUMBER` keeps the ingestion payload a bare number and the low-rating check a simple `<= 6` comparison. |
| `source` | `SELECT` | `TEXT` (freeform) | Bounded enum (`web/email/call/other`) — `SELECT` gives filterable, closed values, following the `PostCardStatus` options-array pattern in `rich-app/post-card.object.ts` (`{ id, value, label, position, color }[]`). |
| `contactEmail` | `EMAILS` | `TEXT` | `EMAILS` is Twenty's standard composite email type (`emails.composite-type.ts`: `{ primaryEmail: string; additionalEmails: string[] | null }`). Using it (not raw `TEXT`) is consistent with how `Person`/`Company` model email and unlocks the built-in email UI (mailto links, uniqueness on `primaryEmail`). |
| `receivedAt` | `DATE_TIME` | — | Direct fit; matches `hello-world`/`rich-app` usage (`deliveredAt` on `PostCard`). |

**Ingestion payload → field mapping**: the HTTP handler receives `{ rating, comment?, source, contactEmail, receivedAt? }` (JSON) and must build the `EMAILS` composite as `{ primaryEmail: contactEmail, additionalEmails: null }` when calling `createFeedback`, and default `receivedAt` to `new Date().toISOString()` when absent.

### AD-3: Low-rating reaction creates a standard-object `Task`, not a custom relation field
`defineLogicFunction`'s `databaseEventTriggerSettings: { eventName: '<objectNameSingular>.created' }` (confirmed via `rich-app/on-post-card-created.function.ts`: `eventName: 'postCard.created'`) fires `flag-low-rating-feedback` on every `Feedback` creation. When `rating <= 6`, the handler calls `client.mutation({ createTask: { __args: { data: { title: ... } }, id: true } })` — the exact pattern already proven in `hello-world/create-hello-world-company.ts` for `createCompany`, swapped to the standard `Task` object.

**Discovered limitation**: we deliberately do **not** wire a `RELATION` field between `Feedback` and `Task` for this PoC. `rich-app` shows the real mechanism (`post-card-recipients-on-post-card.field.ts`: paired `defineField({ type: FieldType.RELATION, relationTargetObjectMetadataUniversalIdentifier, relationTargetFieldMetadataUniversalIdentifier, universalSettings: { relationType } })` on both sides), and it is provably reachable — but adding it here would require a second field-pair on the standard `Task` object (extending a standard object, as `rich-app/company-can-receive-postcards.field.ts` does for `Company`), doubling the object-model surface for a PoC. The follow-up `Task`'s `title` instead embeds the feedback's `contactEmail` and `comment` as free text so a human can trace it back manually. Noted as a scoped-out enhancement, not a framework limitation.

**needs-verification**: the exact required arguments of the generated `createTask` mutation (title only, or additional required fields) are only knowable once `yarn twenty dev:generate-client` produces the typed `CoreApiClient` against the target workspace. Backend implementer must confirm this at Wave 2 implementation time (context7/generated-client check), not assumed here.

### AD-4: HTTP ingestion is unauthenticated by design, validated manually
`httpRouteTriggerSettings: { path: '/feedback-ingest', httpMethod: 'POST', isAuthRequired: false }` (shape confirmed in `hello-world/hello-world.ts` and `rich-app/test-function.function.ts`) is required because the payload originates from an external, un-authenticated web form — the same trust model as a public webhook. `InputJsonSchema` (`twenty-shared/logic-function`) only wires into `toolTriggerSettings` (agent tool calls), not `httpRouteTriggerSettings` — confirmed by grep across `twenty-shared/src`, no schema hook exists for HTTP routes. Payload validation is therefore hand-written in the handler (type/range/regex checks), with **no new runtime dependency** added (the scaffolded app's `dependencies` are empty by convention; adding e.g. `zod` is avoidable for four fields and kept out of scope).

**Discovered limitation / follow-up**: an unauthenticated write endpoint is a real production concern. For this PoC it is accepted (mirrors `hello-world`'s own unauthenticated GET example) and flagged here rather than solved — production hardening would use `forwardedRequestHeaders` (seen in `rich-app/test-function.function.ts`) to check a shared-secret header, or flip to `isAuthRequired: true` with a dedicated API key issued to the external form service.

### AD-5: Object needs an index view + nav item (LLMS.md pitfall)
`packages/twenty-apps/examples/hello-world/LLMS.md` explicitly calls out two common pitfalls: an object without an index view is invisible, and a view without a `navigationMenuItem` never appears in the sidebar. Wave 1 always pairs `defineObject` → `defineView(ViewKey.INDEX)` → `defineNavigationMenuItem(type: NavigationMenuItemType.VIEW)` in the same wave, per the `example-object.ts` → `example-view.ts` → `example-navigation-menu-item.ts` chain.

### AD-6: Test runner is vitest, not jest (SDK-app-local override)
Root `CLAUDE.md`/`AGENTS.md` jest commands apply to `twenty-front`/`twenty-server`. Apps under `packages/twenty-apps/` are scaffolded by `create-twenty-app` with their own `vitest.config.ts` and `"test": "vitest run"` script (confirmed in `hello-world/package.json` and `vitest.config.ts`), matching upstream Twenty SDK-app convention. Tasks in this change use `yarn test` / `vitest run` inside the app package, not `npx jest`.

## Component diagram

```
packages/twenty-apps/customer-feedback/
├── package.json, tsconfig.json, .oxlintrc.json, vitest.config.ts   (mirrors hello-world scaffolding)
├── src/
│   ├── application-config.ts        defineApplication (Wave 1)
│   ├── roles/
│   │   └── default-role.ts          defineRole (Wave 1)
│   ├── objects/
│   │   └── feedback.object.ts       defineObject: Feedback (rating, comment, source, contactEmail, receivedAt) (Wave 1)
│   ├── views/
│   │   └── all-feedback.view.ts     defineView (ViewKey.INDEX) (Wave 1)
│   ├── navigation-menu-items/
│   │   └── feedback.navigation-menu-item.ts   defineNavigationMenuItem (Wave 1)
│   ├── logic-functions/
│   │   ├── feedback-ingest.function.ts        HTTP-route trigger, POST /feedback-ingest (Wave 2)
│   │   └── flag-low-rating-feedback.function.ts  database-event trigger, feedback.created (Wave 2)
│   ├── front-components/
│   │   └── feedback-average-rating.tsx        defineFrontComponent (Wave 3, stretch)
│   └── __tests__/
│       ├── setup-test.ts
│       ├── feedback-ingest.test.ts            payload validation unit tests (Wave 2)
│       └── app-install.integration-test.ts    build+deploy+install smoke test, adapted from hello-world (Wave 1)
```

## Data model

**`Feedback` custom object** (`nameSingular: feedback`, `namePlural: feedbacks`):

| Field | Type | Notes |
|---|---|---|
| `comment` | `TEXT` | Label identifier field (`labelIdentifierFieldMetadataUniversalIdentifier`) |
| `rating` | `NUMBER` | Expected range 0-10, enforced only in the ingestion handler (no DB-level constraint available via SDK field config) |
| `source` | `SELECT` | Options: `web`, `email`, `call`, `other` (`{ id, value, label, position, color }[]`, per `rich-app` `PostCardStatus` pattern) |
| `contactEmail` | `EMAILS` | `{ primaryEmail, additionalEmails }` composite |
| `receivedAt` | `DATE_TIME` | Defaults to ingestion time when omitted from payload |

No `RELATION` fields (see AD-3). No fields added to any standard object.

## Testing strategy

Per `.claude/rules/agent-common.md` → "Test proportionality" and this change's multi-file classification:

- **Per-task cap**: 2-4 tests (happy + error + edge), hard ceiling 6/task.
- **Per-file cap**: hard ceiling 8/file.
- **Per-change cap**: hard ceiling 30 tests cumulative; this change targets **5-6 tests total**, well under cap:
  - `feedback-ingest.test.ts` (Wave 2, task 2.3): 3 tests — happy path (valid payload → record created), missing/invalid `rating` (out of 0-10 range), invalid `contactEmail` format.
  - `app-install.integration-test.ts` (Wave 1, task 1.4, adapted from `hello-world`): 1 test — app appears in `findManyApplications` after build+deploy+install (proves additive install works end-to-end).
  - `flag-low-rating-feedback` behavior is verified via **manual** `dev:function:exec` gate (task 2.4), not a unit test — the handler's only logic is a comparison + one mutation call already covered by the SDK's own `create-hello-world-company` pattern; a redundant unit test here would add token cost without new signal (see LLM-specific over-testing rationale in `agent-common.md`).
- No snapshot tests, no tests for the front component (Wave 3 is verified manually per its `manual:` gate — a rendering smoke check, not unit-testable without mocking the whole `CoreApiClient`).

## SDK dev-loop commands (verified against `packages/twenty-sdk/src/cli/commands/**`)

```bash
# One-time: authenticate the CLI against the local dev workspace
yarn twenty remote:add --api-url http://localhost:3000 --as local

# Type-check / build the app package (no server round-trip)
yarn twenty dev:typecheck packages/twenty-apps/customer-feedback
yarn twenty dev:build packages/twenty-apps/customer-feedback

# Preview then apply metadata changes to the running local workspace
# (equivalent of the deprecated `dev --once --dry-run` / `dev --once`)
yarn twenty plan packages/twenty-apps/customer-feedback
yarn twenty apply packages/twenty-apps/customer-feedback

# Iterative dev loop (watch + build + sync + typed-client generation)
yarn twenty dev packages/twenty-apps/customer-feedback

# Invoke a logic function directly with a JSON payload
yarn twenty dev:function:exec packages/twenty-apps/customer-feedback \
  -n feedback-ingest -p '{"rating":8,"comment":"Great!","source":"web","contactEmail":"a@b.com"}'

# Full package-based install/uninstall lifecycle (used by the integration test)
# appBuild → appDeploy → appInstall → appUninstall, via twenty-sdk/cli programmatic API
```

## Domain-map gap (flagged, not fixed here)

`AGENTS.md` → Domain Map lists `frontend_paths`/`backend_paths` for `packages/twenty-front`, `packages/twenty-server`, `packages/twenty-shared`, `packages/twenty-emails` — it does not list `packages/twenty-apps/**`. Owner-profile assignment below (`backend` for SDK config/logic-functions, `frontend` for the Wave 3 React component) is based on the nature of the code, not an existing domain-map entry. Recommend an `agent-prep` re-run to add `packages/twenty-apps` to the Domain Map; out of scope for this change.
