# CrossCentral: national hub of the Czech Red Cross

Status: **draft for discussion** · Oct 1, 2026

## Summary

CrossCentral is the national-level application of the Czech Red Cross (ČČK).
It has two jobs:

1. **Federation hub.** It keeps a registry of every MemberBase deployment in
   the country (one per Oblastní spolek) and carries requests between them and
   from national bodies to them. The typical case: the Ústřední krizový tým
   (ÚKT) asks every Oblastní spolek in one kraj for the contact details of
   members holding certain qualifications.
2. **National agenda.** It serves the internal work of the national level,
   starting with Úřad ČČK. The first piece is a member register for the
   people who use CrossCentral (roles and permissions inside the app). Later
   candidates: a knowledge base (internal regulations, practical guides), a
   repository of logos and visual rules, and a logo generator for branches.

It is built by the MedCover and MemberBase team and follows MemberBase's
architecture, stack and conventions. Code and docs are English; the UI is
Czech. Úřad ČČK is the first user; other national societies may run their
own deployment later.

Goals:

- Every MemberBase deployment is discoverable: who runs it, where it is, which
  kraj it belongs to, whether it is alive, which protocol version it speaks.
- A request for member data never bypasses the source Oblastní spolek: a
  person there always approves what leaves.
- The hub carries as little personal data as possible, for as short a time as
  possible, and logs every access to it.
- Reuse MemberBase where it fits, without coupling the two so tightly that
  one organisation level's needs break the other.

## Glossary (Czech UI → English code)

| Czech | English (code) | Notes |
| --- | --- | --- |
| Český červený kříž (ČČK) | national society | One CrossCentral deployment per national society |
| Úřad ČČK | national office | First user of the national agenda |
| Ústřední krizový tým (ÚKT) | central crisis team | Main requester of member data |
| Krajský krizový tým | regional crisis team | Assumed; works inside one kraj (Q-3, Q-12) |
| Kraj | region | 14 in Czechia; generic `region` in code (AD-8) |
| Oblastní spolek (OS) | district | One MemberBase deployment each; `crcDistrictId` |
| Místní skupina (MS) | unit | Lives inside a MemberBase deployment only |
| Evidence členů | MemberBase | |
| Žádost | request | Existing MemberBase concept, extended across deployments |
| Osvědčení / kvalifikace | certificate / qualification | Standardised nation-wide (AD-12) |
| Interní předpisy | internal regulations | Later: knowledge base |
| Znak / logo | emblem / logo | Use regulated by law (zákon č. 126/1992 Sb.) and the ČČK visual manual |

Czech UI name of CrossCentral: **open** (Q-1).

## Context

Today:

- Every Oblastní spolek runs its own MemberBase deployment: OpenLDAP (system
  of record), Keycloak (login, MFA), the MemberBase Flask app, and MedCover as
  a consumer. No SQL in MemberBase; audit from slapd `accesslog`. No shared
  stacks for several spolky.
- Deployments already carry a `crcDistrictId` and use UUIDs for people, units
  and qualifications, so identifiers are unique across deployments.
  Communication between deployments was explicitly left out of MemberBase
  ("identifiers ready").
- MemberBase already has requests („Žádosti“) inside one deployment: filed by
  one person, decided by the Chairs of the owning Místní skupina, carried out
  by MemberBase. Cross-deployment requests are the same idea one level up.
- MemberBase's rule: a member's personal data does not leave their Místní
  skupina unless its leadership approves. Cross-deployment requests must
  respect it (AD-4).
- Qualifications and certificates are defined per deployment today; there is
  no national catalogue.
- There is no national-level system.

## Functional Requirements

Source legend: **stated** = from the brief or our discussion; **derived** =
follows from a stated requirement or from MemberBase rules; **assumed** = my
inference, to confirm; **later** = design must not block it, not built in the
first release.

