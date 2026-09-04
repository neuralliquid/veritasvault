---
document_type: specification
classification: internal
status: review
version: 0.2.0
last_updated: "2026-09-04"
applies_to:
  - Web
  - Identity
  - Alpha
dependencies:
  - ../prds/veritasvault-alpha.md
reviewers:
  - "@tech-lead"
  - "@security"
priority: p1
next_review: "2026-12-04"
---

# VeritasVault Alpha Technical Specification

## Status and scope

This is the proposed implementation contract for the
[VeritasVault Alpha PRD](../prds/veritasvault-alpha.md). It is planning evidence, not execution
authority. The canonical origin, product experience, system of record, and OIDC client type were
selected on Baton epic `bcfc1e75`; this draft incorporates those choices for review.

## Design principles

- Prefer the smallest coherent alpha over completing the existing prototype surface.
- One route has one authentication meaning; duplicate and demo paths are removed or fail closed.
- The relying party owns its callback and uses a maintained PKCE-capable OIDC library.
- Identity, application authorization, deployment, and authentic acceptance are separate gates.
- Exact redirect values are observed from merged application code and the selected canonical host.
- A clean plan or apply is infrastructure evidence, not proof that a user can sign in.

## System context

| Component                         | Alpha responsibility                                                           | Current state                                                                                            |
| --------------------------------- | ------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------- |
| `neuralliquid/veritasvault-web`   | Next.js UI, BFF auth callback/session, protected web/API routes, alpha journey | Deployed prototype with overlapping auth                                                                 |
| `neuralliquid/veritasvault`       | Post-alpha domain/API service unless separately approved                       | Not integrated with web; no registered authentication scheme                                             |
| Mystira Identity                  | OIDC issuer and adult-user authentication                                      | Cross-repo, gated change                                                                                 |
| Supabase                          | Alpha data system of record; Auth is legacy                                    | Used throughout web; auth must not remain a competing authority                                          |
| Cloudflare + Azure Container Apps | Edge and application origin                                                    | Cloudflare serves a stale artifact with legacy Vercel headers; intended Azure origin is not yet verified |
| Baton                             | Decisions, documentation, execution graph, evidence, approval gates            | Epic `bcfc1e75`                                                                                          |

## Recommended target architecture

For the shortest safe route to alpha:

1. Keep the two repositories separate.
2. Keep Cloudflare at the edge and use `www.veritasvault.net` as the canonical alpha origin. Create
   or identify, deploy, and verify an Azure Container Apps origin before alpha. Vercel is not a
   permitted host, fallback, or rollback target.
3. Use `veritasvault-web` as a browser-facing BFF: the OIDC transaction, callback, token validation,
   and session are handled server-side.
4. Use Supabase as the alpha data system of record. Retire its Auth feature as a user identity
   authority; data rows map to immutable Mystira `(iss, sub)` through an explicit application-user
   record.
5. Leave the .NET API out of the alpha request path. Bringing it into scope requires a separate
   approved decision and an independently authenticated deployment plan.

## Identity architecture

### Library requirement

Use a maintained OIDC implementation capable of authorization code with S256 PKCE. The current
NextAuth/Auth.js route is a candidate because it already owns a Next.js callback, but the chosen
version and Mystira configuration must prove all requirements below in a non-production
environment. Do not hand-roll discovery, code exchange, JWT validation, or key rotation.

### Proposed routes

The exact library may own route internals, but the application contract is:

| Purpose       | Candidate route              | Access                                              |
| ------------- | ---------------------------- | --------------------------------------------------- |
| Start sign-in | `/api/auth/signin/mystira`   | Anonymous                                           |
| Callback      | `/api/auth/callback/mystira` | Anonymous, transaction-bound                        |
| Session view  | `/api/auth/session`          | Minimal principal or signed-out state; never cached |
| Logout        | `/api/auth/signout`          | Session-bound; local logout required                |

