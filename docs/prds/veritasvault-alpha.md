---
document_type: overview
classification: internal
status: review
version: 0.2.0
last_updated: "2026-09-04"
applies_to:
  - Web
  - Identity
  - Alpha
reviewers:
  - "@product-owner"
  - "@tech-lead"
  - "@security"
priority: p1
next_review: "2026-12-04"
---

# VeritasVault Alpha PRD

## Status and decision boundary

This PRD is a proposal for review. It does not authorize implementation, a Mystira relying-party
registration, credentials, DNS changes, a hosting cutover, Terraform apply, or production
activation.

Implementation starts only after this PRD and the companion
[technical specification](../specs/veritasvault-alpha.md) are approved in Baton. Production
activation remains a separate explicit approval gate.

## Product summary

VeritasVault Alpha should turn the existing public prototype into a coherent, non-custodial
analytics product that a small invited cohort can use safely. The alpha is not a public launch. It
is the first evidence that a real adult user can authenticate, complete one useful portfolio
analytics journey, return later, and receive support without encountering demo authentication or
unprotected application routes.

## Verified starting point

Fresh verification on 2026-09-04 found:

- `www.veritasvault.net` serves a Next.js application. Its server-rendered root is a loading shell,
  but browser hydration reveals Standard and Corporate entry points.
- `/login`, `/standard/login`, and `/corporate/login` are public.
- Authentication is split across broken NextAuth GitHub/Google providers, Supabase Auth, a mock
  GitHub callback, and a demo credentials endpoint that accepts any email-shaped value plus a
  non-empty password.
- The .NET API repository contains substantial domain and infrastructure code, but it is not
  integrated with the web application and does not register an authentication scheme.
- `games.veritasvault.net` is DNS `NXDOMAIN` and is not part of this alpha.
- No off-main branch contains a hidden production-ready authentication replacement.
- A second GET after the owner confirmed Vercel is retired reached Cloudflare but returned a
  month-old cached artifact carrying legacy `x-vercel-*` headers. No matching VeritasVault
  Container App was visible in the active NeuralLiquid Azure subscription. The non-Vercel origin
  and Cloudflare cache state therefore require explicit verification before alpha.

This means the estate is neither an empty stub nor an alpha-ready product.

## Problem

The current deployment presents product and login surfaces without a single trustworthy identity,
session, authorization, or data boundary. Users cannot tell which experience is real, operators
cannot support an end-to-end journey, and deployment health cannot demonstrate that authentication
or business functionality works.

## Alpha users

### Primary user

An invited adult portfolio operator or digital-asset analyst who wants to inspect a portfolio,
understand its exposures, and review risk or allocation information without executing trades or
placing assets in VeritasVault custody.

### Secondary user

An internal product operator who invites users, confirms journey health, reviews telemetry, and can
disable access or roll back the release.

## Recommended alpha proposition

The selected scope is **Standard experience only**, for an invite-only cohort. Corporate remains
out of alpha until it has a distinct validated user and workflow.

The minimum useful journey is:

1. An invited adult user opens the canonical VeritasVault origin.
2. The user signs in through Mystira Identity using authorization code with S256 PKCE.
3. The user lands on one clearly labelled Standard dashboard.
4. The user views a portfolio or seeded representative portfolio, its allocation, and at least one
   explainable risk or analytics result.
5. The user can save a non-transactional preference or draft/watchlist and see it after returning.
6. The user signs out and loses access to protected pages and APIs.

No step may imply trade execution, custody, guaranteed return, or regulated investment advice.

## Goals

- One canonical application origin and one supported alpha experience.
- One identity authority: Mystira Identity.
- One application session and authorization model, keyed by immutable OIDC `(iss, sub)`, not email.
- All non-public pages and APIs fail closed for signed-out or non-invited users.
- One useful, repeatable analytics journey backed by an explicitly chosen data system of record.
- Reproducible build and deployment with health, error, authentication, and journey telemetry.
- A verified non-Vercel origin behind `www.veritasvault.net`, with no Vercel runtime dependency,
  fallback, analytics client, scheduled job, or response header.
- Documented rollback and support path.
- Authentic acceptance by at least one invited user on the production alpha origin.

## Non-goals

- Public self-service registration.
- Child or teen accounts.
- Corporate/enterprise workflows.
- Wallet custody, signing, swaps, trade execution, deposits, withdrawals, or financial advice.
- `vv-game-suite` or restoration of `games.veritasvault.net`.
- Completing every dashboard, AI feature, API route, or domain document already present.
- Forcing the .NET API into alpha solely because it exists.
- Reintroducing Vercel as a host, rollback target, analytics provider, scheduler, or API fallback.
- A broader Azure platform redesign beyond establishing and verifying the required non-Vercel alpha
  origin.