| ID | Requirement | Source |
| --- | --- | --- |
|  | **Registry of MemberBase deployments** |  |
| FR-1 | CrossCentral keeps a registry of all MemberBase deployments in the country: `crcDistrictId`, Oblastní spolek name, IČO, kraj, base URL, contact person, protocol version, status. | stated, derived |
| FR-2 | A new deployment joins the registry through an enrolment step approved by a CrossCentral admin; no deployment can register itself silently. | assumed |
| FR-3 | A deployment can be suspended or removed from the registry; a suspended one neither sends nor receives requests. | assumed |
| FR-4 | The registry shows each deployment's health: last contact, software and protocol version. | assumed |
| FR-5 | Each Oblastní spolek is assigned to exactly one kraj; requests can target a kraj, a list of Oblastní spolky, or the whole country. | stated, derived |
|  | **Requests between deployments** |  |
| FR-6 | An authorised CrossCentral user (e.g. ÚKT) sends a member-data request to one or more Oblastní spolky, with criteria (e.g. certificate, role, status, Místní skupina) and the data level wanted (`basic`, `contact`, `extended` as in MemberBase). | stated |
| FR-7 | A MemberBase deployment sends a request to another MemberBase deployment through CrossCentral (e.g. a member moving between Oblastní spolky, access to named people of another spolek). Exact request kinds: Q-6. | stated |
| FR-8 | Only predefined request kinds exist; there are no free-form queries. | stated |
| FR-9 | Every request needs an explicit approval by an authorised person of the source Oblastní spolek; there is no automatic answer, not even in a crisis. | stated |
| FR-10 | The receiving Oblastní spolek sees incoming requests in MemberBase „Žádosti“ and decides them (approve all, some, none), as with today's in-deployment requests. | derived |
| FR-11 | The requester sees per-target status (delivered, pending, approved, rejected, expired, failed) and the combined answer once targets reply. | derived |
| FR-12 | Every request has a stated purpose and an expiry; unanswered requests expire. | assumed |
| FR-13 | Answers can be viewed on screen and exported (XLSX); every view and export is logged. | assumed |
| FR-14 | Answered data is removed from CrossCentral after a retention period. | assumed (GDPR) |
| FR-15 | Every step (sent, delivered, decided, answered, viewed, exported, purged) is audited with who, when and which deployment. | derived |
|  | **National catalogue of certificates** |  |
| FR-16 | Certificates and qualifications that requests can ask about are defined once nation-wide, with stable codes, and every MemberBase deployment uses those definitions. | stated |
|  | **National member register and access** |  |
| FR-17 | CrossCentral has its own register of the people who use it (Úřad ČČK staff, ÚKT and regional team members, national bodies), with roles and permissions in the app. | stated |
| FR-18 | Roles can be scoped to the whole country or to one kraj; a regional team member works in CrossCentral only on their kraj, as an MS Chair works in MemberBase only on their Místní skupina. | stated (likely), Q-3 |
| FR-19 | Login with SSO, TOTP and passkeys, as in MemberBase; mandatory second factor for admins and for anyone who can request member data. | derived |
|  | **Later** |  |
| FR-20 | Knowledge base: articles and documents (internal regulations, practical guides) with versions, categories, search and per-role visibility. | later |
| FR-21 | Brand repository: official logos in all variants and formats, the visual manual, colours and fonts, downloadable by members. | later |
| FR-22 | Logo generator: produce a correct logo for a named branch (e.g. „Oblastní spolek ČČK Praha 1“) following the visual rules; output SVG, PDF, PNG. | later |
| FR-23 | Another national society can run its own CrossCentral deployment with its own regions, organisation levels and language. | later (Q-8) |

## Non-Functional Requirements

| ID | Requirement | Source |
| --- | --- | --- |
| NFR-1 | All UI, emails and generated documents are in Czech; code, repository and docs in English. | stated |
| NFR-2 | Same stack, tooling and conventions as MemberBase and MedCover, so the same people can develop and operate it. | stated |
| NFR-3 | Each Oblastní spolek remains the owner of its members' data; nothing leaves without its approval. | stated, derived |
| NFR-4 | Data minimisation: the hub stores only what a request needs, for a bounded time; metadata outlives payloads. | derived |
| NFR-5 | MemberBase keeps working, including its local requests, when CrossCentral is down; cross-deployment requests queue and resume. | derived |
| NFR-6 | Every deployment authenticates to CrossCentral with its own credential that can be revoked without touching others. | derived |
| NFR-7 | TLS on every hop; secrets never in the repository; real deployment details only in the private infra repo. | derived (MemberBase rule) |
| NFR-8 | The inter-deployment protocol is versioned; CrossCentral and the MemberBase deployments upgrade independently. | derived |
| NFR-9 | Scale: about 70 Oblastní spolky, tens of thousands of members in total, tens to low hundreds of CrossCentral users. | assumed |
| NFR-10 | Availability 24/7, no HA required; in a crisis the ÚKT must be able to send requests within minutes. RPO/RTO as MemberBase (1 day / 12 h) unless Q-10 says otherwise. | assumed |
| NFR-11 | 100 % line and branch test coverage, as MemberBase. | derived |
| NFR-12 | Open-source, free components; runs on Azure Container Apps like MemberBase. | derived |
| NFR-13 | Processing complies with GDPR; open points are listed in „GDPR“ below and block the first real request until resolved. | stated |