The selected production callback is
`https://www.veritasvault.net/api/auth/callback/mystira`. It must not be registered until the route
is confirmed in merged code.

### Authorization request

- Discovery issuer: `https://identity.mystira.app/` in production.
- Every discovered authorization, token, user-info, logout, and JWKS endpoint must use HTTPS and
  match the issuer origin unless its exact origin is explicitly allowlisted in reviewed
  configuration. Reject discovery documents containing any other endpoint.
- `response_type=code`.
- Scopes initially `openid profile email`; no `offline_access` without a concrete background-use
  case and refresh-token custody design.
- Unique high-entropy `state` and `nonce` per transaction.
- High-entropy PKCE verifier retained server-side; `code_challenge_method=S256`.
- Bind state, nonce, verifier, and return destination to a short-lived, host-only pre-auth browser
  cookie or server session. Rotate into a new authenticated session after a successful callback.
- `redirect_uri` built from trusted configuration, never request headers.
- Return destination restricted to a same-origin relative allowlist.

### Callback and token validation

- Consume each transaction once and reject missing, expired, replayed, or state-mismatched flows.
- Redeem the code with the original verifier and exact redirect URI.
- Reject redirects from the token endpoint so the authorization code, verifier, and client secret
  cannot cross hosts or downgrade to HTTP.
- Validate signature against discovery JWKS, exact issuer, intended audience/client, expiry/not-before,
  nonce, and required subject.
- Use a confidential client with S256 PKCE. Store the client secret in Mystira's production Key
  Vault, inject it into the server-side BFF through the approved hosting secret mechanism, and prove
  the complete contract in a non-production flow before registration or activation. Never expose
  the secret to the browser or commit it to either repository.
- Key the application user by immutable issuer plus `sub`; email is display/contact data, not the
  authorization key.

### Session

- Browser receives only an opaque or encrypted session cookie; no raw OIDC token in local storage.
- Session cookies contain either an opaque server-side identifier or an authenticated-encryption
  (AEAD) payload. Reject every modified or unauthenticated cookie.
- Cookie is `Secure`, `HttpOnly`, host-only, `Path=/`, and `SameSite=Lax` unless the chosen library
  demonstrates a stricter compatible setting.
- `/api/auth/session` returns `Cache-Control: no-store` for both signed-in and signed-out responses.
- Session lifetime never exceeds the verified identity assertion or configured alpha maximum.
- Logout clears the browser cookie and revokes the server-side session or records an equivalent
  denylist entry. Cohort removal and incident response use the same mandatory revocation path.
- Logs may include a one-way safe correlation identifier, never the cookie or token.

### Cohort authorization

- Authentication by Mystira does not automatically authorize alpha access.
- Maintain an operator-controlled allowlist of immutable Mystira `(iss, sub)` pairs or
  application-user rows.
- Unknown or disabled subjects receive 403 and no product data.
- Protect invite, disable, and cohort-administration routes with a separate operator role or
  operator allowlist. Normal cohort users receive 403, and every operator action is audited.
- Alpha is Adult-only; the Mystira RP registration must enforce the same age class.

## Existing-auth retirement matrix

| Existing surface                               | Required action                                                        |
| ---------------------------------------------- | ---------------------------------------------------------------------- |
| NextAuth GitHub/Google provider configuration  | Replace with the single verified Mystira provider                      |
| `/api/auth/login` demo credentials             | Delete or return a fail-closed retired response before alpha           |
| `/api/auth/github` and mock callback           | Delete                                                                 |
| Supabase password/social login UI and callback | Remove from alpha navigation and runtime auth                          |
| Corporate static login form                    | Remove or redirect to the clearly labelled Standard alpha landing page |
| `localStorage.auth_token`                      | Remove and migrate all consumers to the server session                 |
| Middleware with no auth enforcement            | Add explicit public-route allowlist and fail-closed protection         |

CI must contain a negative test proving each retired endpoint cannot create an authenticated state.

## Web authorization

