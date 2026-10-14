# Implementation Gaps and Observations

These are **observations from reading the code**, not changes made to it. Each was checked against the relevant code path. Severity labels are the author's judgement. Nothing here was run against a live system.

Paths use the `Server/` and `Client/src/` convention explained in `traceability.md`.

## A. Frontend / backend alignment

| ID | Finding | Result |
|---|---|---|
| A1 | Frontend calls with no matching backend route | **None found.** All 36 RTK Query endpoints map to a route (see `api-interaction-map.mmd`, `traceability.md` section 4). |
| A2 | Backend endpoints with no frontend caller | **None found.** |
| A3 | `UpdateJob` accepts a `status` field, but `PostJobModal.jsx` never sends one | Capability exists only for direct API callers. Relevant to B1. |
| A4 | Client route is `/forgetPassword`, backend endpoint is `/auth/forgotPassword` | Not a mismatch (different layers), but easy to confuse. |
| A5 | `Contact.jsx` has no backend interaction (static `mailto:` link) | No contact endpoint exists; not a defect. |
| A6 | The client matches the login error text `"not verified"` to start the resend-OTP flow (`Login.jsx` L76-84) | Behaviour depends on message wording, not an error code. If the server text changes, the flow silently stops working. |
| A7 | `login` returns the JWT in the JSON body as well as the cookie; the client never uses `accessToken` from the body | Unused data in the response. |
| A8 | `react-router` and `react-router-dom` are both in `Client/package.json`; all 21 router imports use `react-router` | `react-router-dom` appears unused. |
| A9 | `mongodb` is a dependency of the server but nothing requires it (Mongoose is used) | Appears unused. |

## B. Authorization and business rules

| ID | Finding | Evidence | Impact |
|---|---|---|---|
| B1 | A company can reopen a job the admin closed. `TogglePauseJob` rejects `closed`, but `UpdateJob` accepts `status:"active"` without checking the current status. | `companyController.js` L84-193 vs L302-350 | Admin moderation can be bypassed by a direct API call (not by the current UI). **Medium** |
| B2 | `isFlagged` is never enforced. Flagged applicants can still log in and apply. | repo-wide search: only written/read in `adminController.js` and the model | The "flag" feature is cosmetic. **Low/Medium (feature incomplete)** |
| B3 | Application status transitions are unrestricted: any of six values from any value, including `rejected` back to `pending`, or `pending` straight to `approved`. | `UpdateApplicationStatus` L411-486 | May be intentional; stated in `state-application-status.puml`. |
| B4 | Deadline expiry never changes `Job.status`; `GET /jobs` and `/jobs/latest` list every `active` job regardless of deadline. `ApplyJob` rejects late applications. | `jobController.js` L12-75; `applicantController.js` | Expired jobs stay visible and show as open until someone changes the status. **Low** |
| B5 | `GET /jobs/:slug` returns jobs of any status (paused, closed) while list endpoints return only active. | `jobController.js` L76-103 | Paused/closed jobs remain reachable by URL. May be intended. **Low** |
| B6 | Editing a job re-validates `deadline` as a future date. `PostJobModal` always sends the deadline, so editing a job whose deadline has passed forces choosing a new future date. | `UpdateJob` deadline block; `PostJobModal.jsx` L22-60 | Usability constraint. **Low** |
| B7 | `POST /auth/logout` is not behind `authMiddleware`. | `auth.js` L16 | Harmless (clears a cookie), but inconsistent with other auth routes. **Info** |

Verified positives: every role-restricted router applies `authMiddleware` and `roleMiddleware` as `router.use(...)` before its handlers; company and applicant queries scope by `req.user._id` (`company`/`applicant` filters) so users cannot act on other users' records; admin-only deletions check `role`.

## C. Security-related observations

