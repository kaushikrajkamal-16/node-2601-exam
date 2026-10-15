# JobPortal BD - Architecture and UML Diagrams

Reverse-engineered from the uploaded React client (`Client.zip`) and Node.js server (`Server.zip`). The source code was the only source of truth; nothing was added for completeness. Application code was not modified. Only files under `docs/diagrams/` were created.

> **Rendering status: all 24 diagram sources rendered without errors and were inspected visually.**
> Exports are in `rendered/` (SVG and PNG, same base names as the sources). The `.mmd` / `.puml` files remain the authoritative sources.
> Details, tool versions and limitations are under "Validation status".

## Application summary

| Aspect | Finding |
|---|---|
| Frontend | React 19 + Vite 8, Tailwind CSS 4, React Router, Redux Toolkit / RTK Query (JavaScript + JSX, no TypeScript) |
| Backend | Node.js, Express 5, Mongoose 9, JWT in an httpOnly cookie, multer, Nodemailer, Cloudinary SDK (CommonJS JavaScript) |
| Database | MongoDB: collections `users`, `jobs`, `applications` |
| Roles | applicant, company (recruiter), admin (created by `npm run seed` or by another admin) |
| Architecture style | Routes -> middleware -> controllers (business logic) -> Mongoose models. **There is no separate service or repository layer.** Controllers contain the logic and call models and `utils/*` directly. |
| Source layout | `Server/` and `Client/src/` inside the two archives. They are **not part of this repository**, so line numbers in `traceability.md` refer to the uploaded copies. |

## Diagram index

