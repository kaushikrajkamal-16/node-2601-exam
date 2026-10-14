# Traceability Matrix

Maps diagram elements to the source that supports them.

**Path convention.** `Server/...` and `Client/src/...` are paths inside the two uploaded archives (`Server.zip`, `Client.zip`). Those sources are **not committed in this repository** (it only contains this documentation), so line numbers refer to the uploaded copies.

**Evidence status**
- **Verified**: read directly in code or configuration.
- **Inferred**: concluded from code but not stated by it.
- **Partially Implemented**: exists but incomplete or unenforced.
- **Planned**: no instance found in the repository.
- **Not Confirmed**: the repository does not allow a determination.

No secret values were read into this document. Only the *names* of environment variables are cited.

## 1. Architecture and containers

| Element | Diagram file(s) | Source | Responsible code | Status |
|---|---|---|---|---|
| React SPA (React 19, Vite, Tailwind 4, React Router, RTK Query) | three-tier, c4-container, c4-component-frontend, deployment | `Client/package.json`; `Client/src/main.jsx`; `Client/vite.config.js` | dependencies and `createRoot` + `Provider` + `BrowserRouter` | Verified |
| Express 5 REST API | three-tier, c4-container, c4-component | `Server/package.json`; `Server/server.js` L1-36 | `express()`, `app.use(...)`, `app.listen(PORT)` | Verified |
| API port default 5000, route prefix `BASE_URL` (`/api/v1`) | c4-container, deployment, api-map | `Server/server.js` L36; `Server/routes/index.js` L7-9; `Client/src/redux/baseApi.js` L7 | `process.env.PORT \|\| 5000`; `router.use(process.env.BASE_URL, apiRoute)`; client default `http://localhost:5000/api/v1` | Verified (the value of `BASE_URL` is stated in the repo's own `.env` comment and the client default) |
| CORS with credentials from `CLIENT_URL` (default `http://localhost:5173`) | three-tier, deployment | `Server/server.js` L12-17 | `cors({origin, credentials:true})` | Verified |
| Cookie-based session (httpOnly `accessToken`) | all auth diagrams | `Server/controllers/authController.js` L218-297; `Server/middleware/authMiddleware.js` | `res.cookie("accessToken", ..., {httpOnly, sameSite:"lax", maxAge:1d})`; `req.cookies.accessToken` | Verified |
| MongoDB through Mongoose | three-tier, c4-container, ERD, deployment | `Server/configuration/dbConnection.js`; `Server/model/*.js` | `mongoose.connect(process.env.MONGO_URI)` | Verified (technology); host location **Not Confirmed** |
| Cloudinary integration | c4-*, DFD, deployment | `Server/configuration/cloudinary.js`; `Server/utils/uploadToCloudinary.js` | `upload_stream`, `uploader.destroy` | Verified |
| Gmail SMTP via Nodemailer | c4-*, DFD, deployment | `Server/utils/mail/mailService.js` L1-30 | `createTransport({host:"smtp.gmail.com", port:587, secure:false})` | Verified |
| Global error handler (multer errors to HTTP 400) | three-tier, sequence-apply-job | `Server/server.js` L26-34 | `app.use((err, req, res, next) => ...)` | Verified |
| Hard-coded DNS servers 8.8.8.8 / 1.1.1.1 | deployment note | `Server/server.js` L1; `Server/utils/seed.js` L4 | `dns.setServers([...])` | Verified (that it exists); the reason is **Inferred** |
| Separate physical tiers / hosts | three-tier, deployment | none | none | **Not Confirmed** |
| Dockerfile, Compose, CI/CD, hosting manifests, tests | deployment, README | searched both archives | none found | **Not Confirmed** (absence) |

## 2. Actors and use cases

| Element | Diagram | Source | Responsible code | Status |
|---|---|---|---|---|
| Roles applicant / company / admin | use-case, class | `Server/model/User.js` L18-22 | `role` enum | Verified |
| Visitor (unauthenticated) | use-case | `Server/routes/api/job.js`; `auth.js` L8-16 (routes without `authMiddleware`) | public endpoints | Verified |
| Admin only via seed or by another admin (no self-registration) | use-case, state-user-account | `Server/controllers/authController.js` Registration (role check); `Server/utils/seed.js` | role whitelist `["applicant","company"]`; `userSchema.create({role:"admin"})` | Verified |
| Register, verify OTP, resend OTP | use-case, sequence-registration | `authController.js` Registration L21, VerifyEmail L123, ResendOtp L310 | same | Verified |
| Login / Logout / Get own profile | use-case, sequence-authentication | `authController.js` login L218, Logout L299, GetMe L517 | same | Verified |
| Forgot / reset password | use-case, sequence-password-reset | `authController.js` ForgotPassword L372, ResetPassword L429 | same | Verified |
| Edit profile + photo (role-specific fields) | use-case | `authController.js` UpdateProfile L556; `allowedFields` map | per-role field whitelist | Verified |
| Apply, view applications, withdraw | use-case, sequence-apply-job, activity-apply-for-job | `applicantController.js` ApplyJob L14, GetMyApplications L101, WithdrawApplication L152 | same | Verified |
| Post/edit/list/pause/delete job | use-case, activity-job-posting | `companyController.js` CreateJob L16, UpdateJob L84, DeleteJob L195, GetMyJobs L244, TogglePauseJob L302 | same | Verified |
| View candidates, update status, add note, remove | use-case, sequence-candidate-status | `companyController.js` GetCandidates L352, UpdateApplicationStatus L411, DeleteCandidate L488; `Client/.../RecruiterDashboard.jsx` L203-216 (prompt for note on exam/interview) | same | Verified |
| Admin stats, users list | use-case | `adminController.js` GetStats L51, GetUsers L93 | same | Verified |
| Admin applicants: list / add / flag / delete | use-case | `adminController.js` L137, L188, L240, L275 | same | Verified (flag is stored only; see gaps G2, **Partially Implemented**) |
| Admin companies: list / add / delete | use-case, sequence-admin-delete-company | `adminController.js` L309, L369, L421 | same | Verified |
| Admin jobs: list / close / delete | use-case, sequence-admin-delete-company | `adminController.js` L457, L512, L548 | same | Verified |
| Cascade deletion (applications and resume files) | use-case, DFD | `adminController.js` `deleteApplications` L27-34; `companyController.js` DeleteJob L195-243 | `deleteFromCloudinary` loop + `deleteMany` | Verified |
| Public job browse / latest / detail / companies | use-case, DFD | `jobController.js` L12, L50, L76, L105 | same | Verified |
| Contact page | (not drawn as a use case) | `Client/src/pages/Contact.jsx` | static page, `mailto:` link only, no API call | Verified (no backend interaction) |

## 3. Routes and middleware

| Route group | Mount | Guard | Source | Status |
|---|---|---|---|---|
| auth | `/auth` | none for register/verifyEmail/resendOtp/login/forgotPassword/resetpassword/logout; `authMiddleware` for getme and updateprofile | `Server/routes/api/auth.js` L8-23; `routes/api/index.js` L9 | Verified |
| applicant | `/applicant` | `router.use(authMiddleware, roleMiddleware("applicant"))` | `routes/api/applicant.js` L9 | Verified |
| company | `/company` | `authMiddleware`, `roleMiddleware("company")` | `routes/api/company.js` L8 | Verified |
| admin | `/admin` | `authMiddleware`, `roleMiddleware("admin")` | `routes/api/admin.js` L8 | Verified |
| public jobs/companies | `/` (mounted last) | none | `routes/api/job.js` L6-9; `index.js` L13 | Verified |
| `/jobs/latest` declared before `/jobs/:slug` | api-map | route order | `job.js` L6-8 | Verified |
| Resume upload: pdf/doc/docx, 5 MB, memory storage | sequence-apply-job | `Server/middleware/upload.js` | `uploadResume` | Verified |
| Image upload: png/jpeg/webp, 5 MB | three-tier, api-map | `upload.js` | `uploadImage` | Verified |
| Frontend route guard | sequence-authentication | `Client/src/Components/ProtectedRoute.jsx` | `useGetMeQuery` + `<Navigate>` | Verified (UX only; server authorization is separate) |

## 4. Endpoint inventory (36 routes)

Frontend callers are the RTK Query definitions in `Client/src/redux/*.js` and the components named in `api-interaction-map.mmd`.

| # | Method and path | Handler | Guard | Frontend hook | Status |
|---|---|---|---|---|---|
| 1 | POST /auth/register | authController.Registration | public | useRegisterMutation | Verified |
| 2 | POST /auth/verifyEmail | authController.VerifyEmail | public | useVerifyEmailMutation | Verified |
| 3 | POST /auth/resendOtp | authController.ResendOtp | public | useResendOtpMutation | Verified |
| 4 | POST /auth/login | authController.login | public | useLoginMutation | Verified |
| 5 | POST /auth/forgotPassword | authController.ForgotPassword | public | useForgotPasswordMutation | Verified |
| 6 | POST /auth/resetpassword | authController.ResetPassword | public | useResetPasswordMutation | Verified |
| 7 | POST /auth/logout | authController.Logout | none | useLogoutMutation | Verified |
| 8 | GET /auth/getme | authController.GetMe | auth | useGetMeQuery | Verified |
| 9 | PUT /auth/updateprofile | authController.UpdateProfile | auth + uploadImage | useUpdateProfileMutation | Verified |
| 10 | POST /applicant/apply/:jobId | applicantController.ApplyJob | applicant + uploadResume | useApplyJobMutation | Verified |
| 11 | GET /applicant/applications | applicantController.GetMyApplications | applicant | useGetMyApplicationsQuery | Verified |
| 12 | DELETE /applicant/applications/withdraw/:id | applicantController.WithdrawApplication | applicant | useWithdrawApplicationMutation | Verified |
| 13 | GET /company/jobs | companyController.GetMyJobs | company | useGetMyJobsQuery | Verified |
| 14 | POST /company/jobs/create | companyController.CreateJob | company | useCreateJobMutation | Verified |
| 15 | PUT /company/jobs/update/:id | companyController.UpdateJob | company | useUpdateJobMutation | Verified |
| 16 | PATCH /company/jobs/pause/:id | companyController.TogglePauseJob | company | useTogglePauseJobMutation | Verified |
| 17 | DELETE /company/jobs/delete/:id | companyController.DeleteJob | company | useDeleteJobMutation | Verified |
| 18 | GET /company/candidates | companyController.GetCandidates | company | useGetCandidatesQuery | Verified |
| 19 | PATCH /company/candidates/status/:id | companyController.UpdateApplicationStatus | company | useUpdateCandidateStatusMutation | Verified |
| 20 | DELETE /company/candidates/delete/:id | companyController.DeleteCandidate | company | useDeleteCandidateMutation | Verified |
| 21 | GET /admin/stats | adminController.GetStats | admin | useGetStatsQuery | Verified |
| 22 | GET /admin/users | adminController.GetUsers | admin | useGetUsersQuery | Verified |
| 23 | GET /admin/applicants | adminController.GetApplicants | admin | useGetApplicantsQuery | Verified |
| 24 | POST /admin/applicants/add | adminController.AddApplicant | admin | useAddApplicantMutation | Verified |
| 25 | PATCH /admin/applicants/flag/:id | adminController.ToggleFlagApplicant | admin | useToggleFlagApplicantMutation | Verified |
| 26 | DELETE /admin/applicants/delete/:id | adminController.DeleteApplicant | admin | useDeleteApplicantMutation | Verified |
| 27 | GET /admin/companies | adminController.GetCompanies | admin | useGetAdminCompaniesQuery | Verified |
| 28 | POST /admin/companies/add | adminController.AddCompany | admin | useAddCompanyMutation | Verified |
| 29 | DELETE /admin/companies/delete/:id | adminController.DeleteCompany | admin | useDeleteCompanyMutation | Verified |
| 30 | GET /admin/jobs | adminController.GetAllJobs | admin | useGetAdminJobsQuery | Verified |
| 31 | PATCH /admin/jobs/close/:id | adminController.CloseJob | admin | useCloseJobMutation | Verified |
| 32 | DELETE /admin/jobs/delete/:id | adminController.DeleteJob | admin | useAdminDeleteJobMutation | Verified |
| 33 | GET /jobs | jobController.GetJobs | public | useGetJobsQuery | Verified |
| 34 | GET /jobs/latest | jobController.GetLatestJobs | public | useGetLatestJobsQuery | Verified |
| 35 | GET /jobs/:slug | jobController.GetJobBySlug | public | useGetJobBySlugQuery | Verified |
| 36 | GET /companies | jobController.GetCompanies | public | useGetCompaniesQuery | Verified |

## 5. Data model

| Element | Diagram | Source | Evidence | Status |
|---|---|---|---|---|
| Collection `users` (single model for all roles) | ERD, class | `Server/model/User.js` L5-60 | `mongoose.model("User")` | Verified |
| `User.email` unique | ERD | `User.js` L13 | `unique:true` (creates a unique index) | Verified |
| `User.addedBy` to `User` | ERD, class | `User.js` L52 | `ObjectId ref "User"` | Verified as schema ref; **not DB-enforced** |
| Collection `jobs`, `Job.slug` unique | ERD | `Server/model/Job.js` L9, L22 | `unique:true` | Verified |
| `Job.company`, `Job.moderatedBy` to `User` | ERD | `Job.js` L11-16, L71 | `ref:"User"` | Verified as schema ref; not DB-enforced |
| `Job.status` enum | state-job-status, class | `Job.js` L64-68 | `["active","paused","closed"]` | Verified |
| Collection `applications` | ERD | `Server/model/Application.js` L6-55 | schema | Verified |
| `Application.job / applicant / company` refs | ERD | `Application.js` L9-22 | `ref` | Verified as schema ref; `company` is a **denormalised copy** of `Job.company` (set in ApplyJob) |
| Unique pair (job, applicant) | ERD, class | `Application.js` L53 | `index({job:1, applicant:1},{unique:true})` | Verified (the only compound index) |
| `Application.status` enum | state-application-status | `Application.js` L32-36 | six values | Verified |
| `statusHistory` embedded array | ERD, class | `Application.js` L42-47 | `[{status, changedAt}]` | Verified |
| Password hashing on save (bcrypt, 10 rounds) | class | `User.js` L59-64 | `pre("save")`, `ComparePassword` | Verified |
| Real foreign keys / referential integrity | ERD | none | MongoDB has none; cascades are manual code | **Not applicable** by design (stated in diagram title) |
| Other indexes | ERD | grep for `index(` and `unique` | none besides the three above | Verified |

## 6. State machines and workflows

| Element | Diagram | Source | Status |
|---|---|---|---|
| Application starts `pending` | state-application-status | `applicantController.js` ApplyJob (`status:"pending"`) | Verified |
| Any-to-any status change by company | state-application-status | `companyController.js` UpdateApplicationStatus L411-486 (only checks membership in the allowed list) | Verified (absence of ordering) |
| Withdraw blocked when `approved` | state-application-status | `applicantController.js` WithdrawApplication L152-205 | Verified |
| Job `active/paused` toggle; closed guarded in TogglePauseJob only | state-job-status | `companyController.js` L302-350 | Verified |
| UpdateJob allows `status:"active"` for closed job | state-job-status | `companyController.js` UpdateJob L84-193 (status block checks only the value) | Verified (see gap G1) |
| Admin close sets `closed` + `moderatedBy` | state-job-status | `adminController.js` CloseJob L512-546 | Verified |
| Deadline expiry does not change status | state-job-status, activity | `applicantController.js` ApplyJob (date check); no scheduler found | Verified (no job runner in the code or dependencies) |
| Account verified via OTP, 10-minute expiry | state-user-account | `authController.js` L21-217, L310-370 | Verified |
| Reset sets `isVerified = true` | state-user-account | `authController.js` ResetPassword | Verified |
| `isFlagged` toggled but never read elsewhere | state-user-account | `adminController.js` L240-273; repository-wide search | Verified (**Partially Implemented**) |
| Interview note prompt for exam/interview | use-case, sequence-candidate-status | `RecruiterDashboard.jsx` L203-216 | Verified |

## 7. Statements deliberately left Not Confirmed

- Where the API, SPA and MongoDB are hosted; whether they share a machine.
- Production environment variable values and whether HTTPS is terminated by a proxy.
- Whether the database is MongoDB Atlas or self-hosted (only the DNS override hints at a DNS-based connection string).
- Any automated test coverage (no test files or scripts found).