- Public allowlist: landing, static assets, health/readiness, OIDC start/callback, and legally required
  public pages.
- Machine-authenticated allowlist: only explicitly inventoried service routes. If retained,
  `/api/cron/sync` requires its dedicated, rotated `CRON_SECRET` or an approved workload identity;
  a browser session alone never authorizes it. Failed or missing machine credentials return 401 and
  cannot expose application data.
- Everything outside the public and machine-authenticated allowlists requires a cohort session by
  default.
- Server-rendered pages redirect signed-out users to Mystira start through a stable sign-in page.
- API routes return 401 for signed-out users and 403 for authenticated users outside the cohort.
- Object access checks use the application user/subject on every read and write; route protection
  alone is insufficient.
- Every state-changing cookie-authenticated API route requires a CSRF token or exact trusted
  `Origin` validation. Mutations never use `GET`, and `SameSite=Lax` is not the sole CSRF control.
- Remove any server-side use of service-role database credentials from routes that do not first
  establish and enforce the caller's authorization.

## Post-alpha .NET API integration

If a later approved decision brings the .NET API into scope:

- Register JWT bearer authentication against Mystira discovery and the API audience.
- Call `UseAuthentication` before `UseAuthorization`.
- Replace `AllowAnyOrigin` with the exact canonical web origin.
- Apply authorization policies to every non-health endpoint.
- Use delegated access tokens from the BFF only where required; do not forward ID tokens.
- Add contract and integration tests for issuer, audience, expiry, missing scope, wrong subject, and
  CORS denial.

If no selected journey requires it, record the API as post-alpha rather than adding an unneeded
distributed-system dependency.

## Mystira cross-repo contract

The Mystira work is a separate task and repository. Its required sequence is:

1. Create and approve an ADR-0029 addendum admitting the neuralliquid VeritasVault RP.
2. Define a stable client identifier, exact redirect URI, scopes, Adult-only class, logout behavior,
   the proven confidential-client-plus-S256-PKCE contract, and Key Vault secret delivery.
3. Add Terraform configuration with the client disabled (Phase A).
4. Plan and review explicit resource actions; do not infer safety from a summary count.
5. Apply only through the protected production environment and explicit owner approval.
6. Enable the application configuration and RP in an order with a tested rollback.
7. Remove obsolete redirect URIs and provider registrations only after authentic acceptance.

No VeritasVault task may edit `mystira-workspace` as an incidental change.

## Data boundary

Before implementation, inventory each alpha page/API and classify it as public, cohort-readable,
user-owned, operator-only, or excluded. For Supabase-backed data:

- Confirm row-level security for browser-accessible tables.
- Prefer server-side access through the BFF where authorization requires application context.
- Never expose the service-role key to the browser.
- Store the Mystira `(iss, sub)` mapping separately from mutable profile/email fields.
- Define deletion, cohort-removal, and audit behavior before inviting users.

## Observability

Emit structured, secret-safe events for:

- sign-in started/completed/failed by reason class;
- callback transaction rejection;
- session created/expired/revoked;
- 401 and 403 counts by route class;
- core journey started/completed/failed;
- deployment revision and application version;
- upstream Mystira, database, and optional .NET API dependency health.

Dashboards and alerts must distinguish product errors from deliberate negative auth tests. No event
contains password, code, verifier, token, secret, raw cookie, or full authorization URL.

## Test strategy

### Unit and contract tests

- Return-path and redirect-origin validation.
- Cohort authorization and `(iss, sub)` mapping.
- Public/protected route classification.
- Session expiry and revocation.
- Cookie integrity, logout revocation, and denial when the pre-logout cookie is replayed.
- Operator-route denial for an ordinary cohort user and audit capture for an operator action.
- CSRF-token or exact-Origin rejection on every state-changing route.
- Machine-route rejection for missing, wrong, expired, or browser-session-only credentials.
- Data ownership checks.

### OIDC integration tests

