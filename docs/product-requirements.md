# Product requirements

Status: proposed MVP behavior; no features are implemented. This document defines scope and acceptance. [API contracts](api-contracts.md) and [database design](database-design.md) define the detailed fields and status values.

## Problem and users

Conference attendees collect application URLs, booth check-in pages, recruiter details, and conversation notes across disconnected tools. BoothBridge keeps that information organized by event so the attendee can act during the event and follow up afterward.

The primary user is a job-seeking attendee. A secondary user is an attendee focused on professional networking. Recruiters and networking contacts are saved records, not application accounts or recipients of automated messages. MVP records are private to one authenticated owner; there are no shared events or administrative workflows.

## User stories and acceptance criteria

| ID | As an attendee, I want to… | Acceptance criteria |
| --- | --- | --- |
| MVP-01 | Sign in and keep my information private. | A supported identity provider creates or resumes my account. Signed-out API access fails; another user's records cannot be listed, read, changed, or deleted. Signing out invalidates my session. |
| MVP-02 | Organize companies within an event. | I can create, view, edit, and list events with a name, date range, and time zone. I can add and edit multiple company entries with an optional booth location. A company at a different event is a separate entry. Invalid dates are rejected. |
| MVP-03 | Scan or enter a link and save it for later. | Camera capture is permission-based. A decoded URL appears in a review screen with its host and full destination. I choose an event, company, label, and category before saving. A manual entry path works if the camera is unavailable or denied. Invalid URLs cannot be saved or opened through the app. |
| MVP-04 | Open a saved destination deliberately. | After a successful save, a separate tap opens the reviewed link. Save failure shows a retryable error and retains the draft in the current screen; it never reports success or auto-opens. Opening an existing saved link uses the same review and confirmation. |
| MVP-05 | Keep several useful links for one company. | A company can have distinct application, check-in, careers, and other labeled links. A link can optionally reference an opportunity at that same company. Editing a link requires revalidation. Duplicate URLs are allowed; labels and opportunity associations can differ. |
| MVP-06 | Track booth visits separately from applications. | A company starts `NOT_CHECKED_IN`; I can explicitly mark it `CHECKED_IN` or undo that choice. Each opportunity starts `NOT_APPLIED` and has its own application status. Changing either status never changes the other. |
| MVP-07 | Save recruiter and networking contacts. | I can add, view, edit, list, and delete multiple contacts per company. A contact requires a name and kind; email, phone, and role are optional. Event-level contacts without a company are supported. Contacts need no BoothBridge account. |
| MVP-08 | Track several opportunities at one company. | I can create, edit, list, and delete opportunities with individual titles, statuses, and optional application dates. Statuses can be corrected directly; no fixed interview pipeline is enforced. `NOT_APPLIED` clears any application date. External pages never update status automatically. |
| MVP-09 | Capture notes and follow-up actions. | I can create, edit, list, and delete plain-text notes and tasks at event or company scope. Tasks support an optional due time and completion/reopening. Reloading shows persisted changes, and open tasks can be filtered. The UI displays due times in the event time zone. |
| MVP-10 | Correct mistakes without losing related work. | I can edit and delete individual links and other leaf records. Event/company deletion is allowed only when no dependent records remain. An opportunity referenced by a link cannot be deleted until the link is unlinked or deleted. Conflicts and stale edits are explained instead of silently overwriting data. |

## QR scan → save URL → open destination

1. Start from an owned event and select or create a company. Request camera access only after choosing Scan; use a secure browser context. Offer manual paste throughout.
2. Decode on the device. Pause scanning after one result to avoid repeated frames. Release camera tracks when leaving the scanner; do not upload camera frames or QR images.
3. Treat the payload as untrusted input. Accept only an absolute HTTPS URL under the [URL policy](security.md#qr-and-saved-url-policy). Show a clear error for text, unsupported schemes, or invalid hosts. Do not navigate from a scan result.
4. Show the ASCII hostname prominently and let the attendee inspect the complete URL, including query and fragment. A short link may redirect elsewhere; explain that the final destination is unverified. Allow editing, then revalidate.
5. Require a label and category (`APPLICATION`, `CHECK_IN`, `CAREERS`, or `OTHER`). Source is `QR` or `MANUAL`. Offer an optional opportunity association at the selected company.
6. Save through the authenticated API. Disable repeat submission while pending. On failure, keep the draft in memory and explain the failure. If the save outcome is uncertain, refresh the saved-link list before retrying. No offline queue or persistence guarantee is implied.
7. After success, show the saved record and an explicit **Open destination** action. Open from that user gesture in a separate browsing context with opener isolation and no referrer. No server redirect or fetch is involved.
8. Return to BoothBridge to record check-in, application progress, notes, or a task explicitly. Opening a destination does not verify completion on the external site.

## Status boundaries

- Booth check-in is an attendee-reported state on the event's company entry. Its timestamp records when BoothBridge was marked, not proof of an external check-in.
- An opportunity is the attendee's private application tracking record. Each opportunity has one current application state; a separate application table and status history are deferred.
- A link's category describes its purpose. A check-in link may be opened without checking in, and an application link may be opened without applying.
- Multiple applications to distinct postings at the same company use separate opportunities. Duplicate titles are allowed.

## Quality and scope boundaries

- Design for touch and narrow screens; accept a 360 CSS-pixel viewport without horizontal page overflow. Controls need labels, visible focus, keyboard access, readable errors, and accessible save feedback.
- Support current iOS Safari and Android Chrome as the initial mobile targets; confirm camera behavior on real devices. Manual entry remains available on unsupported browsers.
- Network access is required for persistence. Never display an unsaved change as saved. On session expiry, keep the draft visible in the active screen and offer sign-in in a separate browsing context so the attendee can return and retry. Reloading or leaving that screen may discard the draft; do not store it in browser persistence.
- All event data is owner-scoped. Structured queries and pagination are required; public profiles and cross-user company catalogs are excluded.
- No AI, resume uploads, automated applications, email sending, scheduled notifications, data enrichment, public QR sharing, or offline synchronization in the core MVP. Tasks are an in-app list, not a reminder delivery service.
- Exact performance budgets and data-volume targets require measurements and remain open; there are no claimed benchmarks or availability guarantees.

## First implementation milestone

**Authenticated event/company/link vertical slice**, using manual URL entry first:

- Select an OIDC provider, build tool, runtime and patch versions; introduce the smallest frontend/backend structure and Flyway baseline.
- Sign in, create an owned event and company, save a labeled HTTPS link, reload, review it, and explicitly open it.
- Reject unsafe URLs, invalid dates, unauthenticated access, cross-user IDs, and stale updates with the documented errors.
- Verify persistence and constraints against MariaDB using Testcontainers; verify the main journey in Playwright with synthetic identities and data.
- Add relevant checks to GitHub Actions and document runnable commands only after they work.

Completion requires a demonstrable journey and recorded check results. Camera scanning, contacts, opportunities, notes, and tasks follow as subsequent slices; this milestone does not complete the full MVP.

## Open product questions

- Which OIDC provider and account onboarding experience should launch first?
- Should a later release permit HTTP-only event links with a separate warning? The MVP accepts HTTPS only.
- Are event-level notes/tasks sufficient, or do users need direct contact/opportunity associations?
- What retention, account export, and account deletion experience is required before a public launch?
- Does real conference usage justify offline capture, reminders, or a shared company catalog after the MVP?