| File | Type / language | Represents | Main evidence |
|---|---|---|---|
| `three-tier-architecture.mmd` | Mermaid flowchart | Logical presentation / application / data tiers and external services | `Client/src`, `Server/server.js`, `routes/`, `controllers/`, `model/` |
| `three-tier-architecture-classic.svg` | Hand-authored SVG | The same three tiers in the textbook block style (User, Client, Web Server, DBMS with database); the SVG is its own source | same evidence as `three-tier-architecture.mmd` |
| `three-tier-architecture-structure.svg` | Hand-authored SVG | Minimal textbook three-tier block diagram: User, Client (web browser), Web Server (web application), DBMS with Database, two-way arrows. Generic labels only; technologies are named in the other three-tier diagrams | same evidence as `three-tier-architecture.mmd` |
| `three-tier-architecture-detailed.svg` (+ `.mmd`) | Hand-authored SVG; the `.mmd` is the same content as plain Mermaid for pasting into draw.io (Arrange > Insert > Advanced > Mermaid), auto-laid-out so it looks different | Detailed three-tier view in the user's layout: client modules, access guard, Express pipeline, security / upload / controller / utility components, Mongoose models, external services, collections and constraints, legend | `Client/src`, `Server/server.js`, `routes/`, `middleware/`, `controllers/`, `utils/`, `model/` |
| `rest-api-layers.svg` (+ `.drawio`) | Hand-authored SVG; `rest-api-layers.drawio` is the same diagram as an editable draw.io file (native shapes, generated and structure-checked but not opened in draw.io here) | Logical layers of the REST API: entry point and HTTP pipeline, routing (with the guard on each router), security middleware, controllers, utilities and Mongoose models, external systems. States that no separate service or repository layer exists | `Server/server.js`, `routes/index.js`, `routes/api/*.js`, `middleware/*.js`, `controllers/*.js`, `utils/*`, `model/*` |
| `c4-system-context.mmd` | Mermaid flowchart, C4-styled | System, four human actors, Gmail SMTP, Cloudinary | roles in `User.js`; `mailService.js`; `uploadToCloudinary.js` |
| `c4-container.mmd` | Mermaid flowchart, C4-styled | SPA, REST API, MongoDB, external systems with protocols | `package.json` files, `baseApi.js`, `dbConnection.js` |
| `c4-component.mmd` | Mermaid flowchart, C4-styled | Backend components and dependencies | `server.js`, `routes/api/*`, `middleware/*`, `controllers/*`, `utils/*`, `model/*` |
| `c4-component-frontend.mmd` | Mermaid flowchart, C4-styled | Frontend components (extra view; frontend is large enough to warrant one) | `App.jsx`, `Components/`, `pages/`, `redux/` |
| `use-case-diagram.puml` | PlantUML use case | Actors, system boundary, include/extend, external actors | controllers and routes |
| `use-case-diagram-organized.svg` | Hand-authored SVG | Re-laid-out version of the use case diagram: groups stacked beside their actors, one connector rail per actor, include/extend use cases beside their base use case, external systems on the right, notes and legend in a panel. Same use cases and relationships as `use-case-diagram.puml`; the SVG is its own source | same evidence as `use-case-diagram.puml` |
| `class-diagram-organized.svg` | Hand-authored SVG | One-page class diagram: Part 1 domain model (User, Job, Application, StatusHistoryEntry, four enumerations, multiplicities, composition); Part 2 controllers with their operations, middleware and utility modules. Same content as `class-diagram.puml` + `class-diagram-modules.puml`, laid out by hand; the SVG is its own source | `Server/model/*.js`, `controllers/*.js`, `middleware/*.js`, `utils/*` (checked against both the uploaded zip and the GitHub repo: identical apart from line endings and `.gitignore`) |
| `erd-organized.svg` | Hand-authored SVG | Presentation-quality ERD in crow's-foot notation: users, jobs, applications and the embedded statusHistory, with PK / FK / UK badges, relationship lines drawn from the real foreign-key rows, legend and notes. Same data as `entity-relationship-diagram.mmd`; the SVG is its own source | `Server/model/User.js`, `Job.js`, `Application.js` (identical in the uploaded zip and the GitHub repo) |
| `erd-logical.svg` | Hand-authored SVG | Distributed LOGICAL ERD with 18 entities derived from the backend and the frontend: USER_ACCOUNT specialised into applicant / company / admin profiles, OTP_CODE, profile image, resume file, job post with skill / requirement / benefit lists, application with its status-event timeline, and six reference value sets. Each entity states which of the 3 physical collections stores it. Use `erd-organized.svg` for the physical schema | `Server/model/*.js`, `authController.UpdateProfile` (allowedFields per role), `Client/src/Components/*Modal.jsx`, `pages/Dashboard/*`, `pages/Jobs/JobsDetails.jsx` |
| `erd-detailed-logical.svg` | Hand-authored SVG | Controller-derived DETAILED logical ERD: OTP split into EMAIL_VERIFICATION_CODE and PASSWORD_RESET_CODE, non-stored ACCESS_TOKEN (JWT cookie), role profiles, application with status events and resume file, job lists, six value sets, four derived read models, delete cascades and rules R1-R13 as enforced in the controllers | `Server/controller/*.js` (authController, applicantController, companyController, adminController, jobController), `Server/model/*.js` |
| `class-diagram.puml` | PlantUML class | Domain model: the three Mongoose models, enums, embedded `StatusHistoryEntry`, multiplicities | `model/*.js` |
| `class-diagram-modules.puml` | PlantUML class | Backend function modules (controllers, middleware, helpers) and which models they use | `require()` statements across `Server/` |
| `entity-relationship-diagram.mmd` | Mermaid ER | MongoDB collections, refs, embedded `statusHistory`, indexes | `model/User.js`, `Job.js`, `Application.js` |
| `sequence-overview.puml` | PlantUML | One end-to-end sequence over the main path: register + e-mail code, login, company posts a job, applicant applies with CV, company updates status, applicant tracks it, admin moderation. Detailed per-flow diagrams with error branches follow below | `Server/routes/api/*.js`, `Server/controller/*.js`, `Client/src/features/**` |
| `sequence-authentication.puml` | PlantUML sequence | Login, session check (route guard), logout | `authController.js`, `authMiddleware.js`, `ProtectedRoute.jsx` |
| `sequence-registration.puml` | PlantUML sequence | Register, verify e-mail OTP, resend | `authController.js` |
| `sequence-password-reset.puml` | PlantUML sequence | Forgot, verify, reset | `authController.js`, `VerifyOtp.jsx` |
| `sequence-apply-job.puml` | PlantUML sequence | Apply with resume upload (primary applicant workflow) | `applicantController.js`, `upload.js`, `ApplyModal.jsx` |
| `sequence-candidate-status.puml` | PlantUML sequence | Company reviews candidates, changes status (primary company workflow) | `companyController.js`, `RecruiterDashboard.jsx` |
| `sequence-admin-delete-company.puml` | PlantUML sequence | Admin cascade deletion, job close | `adminController.js` |
| `activity-apply-for-job.puml` | PlantUML activity (swimlanes) | All business rules from click to stored application | as above |
| `activity-apply-and-track.svg` | Hand-authored SVG | Swimlane activity diagram (Applicant / Server / Company): login check, resume validation, job checks, Cloudinary upload, pending application, company status update, applicant tracking and withdrawal (blocked when approved) | `applicantController.js` (ApplyJob, WithdrawApplication), `companyController.js` (UpdateApplicationStatus, DeleteCandidate) |
| `activity-apply-and-track.drawio` | draw.io (native, black and white) | Editable draw.io version of the swimlane activity diagram; open in app.diagrams.net via File > Open from device | same as the SVG row above |
| `activity-apply-and-track-bw.svg` | Hand-authored SVG (black and white) | Same swimlane activity diagram without colour, for printed reports; PNG in `rendered/` | same as the SVG row above |
| `registration-login-activity.svg` / `.drawio` | Hand-authored SVG + native draw.io (black and white) | Swimlane activity diagram (User, React client, API server, E-mail service): client validation, POST /auth/register, server validation, OTP user creation, OTP e-mail, Verify OTP with wrong/expired code and resend, mark verified, login errors, JWT cookie, redirect by role to the applicant, recruiter or admin dashboard | `authController.js` (Registration, VerifyEmail, ResendOtp, login), `pages/Auth/Registration.jsx`, `VerifyOtp.jsx`, `Login.jsx`, `App.jsx` routes |
| `activity-job-posting-and-moderation.puml` | PlantUML activity (swimlanes) | Job creation, pause/edit/delete, admin close | `companyController.js`, `adminController.js` |
| `state-application-status.puml` | PlantUML state | `Application.status` lifecycle | `Application.js`, `UpdateApplicationStatus`, `WithdrawApplication` |
| `state-application-status-organized.svg` | Hand-authored SVG | Cleaner version of the application status machine: pending, shortlisted, exam, interview, approved, rejected; company can set any value from any value (no ordering); exits by WithdrawApplication (not when approved), DeleteCandidate and cascade deletes | `Server/controllers/companyController.js` (UpdateApplicationStatus, DeleteCandidate), `applicantController.js` (ApplyJob, WithdrawApplication), `model/Application.js` |
| `state-job-status.puml` | PlantUML state | `Job.status` lifecycle | `Job.js`, `TogglePauseJob`, `UpdateJob`, `CloseJob` |
| `state-user-account.puml` | PlantUML state | Verification and flag states | `User.js`, auth and admin controllers |
| `dfd-level-0.mmd` | Mermaid flowchart (DFD) | Context-level data flows | whole system |
| `dfd-level-1.mmd` | Mermaid flowchart (DFD) | 7 processes and 3 data stores | controllers, models |
| `deployment-diagram.puml` | PlantUML deployment | Only what the repo confirms (local dev defaults) | `server.js`, `baseApi.js`, scripts |
| `api-interaction-map.mmd` | Mermaid flowchart | 36 routes with callers, handlers, guards, entities | `routes/api/*`, `redux/*` |
| `traceability.md` | Markdown | Element-to-source evidence table | - |
| `implementation-gaps.md` | Markdown | Observed inconsistencies and unverifiable areas | - |