## Architectural Decisions

Status per AD: **open** = options still being collected, no preference yet;
**proposed** = has a leaning, not agreed; **decided** = agreed in discussion.

### AD-1 Form of the application and the national member register — open

**Problem statement:** CrossCentral needs a member register with roles and
permissions (FR-17, FR-18) and also data that does not fit LDAP (registry,
request queue, later documents). The national organisation structure is not
known yet (Q-12): it may have the Úřad with departments, the ÚKT, regional
teams per kraj and national bodies, none of which look like OS → MS. How much
of MemberBase do we reuse?

| Option | Pros | Cons |
| --- | --- | --- |
| A. Unmodified MemberBase deployment for the national level (own OpenLDAP + Keycloak); CrossCentral is a separate app that logs in through it, like MedCover | Member register, roles, grants, invites, MFA, audit exist today; no new member code | MemberBase's model is OS → MS; national bodies and kraj-scoped teams would be forced into „Místní skupiny“. Features built for spolky (Chairs, moves, certificates) may make no sense nationally. Every MemberBase change must be checked against national use, or it backfires there |
| B. Generalise MemberBase: organisation levels and unit types configurable per deployment, one codebase serving OS and national level | One member codebase; national needs become features | Makes MemberBase more complex for every spolek; national requirements drive changes in a product that ~70 spolky run; release coupling |
| C. CrossCentral has its own register on the same building blocks (OpenLDAP + Keycloak, the same patterns: proxied authorization, ACL tests, `accesslog`), with its own schema and tree shaped for the national structure | Fits whatever the national structure is; same skills and patterns; MemberBase stays focused | Similar code in two repositories; fixes to shared patterns done twice |
| D. As C, with the common code (directory access, OIDC login, permission checks) extracted into a shared Python package used by both | No duplicated plumbing; each app keeps its own domain model | A third repository to version and release; changes must stay compatible with both apps |
| E. CrossCentral keeps users and roles in its SQL database, Keycloak's own user store for login (no LDAP) | Simplest; one store for all CrossCentral data | Visibility enforced only in app code; diverges from MemberBase patterns; MFA and SSO still via Keycloak but a different user model to learn |

Criteria to decide: the national organisation structure (Q-12); how much of
MemberBase's member features the national level actually needs; whether
national users are separate accounts or come from the spolky (Q-3).

### AD-2 Message flow between deployments — open

**Problem statement:** How do requests and answers travel between
CrossCentral and about 70 MemberBase deployments, and between two MemberBase
deployments? (FR-6–FR-11, NFR-5, NFR-10)

#### Topology

| Option | Pros | Cons |
| --- | --- | --- |
| Hub and spoke: every message passes through CrossCentral | One place for registry, routing, status and audit; a deployment trusts only the hub; MemberBase-to-MemberBase requests (FR-7) need no direct trust | Hub holds data in transit (AD-5); hub outage stops cross-deployment traffic (local work unaffected) |
| Peer to peer, hub only as registry and key directory | No central data store | Every deployment must reach and authenticate every other one (~70² relationships); each deployment exposes an endpoint; no central status or audit; harder upgrades |

The rest of this AD assumes hub and spoke; it is the only option that keeps
FR-11 and FR-15 in one place.

#### Delivery mechanism

The question is how a MemberBase deployment learns that a request is waiting,
and how its answer gets back. Answers always go MemberBase → CrossCentral
over an outbound HTTPS call, in every option.

**P1. Polling (MemberBase pulls on a schedule).** MemberBase calls
`GET /inbox` every N seconds, acknowledges what it stored, and posts answers.

- Requires: outbound HTTPS from MemberBase; inbox, acknowledge and answer
  endpoints at the hub; idempotent acknowledge (a message may be fetched
  twice). MemberBase's jobs run every 15 minutes today as a Container Apps
  job; N = 1 minute needs a cron of one minute (the Container Apps minimum)
  or a small always-on worker.
