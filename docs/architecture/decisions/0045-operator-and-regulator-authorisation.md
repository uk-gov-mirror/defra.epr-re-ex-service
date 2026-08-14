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

The PoC that established regulator sign-in (PAE-1771) closed with an explicit request: "need a decision (documented via an ADR) about where in the stack to perform 'is the user a regulator' checks (based on the role)". That decision was never written, and two changes answered it independently and differently — one put the check in the frontend, one in the backend. Both were defensible. Neither was recorded, so the next person to add a route or a template had nothing to read that named the layer that owned the rule.

### The vocabulary this builds on

A **role** says who a user is. **Scopes** say what they may do. A role is a named bundle of scopes, and a route declares the scopes it needs, never a role. This separation was established in the admin frontend by ADR 0033, and applied to the backend auth layer by the clean-up under PAE-1556 and PAE-1661, which removed the conflation of the two and named the operator-side permissions for what they are.

### What was inconsistent

The two frontends resolved permissions by different means. The admin frontend asked the backend once at sign-in, through `GET /v1/admin/me`, and stored the answer on the session — the pattern ADR 0016 named as its target and ADR 0033 adopted. `epr-frontend` instead read the Entra `roles` claim itself, inside its Bell profile function, and minted its own session scope. The mapping from identity to permission therefore lived in two codebases, and they drifted: the frontend rendered operator read routes for a regulator that the backend refused.

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

### 4. A scope names an entity and an operation, never a population

`organisation.read` and `organisation.linked.write` name what the holder acts on. `admin.read`, `admin.write` and `regulator` name the holder instead. The difference decides whether a second population can be given the same permission.

Both role-named scopes have reached that point. A regulator needs most of what `admin.read` covers — the organisation list, the summary-log list and file, the market reports, the data extracts — and must be refused the rest of it, because system logs, form submissions and linked organisations carry personal data (PAE-1077). One flat scope cannot express that. Meanwhile `regulator` gates nothing: no route requires it, and every regulator read reaches its route through `organisation.read`.

So the service has one scope vocabulary, drawn on entities and operations, and every role is a bundle over it.

Five operation classes separate permissions in this domain. Each is a different act, not the same act on a different noun:

| Class                 | What sets it apart                                       |
| --------------------- | -------------------------------------------------------- |
| read in context       | the caller already holds the organisation id             |
| search across         | enumerates the population; the caller holds no id yet    |
| download the artefact | the operator's original upload, not the parsed model     |
| bulk extract          | the dataset leaves the service at scale                  |
| aggregate             | no single operator is identifiable; publication-destined |

A finer split fails the test that earns these five. No one holds "read the registration" without "read the accreditation", so those are one scope.

The read vocabulary:

| Scope                   | Covers                                                                                                                                    |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `organisation.read`     | the operator record in context — organisation, registrations, accreditations, PRNs, summary logs, waste balances, reports, overseas sites |
| `organisation.search`   | enumeration across operators — the organisation list, the report submissions list, the PRN lists                                          |
| `summary-log.file.read` | the uploaded spreadsheet, as the operator submitted it                                                                                    |
| `market-data.read`      | public register, tonnage monitoring, PRN tonnage, waste-balance availability                                                              |
| `data-extract.read`     | the waste-record, summary-log and market-insight extracts                                                                                 |
| `service-data.read`     | system logs, form submissions, linked organisations, the dead-letter queue                                                                |

`organisation.read` and its linked and write siblings exist today. This ADR names the other five. `admin.read` is their union, so it stops existing when they land, and `regulator` stops existing with it.

Writes keep their present shape. No regulator-initiated write exists: a service administrator performs each regulator decision, for registration and accreditation status transitions (PAE-1598) and for PRN cancellation (PAE-1807). This ADR names no regulator write scope, because a scope named before its capability invents a requirement.

Every role is a bundle. No role loses access it holds today: the four read scopes that replace `admin.read` are granted together to every tier that holds `admin.read` now, and the regulator bundle is the subset that carries no personal data.

| Role                     | Scopes                                                                                                             |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| operator                 | `organisation.read`, `organisation.write` — both on the link condition — and `organisation.linked.read` / `.write` |
| regulator_standard       | `organisation.read`, `organisation.search`, `summary-log.file.read`, `market-data.read`, `data-extract.read`       |
| support                  | the regulator bundle and `service-data.read`                                                                       |
| service_maintainer       | the support bundle and `admin.dlq.purge`                                                                           |
| service_maintainer_write | the maintainer bundle and `admin.write`                                                                            |

A route declares a scope from the vocabulary. It never names a bundle. A regulator persona that arrives later — the caseworker, the analyst, the field auditor that PAE-1728 must still decide between — is a new row in this table, and changes no route.

### 5. An identity matching more than one rule receives the union

An Entra identity holding a regulator application role and also sitting on a service-maintainer email list receives both bundles. There is no ordering and no short-circuit.

The alternative — first match wins — makes an Entra role assignment silently destructive. Granting a service maintainer a regulator role would remove their admin access, with no change in any repository, no deployment, and no signal but a `403`.

### 6. A session carries a positive identity

An identity that resolves to no role gets `{ role: null, scopes: [] }`, and the frontend renders the not-authorised page. It does not mint a session usable on operator routes.

This closes a class of defect rather than an instance. Where the absence of a scope is a tolerated state, a guard written against a specific scope does not fire, and the session falls through to whatever the default is. Every session is an operator, a regulator or a maintainer, decided at sign-in.

