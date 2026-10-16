
## Initial request
In light of the results of our meeting (QUATAL Company for Rehabilitation and Educational Services) and your esteemed company on Saturday, 10/10/2026.  
The following are the demands of Quatal Company in favor of the professional accreditation system for Egyptian workers wishing to travel to the Kingdom of Saudi Arabia:  
Establishing an electronic platform (Web based – VPS) to operate the system, which includes:  
1. Allowing the user to upload their data and national ID card/passport. 
2. Locations of the accredited training centers affiliated with us. 
3. Sending WhatsApp messages to users. 
4. The ability to register fingerprints (hand - face) on the system. 
5. Register all user data by the company. 
6. Opening a screen in each training center containing:  
   - User data (Name – National ID – Passport – Profession ... etc.)  
   - Fingerprint matching  
   - Test data  
   - The result  
1. Download the theoretical question bank with a search engine.
2. Download the practical question bank with a search engine.
3. The presence of user permissions and roles with data protection and an unmodifiable audit log for sensitive operations.
4. The ability to generate reports and analyze data.
## Technologies used
Front-end: Vue
Back-end: .Net core
Database: Postgresql
File server: Minio
## Description prompt
I want to make a project that is to execute this workflow: 
the company adds user initial data (name and national id) the users then open the platform submit their detailed information and attachments (following KYC standard/process), this info gets confirmed in the companies admin panel, the user then goes to the company to verify and have his fingerprint taken.
the user is later sent a message over WhatsApp to tell me the location and time of an exam he has to take, the exam is taken in a separate place from the company itself (exam center), they also have a panel that shows what users are gonna take exams there and coordinate the exam itself.
the user can get the question bank for the exam from the platform so they can study before the exam
the exam itself is also on the platform only but needs to be taken at the exam center but the system in total is online so there needs to be a way to make sure the user can only take the exam there (a code he receives there for example)
at the exam center the user needs to verify his/her identity to the center using the same Fingerprint he recorded at the Company so a flow how to make sure that his finger print can be recorded and compared between the 2 places correctly
the user than takes the exam which consists of questions from the bank in random order then the score is calculated and shown to the company on their panel and also to the exam center for users that took the exam there.
one of the 2 exams is practical so the scores are submitted to the system manually by the center.
the user if he passes is then sent abroad to another Company we'll call it Job-place, at the Job-place they also have a panel to see which users are coming and when the user gets there he scans his fingerprint again to verify he is the same person and confirm arrival, this sets him as fulfilled for the Company and the flow is done.
The Company can at any time extract detailed reports over time of who came and who was tested and who fulfilled and everything and can show success graphs over time and turnover rate of users.
The total number of concurrent users will never reach 200.
the project auth will be Role based security using JWT.
i want to know:  
1 - what VPS specs I would need (using Contabo)  
2 - how to solve the fingerprint issue in 2 cases (if all devices are the same type and if all devices are of different types from different manufacturers)  
3 - how long this project would take and how many people should be working on it  
4 - a decision on if the user should have an account and can log in at any time and edit his data before choosing to submit and use that account for the exam along with exam code received there or should just register through KYC from a link sent to him that's a link specific to him and take the exam using an exam code generated specifically for him (so the user basically never actually logs in)

## Technical solution
### VPS specs :
	1. 8-Core CPU
	2. 24 GB RAM
	3. 200 NVME and SSD storage = Estimate users x attachment types x 5 MB
	4. a seperate backup server for database with half the specs
### Tech stack:
	1. Vue.js v3.5
	2. .NET 10
	3. Minio latest version
	4. PostgreSQL version 18.6

## Conversion prompt
### ROLE
You are a senior software architect and engineer. You will convert my existing LMS project into a much smaller, purpose-built system called "Workforce Placement Platform" (described below). The LMS is far larger than what this project needs, so your job is mainly to REMOVE, SIMPLIFY and REPURPOSE, and only secondarily to ADD.

### HOW TO WORK (IMPORTANT)
Work in phases. Do NOT start deleting or rewriting until I approve the Phase 1 report.