## Modeling choices and limitations

- **Logical tiers only.** The three-tier diagram shows responsibilities. Nothing in the repository says the tiers run on separate servers.
- **C4 views are drawn as C4-styled Mermaid flowcharts, not with Mermaid's experimental `C4Context` / `C4Container` / `C4Component` syntax.** The native C4 syntax was tried first and rendered badly (overlapping relationship labels, edges crossing unrelated boxes). The flowcharts keep C4 conventions: person / system / container / component stereotypes in each box and a system boundary.
- **C4 containers.** The SPA, the REST API and MongoDB are treated as three containers because each is a separately started or hosted unit by its own `package.json`/runtime. Frontend/backend separation alone would not justify that; the separate start-up scripts and the CORS/HTTP boundary do.
- **Class diagram.** The code has no business classes. Entities are Mongoose models. `User` is one class with a `role` enum (not subclasses), and role-specific fields are tagged `<<applicant>>`, `<<company>>`, `<<admin>>`. Controllers and utilities are shown in a separate companion diagram, `class-diagram-modules.puml`, as `<<module>>` boxes, because combining them with the domain model produced an unreadable web of dependency lines.
- **ERD.** MongoDB enforces no foreign keys. The "FK" markers mean *Mongoose `ref` to another collection*, validated only in application code. The only enforced constraints are unique indexes on `users.email`, `jobs.slug`, and `(job, applicant)` on `applications`.
- **Use case diagram.** `Visitor` is the base actor; authenticated users and the three roles specialise it. `include` / `extend` are used only where one behaviour truly embeds or optionally augments another.
- **API map layout.** The 36 endpoints are grouped into five access-level slices (public, any logged-in role, applicant, company, admin). Each slice contains its frontend callers and endpoints. The diagram is intentionally tall (about 6,800 px); it is meant to be paged or scaled in a report appendix. Per-endpoint error codes are not repeated there; they are in the `sequence-*` diagrams.
- **Deployment.** No Dockerfile, Compose, CI/CD or hosting manifest exists, so hosts are marked unconfirmed. Only local defaults (ports 5000 and 5173) are drawn as facts.
- **No Level 2 DFD** and **no stand-alone authentication-flow diagram** were produced: Level 1 is already readable, and `sequence-authentication.puml` covers authentication and authorization.
- **No registration-approval, notification or background-job diagrams:** the code has none (no scheduler, queue or notification service). OTP e-mail is sent synchronously inside the request.
- **Incomplete / partial features:** `isFlagged` is stored but never enforced; job deadline expiry has no automatic effect on status; there is no application-status ordering.
- **Security representation:** only controls found in code are shown (see `implementation-gaps.md` section C). `ProtectedRoute` is shown as a UX redirect; real authorization is `roleMiddleware` on the server.