- Consequences: latency up to N, plus the time a person needs to approve,
  which dominates anyway. Load: 70 deployments × 1/min ≈ 100 000 cheap calls
  a day. A deployment offline for a day catches up on its next poll. No
  inbound endpoint on any deployment. The hub sees „last contact“ for free
  (FR-4).

**P2. Long polling or server-sent events (MemberBase holds a connection
open).** MemberBase opens `GET /inbox?wait=…`; the hub answers as soon as
something arrives or after a timeout, and MemberBase reconnects.

- Requires: an always-on worker in MemberBase (a scheduled job can't hold a
  connection); hub endpoints that park requests without tying up a worker
  (async server or a separate process); ingress idle timeouts on both sides
  set above the wait time *(verify Azure Container Apps limits)*.
- Consequences: near-instant delivery, still no inbound endpoint. More moving
  parts than P1 in both apps; reconnect and backoff logic; Flask with
  gunicorn sync workers is a poor fit for many parked connections.

**P3. Push (the hub calls a webhook on each MemberBase).** The hub sends the
request to `POST /federation/inbox` on the deployment.

- Requires: every MemberBase exposes a public endpoint that accepts calls
  from the hub and authenticates them (signed JWT or mTLS); the hub runs a
  delivery queue with retries, backoff and a dead-letter state per target;
  each deployment's ingress allows it.
- Consequences: instant delivery. New attack surface on every deployment
  that holds member data. Offline deployments are the hub's problem (retry
  for hours or days). Two code paths for answers vs requests.

**P4. Push a signal, pull the content.** The hub sends a contentless
„something is waiting“ call to the deployment; the deployment then pulls as
in P1. A slow P1 poll (e.g. every 15 minutes) is the fallback when signals
are lost.

- Requires: the inbound endpoint of P3, but it carries no data and does
  nothing except trigger a fetch; the P1 endpoints.
- Consequences: near-instant without holding connections. Still an inbound
  endpoint per deployment, but its worst-case abuse is an extra poll. Two
  mechanisms to build and test.

**P5. Message broker (Azure Service Bus, RabbitMQ).** One queue per
deployment; hub and deployments both connect outbound to the broker.

- Requires: a broker to run (RabbitMQ) or pay for (Service Bus); per-queue
  credentials per deployment; the hub still needs its own API and database
  for registry, status and audit.
- Consequences: durable, standard delivery with retries built in; no inbound
  endpoint on deployments. A new component for volunteers to operate, or
  vendor lock-in; at this volume (dozens of requests a day at most) most of
  its features go unused.

Summary:

| | P1 polling | P2 long poll | P3 push | P4 signal + pull | P5 broker |
| --- | --- | --- | --- | --- | --- |
| Delivery delay | ≤ N (1 min) | seconds | seconds | seconds | seconds |
| Inbound endpoint on MemberBase | no | no | yes, carries data | yes, carries no data | no |
| Deployment offline | catches up | catches up | hub retries | catches up | broker keeps it |
| New components | none | always-on worker | delivery queue at hub | both of the above, smaller | broker |
| MemberBase change | job + client | worker + client | API endpoint + auth | endpoint + job + client | broker client |
| Fits current Flask/jobs setup | yes | partly | yes | yes | yes |

Points for the decision: is a delivery delay of a minute acceptable given
that a person must approve every request anyway (FR-9)? Is a new inbound
endpoint on each deployment acceptable?

### AD-3 Deployment authentication — proposed

**Problem statement:** How does CrossCentral know a call comes from a given
MemberBase deployment (and, for P3/P4, a deployment know a call comes from
the hub), and how is that revoked? (NFR-6)

| Option | Pros | Cons |
| --- | --- | --- |
| A. Static API key per deployment (bearer header) | Trivial to build | Long-lived shared secret on both sides; leaks in logs or proxies are replayable; rotation by hand |
| B. OAuth2 client credentials with a client secret; one client per deployment in CrossCentral's Keycloak | Standard; short-lived tokens; revoke = disable client | Still a shared secret, stored in Keycloak and in the deployment |
| C. OAuth2 client credentials with `private_key_jwt`; the deployment keeps the private key, Keycloak holds its public key | No shared secret; short-lived tokens; revoke = disable client; Keycloak already runs | Enrolment must register the public key; key rotation procedure needed |
| D. Mutual TLS with client certificates from our own CA | Strong; authenticates at connection level | Running a CA; Azure Container Apps ingress support for client certificates is limited *(verify)*; certificate renewal on ~70 deployments |
| E. HTTP message signatures (RFC 9421) with registered per-deployment keys, on top of B or C | Each message is signed, so a stored request or answer proves its origin later (useful for audit and for disputes) | More code on both sides; library support in Python is young *(verify)* |