- Phase 0: Read the entire repo. Detect the stack, architecture, auth (Keycloak integration), multi-tenancy mechanism, hierarchy structures, DB schema, migrations, and quiz/exam code. Report what you found.
- Phase 1: Produce a written MIGRATION PLAN (no code changes) with a table of every module/entity/endpoint/page classified as KEEP, SIMPLIFY, REPURPOSE, or DELETE, with a one-line reason each. Include a dedicated section on (a) how multi-tenancy and hierarchy will be removed and (b) how Keycloak will be replaced by the custom JWT service. List the risks. STOP and wait for my approval.
- Phase 2: Create a new git branch. Delete everything marked DELETE (commit separately so it's revertable). Get the solution building and migrations clean.
- Phase 3: Build the new domain (entities, state machine, API, panels) one module at a time. After each module: build, run tests, commit, and give me a short summary.
- Phase 4: Reports, audit, hardening, Docker/deployment files, documentation.

If the existing code is not ASP.NET Core, tell me in Phase 0 and propose how to port the reusable logic (quiz engine, file handling) rather than silently rewriting everything.

### TARGET STACK (fixed decisions)
- Backend: ASP.NET Core (.NET LTS), EF Core with migrations, PostgreSQL
- Frontend: Vue 3 (TypeScript if the LMS already uses it), one app with role-based routing/layouts
- File storage: MinIO (S3 API), private buckets, access only through short-lived presigned URLs
- Auth: custom JWT service (see AUTH section). NO Keycloak, NO OIDC/SSO, NO external identity provider. Classic username/password login. NO OTP.
- Background jobs: Hangfire (PostgreSQL storage). NO Kafka. Use a transactional outbox table plus Hangfire for notifications, report generation and audit side effects
- Notifications: WhatsApp API for exactly TWO message types (cost control): (1) registration invite with an activation link, (2) exam location/time notice. Everything else is in-platform only. Put it behind an `INotificationSender` interface so it can be swapped or disabled
- Deployment: Docker Compose (api, postgres, minio, nginx, hangfire in the api process). Remove the Keycloak container and its database from compose. Main VPS: 8 vCPU / 24 GB RAM / 200 GB NVMe. Separate backup server (half specs) receiving automated daily pg_dump and MinIO sync. Write the backup scripts and a restore runbook
- Scale: fewer than 200 concurrent users. Do NOT add caching layers, microservices, CQRS, or message brokers. Keep it a clean modular monolith
- Team: 3 people (lead, 1 backend, 1 frontend). Favor simple, readable code over cleverness

### THE BUSINESS (explain every flow to yourself, then implement)

This is a SINGLE-ORGANIZATION system. There is one Company that owns everything. Exam Centers and Job-places are simply reference entities the Company works with, not tenants.

Actors and roles (JWT claims):
- `CompanyAdmin`: the main company. Full control, reports.
- `CompanyStaff`: the company's operators (KYC review, fingerprint enrollment). Permissions are configurable if cheap, otherwise just a subset of admin.
- `CenterStaff`: works at an Exam Center (a separate physical location). Sees only their own center's data.
- `JobPlaceStaff`: works at the Job-place abroad. Sees only candidates assigned to their Job-place.
- `Candidate`: the end user.

Roles are FLAT. No role inheritance trees, no parent/child roles.

Candidate lifecycle (this is the central state machine; every transition must be validated, role-checked, and written to the audit log):

`PreRegistered → InviteSent → Activated → KycSubmitted → KycApproved (or KycRejected → back to editable) → FingerprintEnrolled → ExamScheduled → VerifiedAtCenter → ExamInProgress → ExamCompleted → Passed / Failed → Dispatched → ArrivedVerified → Fulfilled`

(If `InviteSent` and `Activated` are better modeled as flags or timestamps rather than states, propose that in Phase 1.)

Flow details:

1. PRE-REGISTRATION AND INVITE: CompanyStaff creates the candidate with only name, national ID and phone (single entry and bulk import via Excel/CSV, with validation and a duplicate check on national ID). The system generates a single-use, expiring (configurable, default 7 days) activation token and queues WhatsApp message #1 via outbox + Hangfire: it tells the candidate to register their data and contains the activation link. The link opens a page where the candidate confirms their national ID and SETS THEIR OWN PASSWORD, which activates the account. Username is the national ID. Never send passwords over WhatsApp. Staff can resend or regenerate an invite (invalidating the old token, with a resend cooldown and a max-resend count to protect WhatsApp costs). Fallback: staff can generate a one-time activation link to hand over in person if WhatsApp delivery fails. Track delivery status per message.

2. KYC: the candidate logs in, fills detailed personal info, and uploads required attachments (document types are configurable; reuse the LMS's file/attachment code if it exists). Editable and saveable as a draft until they press Submit. After Submit, the profile is locked. CompanyStaff reviews in the admin panel: approve, or reject with per-field/per-document comments, which unlocks the profile for the candidate to fix. Admin can also correct data directly (audited).

3. FINGERPRINT ENROLLMENT (at the Company, in person): staff captures the candidate's fingerprints via a local capture agent. See the BIOMETRICS section. Record the candidate's consent before capture. Candidate becomes FingerprintEnrolled.

4. EXAM SCHEDULING: CompanyStaff assigns candidates to an Exam Session (center, date/time, exam type, capacity). When scheduled, the system queues WhatsApp message #2 with the center location and time (outbox + Hangfire, retry, delivery status column). The candidate can also see this in the platform. If a session is rescheduled or cancelled, show it in the platform and let staff choose whether to send a new WhatsApp message (default: do not, to save cost). There are TWO exam types per candidate: (a) a theory exam taken on the platform, (b) a practical exam scored manually (see below).

5. STUDY: the candidate can view/download the question bank for their exam (read-only study material, available after KycApproved or per a configurable rule). Reuse LMS question-bank entities; strip them down to: Exam (type), Question, Choices, correct answer(s), category (optional).

6. EXAM DAY AT THE CENTER: the platform is online, so the exam must only be takeable at the center.
   a. Candidate arrives; CenterStaff opens the session in the center panel and sees the expected list.
   b. CenterStaff verifies identity by fingerprint 1:1 against the enrolled prints (see BIOMETRICS). On success → VerifiedAtCenter. On repeated failure (3 attempts) → supervisor override with mandatory reason (and optionally a photo), audited.
   c. Only after verification, CenterStaff clicks "Authorize exam", which generates a single-use exam code bound to (candidate, exam session, center), valid for a short window (e.g., 15 minutes to START, configurable) and invalidated when used.
   d. The exam may only be started from registered center workstations: enforce with an allowlist of center IPs/CIDR and/or registered workstation tokens per center, configurable in the admin panel. The candidate logs in with their normal account and enters the exam code. Without a valid code AND an allowed origin, the exam endpoints must refuse. Also reject starting if the candidate's session is not VerifiedAtCenter.
   e. Exam engine (reuse the LMS quiz engine, but simplify aggressively): server-side random selection and ORDER of questions from the bank (configurable count per exam), answers persisted continuously, server-side timer (client timer is cosmetic), resume after disconnect within the time limit, one attempt per authorization, scoring done server-side, correct answers never sent to the client during the exam. Store a per-attempt snapshot of the questions served.
   f. Result: score and pass/fail (configurable pass mark) are visible in the Company panel and in the Center panel for candidates that sat there.

7. PRACTICAL EXAM: one of the two exams is practical. CenterStaff manually enters the score (and notes) per candidate in the center panel. Audit who entered or changed it and when. Edits after submission require a reason. The candidate's overall Passed/Failed is computed from both exams according to configurable rules.

8. DISPATCH: for passed candidates, CompanyStaff assigns them to a Job-place with an expected arrival date. State becomes Dispatched.

9. ARRIVAL AT JOB-PLACE: JobPlaceStaff sees expected arrivals in their panel. On arrival they scan the candidate's fingerprint (1:1 verification against enrolled prints, same override rules). On success → ArrivedVerified and then Fulfilled (this marks the flow done for the Company). Record timestamp, staff, and device.

10. REPORTING (Company only): filterable, exportable (Excel and PDF) reports over date ranges: registered, invited/activated, KYC approved/rejected, enrolled, tested (by center/exam), pass/fail rates, dispatched, arrived, fulfilled. Dashboard charts: funnel/conversion by stage over time, pass rate over time, fulfillment (success) over time, turnover/drop-off rate (candidates who stalled or dropped at each stage, with configurable "stalled after N days"), time spent per stage. Derive these from the state-transition audit table (do not duplicate data).

### BIOMETRICS (design this behind an abstraction)
Browsers cannot talk to USB fingerprint scanners, so a small local capture agent runs on each kiosk PC (localhost HTTPS/WebSocket) and the Vue app calls it. Do NOT hardcode a vendor.

- Define `IFingerprintMatcher` (server side) and a capture-agent contract with this payload: `{ image (WSQ/PNG), imageFormat, dpi, deviceVendor, deviceModel, deviceId, qualityScore, fingerPosition }`.
- Enrollment: capture at least 2 fingers (e.g., both index), 2-3 samples each, reject captures below a configurable quality threshold. Store the RAW IMAGE (encrypted, in MinIO) as the source of truth and the derived ISO/IEC 19794-2 template (encrypted at rest in PostgreSQL), plus device metadata.
- Verification is ALWAYS 1:1 against the specific candidate's templates, never 1:N. Matching runs on the server via `IFingerprintMatcher`, with the threshold in config, and every attempt logged (score, device, result, staff).
- Ship a `NistBozorthMatcher` (or similar open-source) implementation as the default, and a `MockMatcher` for dev/test, so the system works end-to-end before the real hardware SDK is chosen. Leave a clearly documented place to plug in a vendor SDK.
- Write a doc explaining both scenarios: (A) all devices same vendor/model, (B) mixed vendors, and what must change for B (standard formats only, re-extract templates from stored raw images with one matcher).
- Encrypt biometric data, restrict access by role, log every read. Include a consent record per candidate (Egypt's Personal Data Protection Law 151/2020).

### AUTH AND SECURITY: REPLACE KEYCLOAK WITH A CUSTOM JWT SERVICE
Remove Keycloak ENTIRELY: the Keycloak container/realm exports/themes/config, OIDC/OAuth middleware, `keycloak-js` or any OIDC client in the frontend, token-introspection or admin-API calls, SSO/social login, Keycloak role/group mappings, and any Keycloak-specific claims or user-sync code. No external identity dependency may remain. Grep the whole repo (backend, frontend, compose, CI, docs) at the end to prove it.

Build an `Identity` module that owns auth:
- Users, roles and refresh tokens in PostgreSQL. Use ASP.NET Core Identity with a custom, minimal setup (or a lean custom user table with `PasswordHasher`) and say in Phase 1 which you chose and why.
- Roles: exactly the five above, flat, as claims. Policy-based authorization on top.
- Short-lived access token (e.g., 15 min) signed with a key from configuration/secret (RS256 preferred, HS256 acceptable if simpler; document the choice and how to rotate the key). Claims: user id, role, and for CenterStaff/JobPlaceStaff their center id / job-place id.
- Opaque refresh tokens stored HASHED in the DB, rotated on every use, with reuse detection (reuse revokes the token family), expiry, and revoke on logout/password change/admin disable.
- Endpoints: login, refresh, logout, change password, activate account (invite token), admin password reset (generates a one-time reset link or temp password shown once to the admin; no email/WhatsApp dependency), disable/enable user.
- Password policy, strong hashing, account lockout after repeated failures, rate limiting on login and activation endpoints, forced password change when an admin resets a password.
- Activation and reset tokens: random, single-use, stored hashed, expiring.
- Frontend: replace the Keycloak flow with a login page, token storage strategy (access token in memory, refresh token in an HttpOnly Secure SameSite cookie preferred; justify your choice), an HTTP interceptor with silent refresh, route guards by role, and clean logout.
- Seed an initial CompanyAdmin via an environment-configured bootstrap (never hardcoded credentials).
- Existing Keycloak user data does NOT need migrating unless Phase 0 shows it contains something we must keep; tell me what you find and ask.

Other security rules:
- Row-level scoping: CenterStaff only see their center; JobPlaceStaff only their Job-place; Candidates only themselves. Enforce in the query layer (e.g., scoped repository/query filters based on claims), not just the UI. This is simple foreign-key scoping, NOT multi-tenancy.
- Append-only audit log for: every state transition, every data edit, every login, every biometric action, every score entry/override, every file access, every invite/reset.
- Files: validate type/size, scan extension/MIME, store in MinIO with random keys, serve through authorized presigned URLs.

### REMOVE MULTI-TENANCY AND HIERARCHY COMPLETELY
The LMS is multi-tenant and hierarchical in almost everything. This project has NO tenants and NO hierarchies. In Phase 0, map every place these appear; in Phase 2, remove them all:
- Tenant/organization/school/branch/campus entities, `TenantId`/`OrgId`-style columns on every table, EF global query filters, tenant resolution middleware (subdomain/header/claim), tenant-scoped configuration, tenant-aware services, tenant-aware caches, per-tenant file buckets/prefixes, per-tenant seed data, per-tenant frontend theming/routing/selectors.
- Hierarchical structures: parent/child org units, department/faculty/grade/class trees, closure tables, materialized paths, recursive queries, role or permission inheritance, hierarchical menus, nested categories (keep at most a flat optional category on questions).
- Clean the schema through the migration squash so no tenant/hierarchy columns, indexes or FKs remain. Simplify composite keys and unique constraints that included the tenant id.
- Final grep check for `tenant`, `organization`, `org`, `branch`, `parent` etc. and report what remains and why.

### WHAT ELSE TO REMOVE FROM THE LMS (verify each against the code in Phase 1)
Courses, lessons, content authoring, classrooms/virtual sessions, video/live features, scheduling/timetables, student rollover/promotion, grades/GPA/transcripts, attendance, assignments, discussion/chat/forums, certificates (unless trivially reusable), parent/teacher/student role hierarchy, payments/subscriptions, analytics unrelated to this flow, and unused frontend pages/routes/stores/i18n keys. Also remove their DB tables via a clean migration squash (a fresh initial migration for the new schema is acceptable if it is safer; tell me which you chose and why).

### WHAT TO KEEP/REUSE (verify)
User entity basics and password-hashing code (if not Keycloak-bound), role/permission middleware (after de-tenanting), file upload/attachment handling, question bank and quiz-taking/scoring logic (simplified), audit/logging, base layout/UI components, localization/RTL support if present (Arabic support is likely needed; confirm and preserve), and any report/export helpers.

### DELIVERABLES
1. `MIGRATION_PLAN.md` (Phase 1) with the KEEP/SIMPLIFY/REPURPOSE/DELETE table plus the multi-tenancy removal plan and Keycloak replacement plan.
2. Clean solution: modular monolith (Identity, Candidates/KYC, Biometrics, Exams, Centers and JobPlaces, Notifications, Reporting, Audit), each with its own folder and clear boundaries.
3. EF Core schema and migrations, seed data (roles, bootstrap admin, sample center, Job-place, exam, questions).
4. REST API with OpenAPI docs, consistent error format, role-based authorization attributes/policies.
5. Vue app: Candidate portal (activation, profile/KYC, study bank, status tracker, exam), Company panel (candidates, bulk import, invites, KYC review, enrollment, scheduling, dispatch, reports and charts, users/roles, centers, Job-places, settings), Center panel (session roster, fingerprint verify, authorize exam, practical score entry, results), Job-place panel (expected arrivals, fingerprint arrival confirm).
6. Capture-agent contract spec plus a mock agent for development.
7. `docker-compose.yml` (no Keycloak), nginx config, `.env.example`, backup and restore scripts for the second server, and a deployment guide for a single VPS.
8. Tests: unit tests for the state machine, JWT service (issue, expiry, refresh rotation, reuse detection, lockout), exam engine (randomization, timer, scoring, single-use codes) and authorization scoping; a few integration tests for the full happy path.
9. `docs/` with: architecture overview, the state machine diagram (Mermaid), flow-by-flow explanation for each of the 10 steps above, auth guide, biometrics guide, role/permission matrix, WhatsApp message templates and cost notes, and a runbook.

### RULES
- Ask me before making any irreversible or ambiguous decision. Batch your questions; don't ask one at a time.
- Do not invent features beyond this spec. Put ideas in `docs/FUTURE.md`.
- Prefer the simplest solution that meets the spec; 3 developers must maintain this.
- Keep commits small with clear messages. Never commit secrets.
- At the end of each phase, summarize what was done, what remains, and any deviations from this spec.

Begin with Phase 0 and Phase 1 now. Do not modify any files yet.