- Registry edits, DNS changes, or Mystira Terraform changes from this repository.

## Functional requirements

### Identity and access

- Mystira Identity is the only alpha sign-in authority.
- Access is invite-only and Adult-only.
- Demo credentials, mock callbacks, and unused competing provider buttons are absent or fail closed.
- Protected pages and APIs enforce the same session and cohort authorization.
- Logout invalidates the VeritasVault session. Provider-wide logout is added only if its exact
  post-logout redirect is registered and tested.

### Product journey

- The landing page clearly identifies the alpha and links to its single supported experience.
- A signed-in user can reach a stable dashboard without dead navigation.
- The selected portfolio/analytics journey uses real or explicitly labelled representative data.
- Data provenance and freshness are visible where a decision could otherwise be misleading.
- Errors provide a recoverable next action without exposing identity or system internals.

### Operator journey

- Operators can control the invited cohort without editing source code.
- Operators can identify failed sign-ins and failed core journeys without reading secrets or token
  contents.
- A release can be rolled back to the last known-good build without changing DNS or identity
  registration.

## Quality and security requirements

- No access or identity token is stored in browser `localStorage`.
- OIDC uses discovery, authorization code, S256 PKCE, state, and nonce through a maintained library.
- Issuer, audience/client, signature, expiry, nonce, and redirect URI are validated exactly.
- Return URLs are restricted to same-origin relative paths.
- Session cookies are `Secure`, `HttpOnly`, host-only (no `Domain` attribute), and use an
  appropriate `SameSite` policy.
- Tests cover signed-out, invited, non-invited, expired, replayed, and malformed callback states.
- Logging excludes passwords, authorization codes, verifiers, tokens, secrets, and raw session IDs.
- Existing high-severity security findings that affect the alpha journey are closed before release.

## Success measures

The cohort size is deliberately small; qualitative evidence matters more than vanity traffic.

- At least one authentic invited user completes sign-in, the core analytics journey, return visit,
  and logout on the production alpha origin.
- At least 95% of invited-user sign-in attempts complete during the acceptance window, excluding
  deliberate negative tests.
- All protected-route negative tests pass.
- No unresolved Severity 1 or Severity 2 defect affects identity, authorization, privacy, data
  integrity, or the core journey.
- Operators can correlate a failed journey across request, session-safe subject identifier, and
  deployment revision.
- Every alpha participant provides structured feedback on utility, trust, and the next missing
  capability.

## Alpha entry gates

1. PRD approved and linked as satisfied evidence in Baton.
2. Technical specification approved and linked as satisfied evidence in Baton.
3. The four product/architecture decisions on epic `bcfc1e75` are resolved.
4. Scope is decomposed into reviewable implementation tasks with owners and estimates.
5. No production identity, DNS, credential, or hosting mutation is bundled into an ordinary code
   task.

## Alpha release gates

1. Source gate: exact-head CI, tests, review, and no unresolved actionable review threads.
2. Identity gate: accepted ADR-0029 addendum and a disabled RP registration before activation.
3. Deployment gate: intended artifact/revision is healthy on the canonical origin.
4. Security gate: demo auth is unreachable; protected pages and APIs fail closed.
5. Production approval gate: explicitly closed by the owner before the RP is enabled.
6. Acceptance gate: authentic invited-user sign-in, callback, session, core journey, return visit,
   and logout are observed and recorded separately from deployment health.

## Resolved decisions

| Decision                           | Selected direction                                                                                            | Baton decision |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------- | -------------- |
| Alpha experience                   | Standard only; one coherent retail analytics journey                                                          | `f4e3bab5`     |
| Canonical origin and callback host | `www.veritasvault.net`; Cloudflare edge with a verified Azure Container Apps origin; no Vercel fallback       | `d743950f`     |
| Alpha system of record             | Supabase data with Mystira as the sole identity authority; .NET remains post-alpha unless separately approved | `deb2ec15`     |
| Mystira OIDC client contract       | Confidential client plus S256 PKCE; server-side BFF owns callback and secret                                  | `141c10a2`     |

These choices were selected by the product owner on 2026-09-04 and are recorded on Baton epic
`bcfc1e75`. They settle direction but do not satisfy the separate document-review or production
approval gates.