- Successful authorization code plus S256 PKCE.
- Missing/wrong verifier, state, nonce, issuer, audience, signature, and expired token.
- Callback replay.
- A discovery document with an untrusted endpoint and a redirecting token endpoint.
- Pre-auth browser-binding mismatch and authenticated-session rotation after callback.
- Disabled RP and non-invited user.
- An otherwise invited child or teen identity rejected before application data is exposed, proving
  the Adult-only RP and application policy is not masked by cohort denial.
- Key rotation through discovery/JWKS refresh.

### Browser journeys

- Signed-out visit to protected page.
- Invited-user sign-in and intended return path.
- Core analytics journey and persisted non-transactional preference.
- Logout and denial on back/refresh.
- Non-invited adult denial.

Automated tests do not replace the authentic invited-user acceptance gate.

## Delivery sequence

### Phase 0 — decisions and inventory

Reconcile the four resolved Baton decisions into the documents; enumerate alpha pages/APIs/data;
identify the real non-Vercel origin and Cloudflare configuration; define a non-Vercel rollback; and
approve the PRD and technical specification.

### Phase 1 — quality and security baseline

Create a reliable web CI gate, stop ignoring build/type errors, add route-level tests, and close
alpha-impacting critical/high security findings. Replace Vercel Analytics and Vercel Cron, then
remove `@vercel/analytics`, Vercel environment detection, `vercel.app` API fallback, the
`x-vercel-skip-auth` response header, and active Vercel deployment guidance.

### Phase 2 — app-owned auth, dark

Implement the Mystira provider and session behind a disabled feature flag or non-production client.
Remove demo/mock auth, protect routes, and prove negative cases. No production RP activation.

### Phase 3 — product slice

Reduce navigation to the selected experience and complete the one PRD journey against the chosen
system of record. Label representative data and exclude unfinished claims/features.

### Phase 4 — cross-repo registration

Land the accepted ADR addendum and disabled Mystira Terraform client. Confirm exact callback,
issuer, scopes, client type, and deployment origin. Production apply remains gated.

### Phase 5 — production alpha activation

Deploy the exact reviewed artifact to the verified Azure Container Apps origin behind Cloudflare,
purge stale edge content, and prove that responses contain no Vercel headers. Then close the
explicit owner gate, enable the RP/configuration, verify health and telemetry, and run authentic
invited-user acceptance. Roll back to the prior verified non-Vercel artifact if the callback or
session fails; do not improvise redirect URIs.

## Rollback

- Preserve the last known-good non-Vercel application artifact and hosting configuration.
- Application auth is activated by a reversible configuration flag independent of DNS.
- Mystira client enablement can be turned off without deleting its registration.
- Never delete the prior provider/redirect configuration in the same step that first enables
  Mystira; remove it after acceptance in a separate reviewed change.
- Cloudflare and Azure origin changes use their own rollback plan and approval; no rollback points
  to Vercel.

## Definition of alpha-ready

Alpha-ready requires all of the following, recorded separately in Baton:

- approved PRD and technical specification;
- resolved product and architecture decisions;
- exact-head source checks and reviews;
- deployed intended artifact and healthy dependencies;
- enabled, governed Mystira RP with no demo auth path;
- protected-route and negative-security evidence;
- operator telemetry and rollback rehearsal;
- authentic invited-user sign-in, core journey, return visit, and logout.

## Resolved decisions

| Decision                           | Selected technical direction                          | Baton decision |
| ---------------------------------- | ----------------------------------------------------- | -------------- |
| Alpha experience                   | Standard only                                         | `f4e3bab5`     |
| Canonical origin and callback host | `www.veritasvault.net`                                | `d743950f`     |
| Alpha system of record             | Supabase data; Mystira-only identity; .NET post-alpha | `deb2ec15`     |
| Mystira OIDC client contract       | Confidential client plus S256 PKCE                    | `141c10a2`     |

The product owner selected all four directions on 2026-09-04. Document approval, executable
non-production OIDC proof, and production activation remain separate gates.