### 7. Frontend wiring

1. **Fetch at sign-in and on refresh.** `epr-frontend` calls the backend for `{ role, scopes }` when a session is established, and again when tokens refresh, and stores the result on the server-side session. The staleness window is the token refresh cadence, not the session lifetime. The admin frontend already does this through `GET /v1/admin/me`. `GET /v1/me` is the equivalent for any identity.
2. **Routes declare scopes.** A frontend route declares `auth: { scope }` exactly as a backend route does. Until a route carries its own declaration, a blanket guard refuses every non-GET request from a session that holds no write scope. The framework enforces the same rule once the declaration arrives, so the guard shrinks as routes gain them.
3. **Templates render on scope presence.** A write control appears because the session holds a write scope, not because the user is an operator. No template knows what a regulator is. The test for an author: if this were a different role with the same permissions, would the markup change? If not, use the scope.
4. **Redirects choose on role.** Where a user lands after sign-in, which navigation shell renders, what the header calls the service — these are questions about identity, not permission, and cannot be derived from a scope set. They use the role.

### 8. What each layer owns

The backend owns the decision and is the only gate. The frontend owns the shape — what renders and where a user lands.

The frontend's guards are defence in depth and may assume nothing. A write that gets past them is still refused. This restates the position ADR 0033 already took: "Frontend hiding is UX, not security. The backend scope checks remain the actual gate."

### 9. The forwarded token carries the role claim

The backend can only resolve a regulator if the token it receives carries the role. The Entra `roles` claim arrives on the access token — the application requests `api://{clientId}/.default` specifically so that it does — and not on the ID token.

An Entra session therefore presents its access token to the backend. A Defra ID session presents its ID token.

### 10. A scope changes without a migration

The backend derives a credential's scopes on every request, from the token's `roles` claim and the email lists. It never reads them from the token, and does not store them.

The instance that derives the set is the instance that then checks it against the route. Two properties follow, and they are what make the vocabulary cheap to change.

**A rename ships in one deployment.** During a rolling deployment every instance grants and demands the same strings, because both come from the same artefact. An old instance grants `admin.read` and its routes ask for `admin.read`. A new instance grants the new name and its routes ask for the new name. No request meets a route that demands a string the resolver did not grant. A rename therefore needs no expand-and-contract sequence.

**A change to a role's bundle takes effect on the next request.** Nothing is cached that must expire first. A scope added to a bundle, or removed from one, reaches every signed-in user as soon as the deployment completes.

A frontend holds a copy of the derived set, not the decision. Between a backend deployment and a frontend deployment, a template can test for a string the session does not carry. The page renders the wrong affordance. The backend still refuses the write, so the skew costs a confusing page and never a permission.

A session's copy therefore lags a bundle change by the token refresh cadence. This ADR sets no shorter cadence: a shorter one costs a round trip per navigation, and buys a faster correction to an affordance the backend was already gating.

## Consequences

### Enabled

- One frontend serves both populations over the same entities, and reaches regulators in all four nations.
- A read route admits a regulator without its author doing anything, so the frontend and backend cannot drift over which routes a regulator may read.
- Regulators are refused writes by absence rather than by a list. Nothing enumerates what a regulator may not do, so nothing can be forgotten from that enumeration.
- Adding a regulator role later is a security-group assignment and a map entry. No route or template changes.
- Both frontends resolve permissions the same way, so a reader learns one pattern.
- A regulator reads the summary-log file, the organisation list and the data extracts without reading the personal data that sits beside them in `admin.read` today.
- Market data can be granted on its own, so a reader who needs the published aggregates never receives an operator record.

### Costs

- A new backend endpoint, and a round trip at sign-in and on each refresh.
- Every route that carries `admin.read` is re-declared against the scope its data belongs to. The change is mechanical, and it touches the routes the admin frontend and the regulator frontend share.
- `summary-log.file.read` is a strict escalation of `organisation.read`: a holder of the file scope can already read the parsed data. It earns a separate scope only where a reader gets the data and not the artefact.
- A frontend's cached scope set can only be refreshed by asking the backend, so nothing on the session can detect that it has gone stale.

### Not solved

- **Jurisdiction.** A regulator sees every organisation, not only those their authority regulates. Narrowing `organisation.read` is a grant condition, the same shape as the operator's link check. Narrowing `organisation.search`, `data-extract.read` and `market-data.read` is not, because a condition on the request cannot filter a result set or a file, and those are the scopes where jurisdiction matters most. Neither is designed here.
- **The regulator role taxonomy.** One flat role exists. This ADR describes how a second would be added, and adds none. PAE-1728 must first decide how many regulator personas the service serves.
- **Regulator writes.** Registration and accreditation status transitions, PRN cancellation, and overseas-site maintenance are the functions that will need a regulator write scope. None is designed here, because a service administrator performs all three today.
- **Machine credentials.** The vocabulary describes three human populations. Two machine credentials sit outside it: the basic-auth credential carries `organisation.read`, the same scope a regulator holds, and the API gateway credential carries no scope at all, so its routes are authorised by the strategy alone. Whether a machine takes a scope from this vocabulary, or a vocabulary of its own, is not decided here.
- **Two role strings.** `EPR.Regulator` and `Waste.Regulator.Standard` both exist and mean different things to different consumers. Neither is named canonical here.
- **Service-maintainer assignment.** Maintainer tiers still come from email lists in configuration rather than from Entra groups. ADR 0033 named group-based assignment as a future concern and it remains one.