**Leaning:** C, with E as an option for answers if we want the requester to
verify that the hub did not alter them. Enrolment (FR-2): the spolek admin
generates a key pair in MemberBase; a CrossCentral admin approves it.

### AD-4 Approval and producing the answer — open

**Problem statement:** Every request is approved by a person in the source
spolek (FR-9, decided). Who approves, and with whose rights is the answer
read? MemberBase's rule is that data does not leave a Místní skupina without
its leadership's approval, so an OS-level approval may not be enough. (FR-10,
NFR-3, Q-2)

Who approves:

| Option | Pros | Cons |
| --- | --- | --- |
| A. One OS-level person (OS koordinátor, ředitel OS) approves for the whole spolek | One decision per spolek; fast | Overrides the MS leadership's say over their members' data, unless the organisation agrees that crisis requests are an OS matter |
| B. The request is split per Místní skupina; each MS Chair approves for their people | Matches MemberBase's data rule exactly | A kraj-wide request becomes dozens of decisions; slow; one inactive Chair blocks part of the answer |
| C. OS-level approval, with MS Chairs notified and able to withdraw their people before the answer is sent (a short objection window) | Fast default, MS keeps a voice | Window delays the answer; more states to build |
| D. Depends on the request kind: crisis kinds at OS level (A), others per MS (B) | Each kind gets the right trade-off | Two decision flows to build and explain |

How the answer is read after approval:

| Option | Pros | Cons |
| --- | --- | --- |
| R1. As the approving person (proxied authorization) | Directory access rules apply to the answer; nobody gets more than the approver may see | The approver must be able to read every targeted person at the requested level; an MS Chair reads only their MS, so this pairs with approval options B or D |
| R2. By MemberBase's service account, after the approval is recorded (as today's approved moves and access requests are carried out) | Works whoever approves; same pattern as existing requests; re-checks the approval before acting | Service account reads beyond the approver's rights; one more agreed exception to MemberBase's „runs as the person“ rule |
| R3. The approver sees a preview of exactly the people and fields that would be sent and can remove people before confirming; then R1 or R2 sends it | The approver knows what leaves; per-person exclusions | Long lists are tedious to review; more UI |

R3 combines with R1 or R2. Decision depends on Q-2.

### AD-5 What the hub stores — open

**Problem statement:** Answers contain personal data of many people. Where
do they live, and for how long? (FR-13, FR-14, NFR-4)

| Option | Pros | Cons |
| --- | --- | --- |
| A. Hub stores answers encrypted at rest, readable only by the requester and their team, purged after N days; metadata kept for audit | Simple; works when the source deployment is offline later | Hub is a store of personal data for N days |
| B. End-to-end encryption to the requester's key; hub sees only metadata | Hub breach reveals nothing | Key management for people in browsers; lost key = lost answer; much more code |
| C. Hub stores nothing; the requester fetches answers live from each deployment | No central data | Needs an inbound endpoint on each deployment (conflicts with P1/P2/P5); fails if a deployment is down |

Retention N and export rules: see „GDPR“.

### AD-6 Request format — decided (predefined kinds)

**Problem statement:** How expressive are requests, and how do two versions
of the software agree on them? (FR-6–FR-8, NFR-8)

| Option | Pros | Cons |
| --- | --- | --- |
| **Predefined request kinds, each a versioned JSON schema; criteria from a closed vocabulary (national certificate codes, role, status, Místní skupina, kraj); fields only as MemberBase visibility levels** | Each kind is reviewed once for data protection; easy to show to the approver in Czech; testable | New kind needs a release on both sides |
| Free-form query (LDAP filter or similar) | Flexible | Approver cannot judge it; injection and over-collection risk |

**Decision:** predefined kinds only. The registry records which kinds and
versions each deployment supports; the hub refuses to route a kind the
target does not speak.

### AD-7 Persistence — proposed

