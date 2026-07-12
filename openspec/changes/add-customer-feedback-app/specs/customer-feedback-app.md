# Spec delta: customer-feedback-app

Capability: a workspace-installable Twenty app (`packages/twenty-apps/customer-feedback/`) that captures, ingests, and reacts to structured customer feedback, built entirely on the public `twenty-sdk/define` surface.

## ADDED Requirements

### Requirement: Feedback custom object with UI
The app SHALL define a custom `Feedback` object with fields `comment` (TEXT, label identifier), `rating` (NUMBER), `source` (SELECT: web/email/call/other), `contactEmail` (EMAILS), and `receivedAt` (DATE_TIME), paired with an index view (`ViewKey.INDEX`) and a navigation menu item, so the object is visible and usable in the workspace sidebar without any custom frontend code.

#### Scenario: App installed in a workspace
- **GIVEN** the `customer-feedback` app has been built and installed against a Twenty workspace via `yarn twenty apply` (or the publish/install CLI flow)
- **WHEN** a user opens the workspace sidebar
- **THEN** a "Feedback" navigation item is visible, and clicking it shows the default index view listing `Feedback` records with `comment`, `rating`, `source`, `contactEmail`, and `receivedAt` columns

### Requirement: External feedback ingestion via HTTP-route logic function
The app SHALL expose a logic function with an `httpRouteTriggerSettings` HTTP-route trigger (`POST /feedback-ingest`, `isAuthRequired: false`) that validates an incoming JSON payload (`rating` required numeric 0-10, `source` required one of `web|email|call|other`, `contactEmail` required valid email string, `comment` optional string, `receivedAt` optional ISO 8601 string) and creates a `Feedback` record via the generated `CoreApiClient` on success, rejecting invalid payloads with a descriptive error instead of creating a record.

#### Scenario: Valid external submission
- **GIVEN** the `feedback-ingest` logic function is deployed and installed
- **WHEN** it is invoked with `{"rating": 8, "comment": "Great support", "source": "web", "contactEmail": "user@example.com"}`
- **THEN** a new `Feedback` record is created with those field values and `receivedAt` defaulted to the invocation time

#### Scenario: Invalid external submission is rejected
- **GIVEN** the `feedback-ingest` logic function is deployed and installed
- **WHEN** it is invoked with a payload missing `rating` or containing a malformed `contactEmail`
- **THEN** no `Feedback` record is created and the function throws a descriptive validation error

### Requirement: Automatic low-rating follow-up via database-event logic function
The app SHALL expose a logic function with a `databaseEventTriggerSettings` trigger on `eventName: 'feedback.created'` that, when the created record's `rating` is `<= 6`, creates a follow-up `Task` record (via the standard `createTask` mutation) referencing the feedback's `contactEmail` and `comment`, and performs no action when `rating > 6`.

#### Scenario: Low rating creates a follow-up task
- **GIVEN** the `flag-low-rating-feedback` logic function is installed and subscribed to `feedback.created`
- **WHEN** a `Feedback` record is created with `rating: 4`
- **THEN** a `Task` record is created whose title references the feedback's `contactEmail` and `comment`

#### Scenario: High rating does not create a follow-up task
- **GIVEN** the `flag-low-rating-feedback` logic function is installed and subscribed to `feedback.created`
- **WHEN** a `Feedback` record is created with `rating: 9`
- **THEN** no `Task` record is created by this function

### Requirement: Additive-only installation
The app SHALL install and run against a local Twenty dev workspace using only the `twenty-sdk` CLI and the public `define*` API, without modifying any file under `packages/twenty-server/src/engine` or any other existing module.

#### Scenario: No core files touched
- **GIVEN** a git diff of the change is taken after implementation
- **WHEN** the diff is inspected
- **THEN** it contains only new files under `packages/twenty-apps/customer-feedback/` and `openspec/changes/add-customer-feedback-app/`, with zero changes under `packages/twenty-server/src/engine`

## Out of scope (see design.md for rationale)

- A `RELATION` field between `Feedback` and `Task`.
- Authentication/hardening of the ingestion endpoint beyond payload validation.
- The Wave 3 average-rating front component is a stretch requirement — its absence does not fail this spec.
