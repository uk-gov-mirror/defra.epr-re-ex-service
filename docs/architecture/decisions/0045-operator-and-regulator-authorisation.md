# 45. Authentication and authorisation for operators and regulators

Date: 2026-08-12

## Status

Proposed

Extends [ADR 0016 (Admin UI Authorisation MVP)](0016-admin-ui-authorisation-mvp.md) and [ADR 0033 (Admin UI scope-based RBAC)](0033-admin-ui-scope-based-rbac.md) from the admin frontend to the operator frontend.

Supersedes the statement in [the high-level design](../defined/pepr-hld.md#how-do-users-access-this-service) that regulators access the service through the admin frontend, and the corresponding statement in [the API low-level design](../defined/pepr-lld-auth-api.md) that `epr-re-ex-admin-frontend` is the application used by regulators.

## Context

`epr-frontend` now authenticates two kinds of user. Operators sign in with Defra ID, as they always have. Regulators sign in with Entra ID, added under PAE-1798. The service therefore has one public frontend serving two populations who are told apart by which identity provider issued their token.

### Why the operator frontend and not the admin frontend

Two reasons, and the second is the durable one.

Non-EA regulators have no access to the Defra network. The admin frontend was restricted to Defra staff on that network because it exposes a JSON editor over the database. SEPA, NRW and NIEA staff cannot reach it. PAE-1077 recorded the requirement plainly: "non EA regulators do not have access to The Defra Network. We will need a solution that allows all regulators access."

More importantly, operators and regulators read the same entities. An organisation, a registration, an accreditation, a summary log, a PRN, a waste balance — a regulator supervises exactly the records an operator submits. Building a second frontend over the same entities duplicates every page to no purpose. As regulator functions become self-service in `epr-frontend`, the admin frontend narrows to what it was built for: service maintenance.

### Why this ADR exists

The PoC that established regulator sign-in (PAE-1771) closed with an explicit request: "need a decision (documented via an ADR) about where in the stack to perform 'is the user a regulator' checks (based on the role)". That decision was never written, and two changes have since answered it independently and differently — one placing the check in the frontend, one in the backend. Both are defensible. Neither is recorded, so the next person to add a route or a template has nothing to read that tells them which layer owns the rule.

### The vocabulary this builds on

A **role** says who a user is. **Scopes** say what they may do. A role is a named bundle of scopes, and a route declares the scopes it needs, never a role. This separation was established in the admin frontend by ADR 0033, and applied to the backend auth layer by the clean-up under PAE-1556 and PAE-1661, which removed the conflation of the two and named the operator-side permissions for what they are.

### What is inconsistent today

The two frontends resolve permissions by different means. The admin frontend asks the backend once at sign-in, through `GET /v1/admin/me`, and stores the answer on the session — the pattern ADR 0016 named as its target and ADR 0033 adopted. `epr-frontend` instead reads the Entra `roles` claim itself, inside its Bell profile function, and mints its own session scope. The mapping from identity to permission therefore lives in two codebases, and they have already drifted: there are operator read routes the frontend renders for a regulator that the backend still refuses.

## Decision

### 1. Authentication establishes identity and grants nothing

| Provider | Authenticates                                 | Zone                       |
| -------- | --------------------------------------------- | -------------------------- |
| Defra ID | Operators — external, organisation-linked     | Public                     |
| Entra ID | Regulators and service maintainers — internal | Public, restricted by role |

Both mint a session in `epr-frontend`. Both are verified against the issuing provider's JWKS. Neither decides access.

### 2. Authorisation is resolved in one place, in the backend

`epr-backend` holds the only mapping from an authenticated identity to a set of scopes. No frontend, template or route handler performs that mapping.

Resolution has two halves, and separating them is what makes it safe for a frontend to cache the result.

**Durable resolution** runs at sign-in and again on token refresh. It answers "who is this, permanently?" from the identity alone:

- an Entra `roles` claim carrying a regulator application role resolves to the regulator role
- an Entra `preferred_username` on a service-maintainer email list resolves to that tier
- a Defra ID identity resolves to the operator role

**Per-request resolution** runs on every API call. It adds the scopes that depend on the request itself — today, the operator's access to their own linked organisation.

A frontend receives the durable half only. A sign-in-time answer cannot say whether an operator may read organisation 42, because that question has no answer until there is a request. The backend keeps deciding per-request access on every call, so nothing cached can go stale in a way that grants access.

### 3. A regulator reads what an operator reads

`organisation.read` already means "may read the organisation this request is about". Its grant condition differs by role:

| Scope                                 | An operator holds it                                                     | A regulator holds it |
| ------------------------------------- | ------------------------------------------------------------------------ | -------------------- |
| `organisation.read`                   | when the request's `organisationId` is their linked, active organisation | always               |
| `organisation.write`                  | on the same condition                                                    | never                |
| `organisation.linked.read` / `.write` | always — the scope concerns their own links                              | never                |

No operator read route changes to admit a regulator. The grant lives in the resolver, so a read route written next year admits a regulator without its author knowing regulators exist.

There are no carve-outs. A regulator may read anything an operator may read, for any organisation. A regulator changes nothing.

The operator path is unchanged by this ADR. The organisation-link check, the linked scopes and the conditions on them stay exactly as they are. The regulator grant is a new branch alongside them.

### 4. Two families of scope

**Shared scopes** are the same permission held by both populations on different conditions — `organisation.read` above. They gate the operator pages a regulator reads.

**Regulator-only scopes** gate what only a regulator does. Today there is one, and it is coarse: it stands for the whole set of regulator functions. It subdivides as caseworking functions arrive, at which point a route names the member it needs.

### 5. An identity matching more than one rule receives the union

An Entra identity holding a regulator application role and also sitting on a service-maintainer email list receives both bundles. There is no ordering and no short-circuit.

The alternative — first match wins — makes an Entra role assignment silently destructive. Granting a service maintainer a regulator role would remove their admin access, with no change in any repository, no deployment, and no signal but a `403`.

### 6. A session carries a positive identity

An identity that resolves to no role gets `{ role: null, scopes: [] }`, and the frontend renders the not-authorised page. It does not mint a session usable on operator routes.

This closes a class of defect rather than an instance. Where the absence of a scope is a tolerated state, a guard written against a specific scope does not fire, and the session falls through to whatever the default is. Every session is an operator, a regulator or a maintainer, decided at sign-in.

### 7. Frontend wiring

1. **Fetch at sign-in and on refresh.** `epr-frontend` calls the backend for `{ role, scopes }` when a session is established, and again when tokens refresh, and stores the result on the server-side session. The staleness window is the token refresh cadence, not the session lifetime. The admin frontend already does this through `GET /v1/admin/me`; the operator frontend needs the equivalent for any identity.
2. **Routes declare scopes.** A frontend route declares `auth: { scope }` exactly as a backend route does. Most `epr-frontend` routes declare none today, which is why a blanket guard is needed instead (see the interim position below).
3. **Templates render on scope presence.** A write control appears because the session holds a write scope, not because the user is an operator. No template knows what a regulator is. The test for an author: if this were a different role with the same permissions, would the markup change? If not, use the scope.
4. **Redirects choose on role.** Where a user lands after sign-in, which navigation shell renders, what the header calls the service — these are questions about identity, not permission, and cannot be derived from a scope set. They use the role.

### 8. What each layer owns

The backend owns the decision and is the only gate. The frontend owns the shape — what renders and where a user lands.

The frontend's guards are defence in depth and may assume nothing. A write that gets past them is still refused. This restates the position ADR 0033 already took: "Frontend hiding is UX, not security. The backend scope checks remain the actual gate."

### 9. The forwarded token must carry the role claim

The backend can only resolve a regulator if the token it receives carries the role. `epr-frontend` currently forwards the ID token on every backend call, while the Entra role arrives on the access token — the application requests `api://{clientId}/.default` specifically so that it does.

This ADR does not choose between forwarding the access token and configuring Entra to place the role claim on the ID token. It records that the forwarded token must carry the role, and that this is not true today.

## Interim position, August 2026

Regulator sign-in is behind the `FEATURE_FLAG_REGULATOR_ACCESS` flag: enabled in `dev`, `test`, `ext-test` and `perf-test`, disabled in `prod`. A penetration test on 18 August probes the separation of the two providers in `ext-test`.

The deployed code diverges from the model above in four places. Each is scheduled, not accepted.

| Divergence           | Deployed behaviour                                                    | Target                                               |
| -------------------- | --------------------------------------------------------------------- | ---------------------------------------------------- |
| Regulator read grant | A dedicated regulator read scope, admitted on a named list of routes  | The grant moves into the resolver; no route names it |
| Precedence           | First match wins; the application role short-circuits the email lists | Union                                                |
| Frontend affordances | Templates and the write guard test whether the session is a regulator | Both test for scope presence                         |
| Forwarded token      | The ID token, which does not carry the role claim                     | A token that carries the role claim                  |

The frontend's blanket write guard — a refusal of every non-GET request from a regulator session — is transitional. It exists because `epr-frontend` routes do not yet declare scopes. As they gain declarations the framework enforces the same rule, and the guard shrinks to nothing.

## Consequences

### Enabled

- One frontend serves both populations over the same entities, and reaches regulators in all four nations.
- A read route admits a regulator without its author doing anything, so the frontend and backend cannot drift over which routes a regulator may read.
- Regulators are refused writes by absence rather than by a list. Nothing enumerates what a regulator may not do, so nothing can be forgotten from that enumeration.
- Adding a regulator role later is a security-group assignment and a map entry. No route or template changes.
- Both frontends resolve permissions the same way, so a reader learns one pattern.

### Costs

- A new backend endpoint, and a round trip at sign-in and on each refresh.
- The regulator grant is invisible at the route. A reader of a route declaration cannot tell that regulators reach it; only the resolver says so. This is the price of the drift being unrepresentable, and it needs a comment where the resolver grants it.
- The four divergences above have to be closed, and the work is not sequenced by this ADR.

### Not solved

- **Jurisdiction.** A regulator sees every organisation, not only those their authority regulates. Narrowing would be a grant condition on `organisation.read`, the same shape as the operator's link check, but it is not designed here.
- **The regulator role taxonomy.** One flat role exists. This ADR describes how a second would be added, and adds none.
- **Two role strings.** `EPR.Regulator` and `Waste.Regulator.Standard` both exist and mean different things to different consumers. Neither is named canonical here.
- **Service-maintainer assignment.** Maintainer tiers still come from email lists in configuration rather than from Entra groups. ADR 0033 named group-based assignment as a future concern and it remains one.