| ID | Finding | Evidence |
|---|---|---|
| C1 | No limit on OTP attempts and no rate limiting on login, register, OTP verify/resend, or forgot-password. A 6-digit code valid for 10 minutes could be guessed by repeated requests. | `authController.js`; no rate-limit package in `Server/package.json` |
| C2 | Account enumeration: `login` returns "User not found" for unknown e-mails; `forgotPassword` and `verifyEmail` return 404 for unknown e-mails; `register` returns 409 for existing ones. | `authController.js` |
| C3 | Cookie has `httpOnly` and `sameSite:"lax"` but no `secure` flag. | `authController.js` login |
| C4 | No security-headers middleware (e.g. helmet) and no request logging library in dependencies. | `Server/package.json` |
| C5 | `generateOTP` uses `crypto.randomInt(100000, 999999)`, so `999999` can never be generated. Negligible. | `utils/helpers.js` |
| C6 | Pagination `limit` has no upper bound on public and admin list endpoints. | `jobController.js` L12-20; `adminController.js` `getPage` L16-21 |
| C7 | Registration creates the user *before* sending the e-mail. If SMTP fails, the API returns 500 but the unverified account exists; retry returns 409. The client recovers only via the login "not verified" path (A6). | `authController.js` L21-121 |
| C8 | JWT sessions have no refresh or revocation list. Deleted users are rejected because `authMiddleware` reloads the user on each request. | `authMiddleware.js` |

Controls that **are** present (verified): bcrypt password hashing; strong-password regex; e-mail validation; ObjectId validation before queries; regex escaping in admin search; file type and size limits (5 MB); CORS restricted to `CLIENT_URL` with credentials; `.env` excluded by `Server/.gitignore`.

## D. Data integrity and consistency

| ID | Finding | Evidence |
|---|---|---|
| D1 | References between collections are Mongoose `ref`s only; no database foreign keys or transactions exist. Deleting a user/company/job relies on manual cascades (`deleteApplications`, `deleteMany`). A failure midway can leave orphans. | `adminController.js` L27-34, L275-307, L421-455, L548-end; `companyController.js` DeleteJob |
| D2 | `ApplyJob` uploads the resume to Cloudinary before `Application.create`. If creation fails, the uploaded file stays in Cloudinary. | `applicantController.js` L14-99 |
| D3 | A concurrent duplicate apply would hit the unique index and surface as a generic 500 (the code only returns 409 from its pre-check). | `Application.js` L53; `ApplyJob` catch block |
| D4 | `Application.company` duplicates `Job.company`. Consistent today because it is set at creation and jobs cannot change owner. | `applicantController.js`; `Application.js` |
| D5 | Free-text `Job.type`, `mode`, `level`, `salary`, `department` have no enum or format validation. | `Job.js` |
| D6 | The OTP e-mail template says "valid for 5 minutes", but the code sets a 10-minute expiry everywhere. | `utils/mail/mailTemplate.js` vs `authController.js` |
| D7 | `Job.moderatedBy` is written by CloseJob but returned only in the close response; it is not used elsewhere. | `adminController.js` CloseJob |

## E. Operations and unverifiable items

| ID | Finding |
|---|---|
| E1 | No Dockerfile, Compose file, CI/CD workflow, hosting manifest or deployment documentation in either archive. Deployment topology is Not Confirmed (see `deployment-diagram.puml`). |
| E2 | No automated tests and no `test` script in either `package.json`; the client has a `lint` script. |
| E3 | `npm start` and `npm run dev` both run `nodemon`, and `nodemon` is listed under `dependencies`. |
| E4 | `server.js` forces DNS servers `8.8.8.8` / `1.1.1.1` at startup (also in `seed.js`), a hard-coded environment assumption. |
| E5 | The Cloudinary connection check (`api.ping`) only logs; the server starts regardless of Cloudinary or DB connection failure. |
| E6 | `.env` files were present in both archives. Only variable *names* were read into this documentation. Rotate any real credentials if the archives were shared. |
