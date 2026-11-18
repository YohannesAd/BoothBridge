# Security design

Status: required design controls, not implemented protections. Threats in scope include account/session theft, cross-user record access, malicious QR destinations, stored script injection, and leakage of contact details or token-bearing URLs.

## Authentication and sessions

- Use Spring Security OIDC login with a configured provider, authorization-code flow, PKCE where supported, and standard state/nonce validation. Provider selection and configuration are prerequisites for the first authenticated milestone. Request only necessary identity scopes; the MVP does not need mail or contact-provider access.
- Identify users by the exact OIDC issuer/subject pair. Do not treat an email address as an account key or automatically merge accounts by email. Provision the local account after validated sign-in; never store local passwords or provider tokens in event tables.
- Keep provider credentials and any tokens server-side. The browser receives only an opaque Spring-managed session cookie: `__Host-boothbridge_session`, `Secure`, `HttpOnly`, `SameSite=Lax`, `Path=/`, no `Domain`. Use HTTPS, including local HTTPS for cookie/camera verification. Do not store session or bearer tokens in localStorage, sessionStorage, URLs, or frontend configuration.
- Renew the session identifier after authentication and invalidate it on logout. Proposed idle timeout is 30 minutes with an absolute lifetime of 12 hours, enforced server-side; verify UX before launch. Start with one backend instance and in-memory sessions, so restart requires sign-in again.
- Authenticate protected API paths with JSON `401` failures rather than HTML login redirects. Permit only configured OIDC callbacks and validated same-origin return paths. Never accept a QR destination as an authentication return URL.

Keep CSRF protection enabled for session-authenticated unsafe methods, including logout. The frontend obtains the current token from `/api/v1/auth/csrf`, sends it using the returned header name, and refreshes after authentication changes. Do not confuse the readable CSRF token with the HttpOnly session credential. Follow [Spring Security's JavaScript CSRF guidance](https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html), including token rotation and deferred-token behavior. SameSite cookies complement CSRF validation; they do not replace it. The initial same-origin topology requires no credentialed cross-origin API access; any later CORS policy must name specific trusted origins.

## Authorization and data integrity

- Resolve the principal on the server. Every business operation loads or queries through an owned event; all child IDs, associations, and filters must be in that same scope. A record ID or a guessed URL grants no authority.
- Do not accept `ownerId`, security roles, timestamps, or persistence versions as body fields. Use explicit DTOs to prevent mass assignment. Reject unknown fields.
- Return the same `404` for missing and inaccessible resources; do not disclose another user's names, identifiers, or dependency counts. Use `403` for CSRF or operation-level denials, not to reveal object ownership.
- Apply database constraints as defense in depth, with service-level reference and ownership checks inside transaction boundaries. Conditional writes protect against stale edits; they are not a replacement for authorization.
- Keep event data private in caches and storage. Use `Cache-Control: no-store` for authenticated responses and avoid private-data prefetching into shared caches. The MVP has no public sharing, staff roles, or admin data access feature.

## QR and saved URL policy

QR contents are untrusted data. A successfully decoded or syntactically valid URL is not a trusted website.

1. Decode in the browser after camera consent; do not upload or persist images/frames. Camera APIs require a secure context and user permission; see [MDN camera access requirements](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia). Stop camera tracks when done and offer manual entry if permission or decoding fails.
2. Trim surrounding whitespace, then require an absolute `https://` URL of at most 2,048 characters. Reject embedded whitespace/control characters, backslashes, relative/protocol-relative URLs, user-info credentials, and every other scheme, including `javascript:`, `data:`, `file:`, and `http:`. Do not silently upgrade HTTP destinations.
3. Normalize the hostname to lowercase ASCII/punycode before comparison and display. Require valid DNS labels and at least two labels; reject empty labels, including a trailing dot. Reject literal IP addresses, `localhost`, and names equal to or beneath `.localhost`, `.local`, or `.internal`. Accept only the default HTTPS port or explicit port 443. Do not perform DNS lookups as part of saving.
4. Canonicalize only the scheme, hostname, and default port; preserve the path, query, and fragment, including their escapes. Those values may be essential to an event link and may also contain private tokens. Do not strip them, decode them into HTML, emit them to logs, or place them in analytics.
5. Validate on the client for feedback and again on the server before storage or edits. At open time, revalidate the saved URL against the current policy and show the hostname and complete destination for explicit confirmation. Render it as text/validated navigation data, never raw HTML.
6. Open only from an explicit user gesture after a successful save, or from a reviewed existing record. Use a separate browsing context with `noopener` and `noreferrer` protections and a no-referrer policy. A scan result and an API response never initiate navigation automatically.
7. The backend never follows, previews, unfurls, resolves short links, downloads favicons, checks availability, or redirects to saved destinations. Keep arbitrary destination URLs out of server HTTP clients and redirect parameters. This removes the MVP's URL-fetching entry point for SSRF; see [OWASP SSRF guidance](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html).

HTTPS and hostname rules cannot prove a site is safe. A public-looking hostname can resolve to a private address, and an external site or short link can redirect elsewhere. Without fetching it, BoothBridge cannot attest to the final destination. The review screen must explain this limitation without labeling links verified. Avoid a general-purpose redirect endpoint; [OWASP redirect guidance](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html) explains how trusted-looking redirects can aid phishing. Any future server preview requires a separate design for destination policy, DNS rebinding, redirect validation, egress controls, and resource limits.

## Input, secrets, and privacy

- Store notes and labels as plain text and rely on escaped rendering. Do not enable raw HTML or unrestricted Markdown. Use parameterized persistence operations; do not build SQL from submitted text or sort fields.
- Keep database credentials, OIDC client secrets, and future OpenAI keys in environment-specific secret storage. Local `.env`/local application files are ignored; commit only sanitized example files. Never put secrets in `NEXT_PUBLIC_*` variables, repository documentation, images, CI output, or client bundles.
- Use least-privilege runtime database credentials and separate migration privileges. Rotate any exposed credential and remove it from logs/artifacts; deleting an ignored file does not revoke a leaked secret or remove Git history.
- Protect runtime database connections with TLS outside trusted local development and protect stored data/backups with environment-managed encryption. Access, backup retention, and restore procedures must be defined before production.
- Treat recruiter details, notes, and full URLs as private user data. Collect only necessary fields, never seed real people into examples, and do not send data to AI providers during the MVP. Any later AI feature needs explicit data-flow and consent decisions.
- Log correlation IDs, outcomes, and timings, with sensitive request headers, bodies, query strings, and destination URLs redacted at the app, proxy, error tracker, and CI layers. Do not claim legal or regulatory compliance without a separate assessment.

## Controls required before public launch

Implement and verify request-size limits, security headers/CSP, OIDC/session controls, owner isolation, API throttling per session and source, and safe error envelopes. Limit camera permission to the application origin, disable framing, and avoid inline-script policies that weaken XSS protections. Rate thresholds and CSP details depend on the chosen host and measured usage and remain open.

Before storing real attendee data, decide the retention period, account deletion/export process, backup expiry, abuse response, and recovery ownership. The proposed API includes individual record deletion but no account-wide removal or export workflow. Hosting, provider settings, secret storage, and security checks are not configured by these documents.