**Problem statement:** Where does CrossCentral keep the registry, requests,
answers, audit, the certificate catalogue and later documents?

| Option | Pros | Cons |
| --- | --- | --- |
| LDAP only (like MemberBase) | No new store | Queues, payloads and documents fit LDAP badly |
| **SQL database (Azure SQL, Flask-SQLAlchemy + Flask-Migrate, as MedCover) plus Blob storage for files** | Known to the team; transactions for request state; audit table | One more database |

**Leaning:** SQL + Blob for CrossCentral's own data. Where people live
depends on AD-1.

### AD-8 Regions — proposed

**Problem statement:** Requests and roles are scoped to a kraj (FR-5, FR-18);
other countries have other structures (FR-23).

| Option | Pros | Cons |
| --- | --- | --- |
| **`region` table as data (Czech deployment: 14 kraje); each registered deployment and each regional role points to one region** | No code change for another country or a territorial reform | One more table |
| Kraj as a hardcoded enum | Simplest | Blocks FR-23 |

**Leaning:** data, one level deep; deeper hierarchies when a country needs
them.

### AD-9 Multi-country — proposed

**Problem statement:** One deployment for several national societies, or one
per society? (FR-23)

| Option | Pros | Cons |
| --- | --- | --- |
| **One deployment per national society** | Data stays in its country and legal entity; no tenant isolation code | Each society operates its own stack |
| Multi-tenant service | One stack | Tenant isolation everywhere; cross-border processing questions |

**Leaning:** one deployment per society. Translation readiness: Q-8.

### AD-10 Knowledge base and brand repository (later) — open

**Problem statement:** Build or adopt? (FR-20, FR-21)

Criteria: Czech UI, login through Keycloak (OIDC), per-role visibility,
version history of pages and files, full-text search including attached
PDFs, approval or publishing workflow for regulations, effort to operate,
licence.

| Option | Pros | Cons |
| --- | --- | --- |
| A. Build into CrossCentral | One app, one look, same roles | Editor, versioning, search and attachments are large to build well; competes with the hub for team time |
| B. BookStack (PHP, MySQL/MariaDB) | Mature, simple structure (shelves, books, pages), OIDC, Czech translation, per-item permissions *(verify)* | PHP stack and MySQL new to the team |
| C. Wiki.js (Node.js) | Modern editor, OIDC, several databases incl. PostgreSQL *(verify MSSQL support)* | Version 3 long in development; future of v2 unclear |
| D. Outline | Polished editor, OIDC | Business Source Licence, not open source (conflicts with NFR-12); needs PostgreSQL, Redis, S3 storage |
| E. Microsoft 365 / SharePoint, if ČČK has a nonprofit tenant | Familiar, Office documents, search | Separate identities unless federated with Keycloak; not open source; licence terms for nonprofits *(verify)* |
| F. Confluence | Rich, familiar | Commercial; cloud-hosted; separate identities |

Brand repository: either a section of the chosen knowledge base, or a simple
file listing in CrossCentral on Blob storage. A dedicated asset management
system is out of proportion for a few dozen files.

### AD-11 Logo generator (later) — open

**Problem statement:** How do branches get correct logos with their name?
(FR-22)

| Option | Pros | Cons |
| --- | --- | --- |
| A. No generator: a designer produces the logos for every branch once (the list is finite: spolky from the registry, Místní skupiny from the deployments) and they go to the brand repository | No code; a human checks every result | Redo for new or renamed branches; a backlog when many change |
| B. Server-side generator: official vector emblem + templates per layout; the branch name is set in the licensed font and converted to outlines on the server; output SVG, PDF, PNG | Exact rules every time; font file never leaves the server; names can be taken from the registry instead of free text, which prevents misuse | Encoding the visual manual's rules (clear space, sizes, line breaks of long names) is real work; needs a review by the brand owner |
| C. Browser-side generator | No server code beyond static files | Font must be shipped to the browser, likely violating its licence; easier to abuse |
| D. Editable templates (vector files or an online design tool) for local designers | Cheap | No control over the result; the rules are only as good as each designer |

Prerequisites for any option: the official visual manual, the font licence,
and the rules for using the emblem.

### AD-12 National certificate catalogue — open

**Problem statement:** A country-wide request for, say, members with a valid
first-aid instructor certificate only works if every spolek means the same
thing by it (FR-16). Today definitions are per deployment.