## Validation status

Rendered in the authoring environment with tools installed **outside the project** (in a temporary scratch folder; no project file, `package.json` or dependency was touched):

| Tool | Version | Used for |
|---|---|---|
| `@mermaid-js/mermaid-cli` (mmdc) | 11.4.2, headless Chromium | all 9 `.mmd` files |
| PlantUML jar | 1.2024.7, built-in Smetana layout (`-Playout=smetana`; Graphviz was not available) | all 15 `.puml` files |
| Playwright Chromium | preinstalled | SVG to PNG export |

| Check | Result |
|---|---|
| Mermaid syntax | 9 of 9 parsed and rendered with no errors |
| PlantUML syntax | 15 of 15 rendered with no syntax-error output |
| Visual inspection | The rendered images were viewed for overlap, clipping and unreadable labels. **Viewed individually:** three-tier, C4 container / context / backend component / frontend component, use case, class (both), ER, DFD 0 and 1, deployment, API map (top section), both activity diagrams, all three state diagrams, and the apply-job and candidate-status sequence diagrams. **Not viewed as images** (same generator and structure as the viewed sequence diagrams; syntax-checked only): sequence-authentication, sequence-registration, sequence-password-reset, sequence-admin-delete-company. Problems found and fixed: C4 container layout (replaced), `pending` outside its composite state, tangled class diagram (split in two), cluttered DFD Level 1 and overlapping DFD Level 0 labels (rebuilt), an over-tall API map (restructured into slices), fragmented job-posting activity diagram (redrawn), legend wording in the deployment and class diagrams |
| Exports | 24 SVG + 24 PNG in `rendered/` |
| Manual content review | Terminology, endpoint names, status values and counts cross-checked against source (36/36 routes, 3 collections, 6 application statuses, 3 job statuses) |
| Secrets | Only environment variable *names* were read. No values, tokens, URIs or credentials appear in these files |

**Known rendering limitations**
- PlantUML output used the Smetana layout engine, not Graphviz. Layout can differ slightly if re-rendered with Graphviz; content is identical.
- Mermaid SVGs use HTML labels (`foreignObject`). They display correctly in browsers but some desktop tools (e.g. older Word or Inkscape) may drop the text. Use the PNG exports for those targets.
- Only the top section of the API map image was viewed at full resolution; its lower slices are generated by the same template.
- `dfd-level-1.mmd`: a few edges to Cloudinary cross the process column. This is a layout artefact of one shared external service; no labels collide.

## How to preview or render

Mermaid (`.mmd`):
- Paste into <https://mermaid.live>, or use the VS Code extension "Markdown Preview Mermaid Support" / "Mermaid Preview".
- CLI: `npx @mermaid-js/mermaid-cli -i three-tier-architecture.mmd -o rendered/three-tier-architecture.svg` (needs a Chromium; behind a sandbox, pass a puppeteer config with `--no-sandbox`)

PlantUML (`.puml`):
- VS Code extension "PlantUML", or paste into <https://www.plantuml.com/plantuml>.
- CLI (needs Java; Graphviz optional): `java -jar plantuml.jar -tsvg -o rendered *.puml`, or add `-Playout=smetana` if Graphviz is not installed

## Suggested reading order for a report

1. `c4-system-context` -> `c4-container` -> `three-tier-architecture` -> `c4-component`
2. `use-case-diagram`, `entity-relationship-diagram`, `class-diagram`
3. `sequence-*`, `activity-*`, `state-*`
4. `dfd-level-0`, `dfd-level-1`, `deployment-diagram`, `api-interaction-map`
5. `implementation-gaps.md` for limitations