| Option | Pros | Cons |
| --- | --- | --- |
| A. CrossCentral is the master of the catalogue; MemberBase deployments sync it (pull) and use national definitions read-only | One truth; changes reach everyone without a release | MemberBase depends on the hub for definitions (cached, so not for operation); a new sync to build |
| B. National catalogue + local extensions: national definitions synced read-only as in A, spolky may add local ones that requests never ask about | Standard where it matters, freedom elsewhere | Two kinds of definitions in MemberBase's UI |
| C. Catalogue shipped with MemberBase releases (schema/LDIF) | No runtime dependency | Every change needs a release and its rollout to ~70 deployments |
| D. Local definitions stay; each spolek maps them to national codes | Least change to MemberBase | Mappings drift; a wrong mapping silently changes who a request finds |

Also to settle: who owns the catalogue (which national body), whether
validity rules (expiry, renewal) are national too, and how existing local
definitions migrate.

## GDPR (open)

Not decided; to be resolved with the DPO (pověřenec) before the first real
request (NFR-13). Known questions:

- Controller and processor roles: each Oblastní spolek is a separate legal
  entity (pobočný spolek). Who is the controller for data the ÚKT or a
  regional team receives? The national ČČK, the source spolek, joint
  controllers? Who operates CrossCentral, and is it a processor for the
  spolky?
- Agreements needed between ČČK and the spolky (processing agreement, joint
  controller arrangement).
- Legal basis per request kind: legitimate interest, vital interests in a
  crisis, consent, tasks under zákon č. 126/1992 Sb.
- Whether a DPIA is required (large-scale processing of member data across
  the country).
- Informing members: privacy notice that their data may be shared with
  national and regional teams; whether a member can see who received their
  data (could be shown in MemberBase).
- Retention: how long answers stay in CrossCentral (AD-5), how long audit
  records stay.
- Exports: whether XLSX export is allowed, to whom, and what rules apply to
  the file once it leaves the system.
- Records of processing activities for CrossCentral.
- Hosting location (EU region) and, for other countries later, cross-border
  transfers.

## Impact on MemberBase

CrossCentral needs a new part in MemberBase, behind a setting so a deployment
without CrossCentral works as today. Its shape depends on AD-2, AD-3, AD-4
and AD-12:

- Enrolment: key pair (AD-3), `crcDistrictId`, CrossCentral URL.
- Receiving requests (job, worker or endpoint, per AD-2) and sending answers.
- Incoming requests shown in „Žádosti“ with the requesting body, purpose,
  criteria and level; decided per AD-4.
- Outgoing requests to other spolky (FR-7).
- National certificate catalogue (AD-12).

## Open questions

| ID | Question | Status |
| --- | --- | --- |
| Q-1 | Czech UI name of CrossCentral? | open |
| Q-2 | Who decides an incoming request in an Oblastní spolek: the OS koordinátor, the Chairs of the affected Místní skupiny, the OS director? See AD-4. | open |
| Q-3 | How do ÚKT and regional team members log in? Likely they log in to CrossCentral and work there, scoped to their kraj (FR-18). Open: separate national accounts, or login with their spolek account via identity brokering from the spolek's Keycloak. Depends on Q-12. | open |
| Q-4 | Answers without human approval in a crisis? | **decided:** no, approval is always needed (FR-9) |
| Q-5 | Retention of answers, export rules. | moved to „GDPR“ |
| Q-6 | Which request kinds are needed first: member transfer (přestup), access to named people, certificates, crisis contact lists, something else? | open |
| Q-7 | National catalogue of certificates? | **decided:** yes, certificates are standardised nation-wide (FR-16); how: AD-12 |
| Q-8 | Prepare the UI for translation now (other national societies), or stay Czech-only until a second society commits? | open |
| Q-9 | GDPR roles and legal basis. | moved to „GDPR“ |
| Q-10 | Hosting and budget: whose Azure subscription, same Container Apps environment as the spolky or a separate one? | open |
| Q-11 | Does every Oblastní spolek run its own MemberBase? | **decided:** yes, one deployment per spolek |
| Q-12 | What is the national and regional structure of the people using CrossCentral: Úřad departments, ÚKT, regional crisis teams per kraj, national bodies? Who appoints them and who administers their accounts? Feeds AD-1, Q-3 and FR-18. | open |
