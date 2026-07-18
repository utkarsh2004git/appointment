# web-client — Single Complete Prompt (GitHub Copilot / Amazon Q)

Paste this whole block as ONE message. It's intentionally not split into multiple prompts — the profile-completion flow (first login → fill profile → dashboard) is cross-cutting logic that breaks when generated across separate, disconnected prompts, which is what caused the earlier broken login flow.

---

```
Create a COMPLETE, fully working Spring Boot 4.1.0, Java 25 module named web-client, using Thymeleaf as the server-rendered view layer (no JavaScript framework, no SPA). This module has NO database of its own — it only calls the API Gateway at http://localhost:8080 over HTTP for all data, using a JWT stored server-side in HttpSession.

BACKEND CONTEXT YOU MUST ASSUME ALREADY EXISTS (do not regenerate these, just call them correctly):
- POST /auth/register — body {name, email, password, role} → 201
- POST /auth/login — body {email, password} → 200 with {token, role, userId}
- GET /patients/{userId} — returns 200 with the Patient profile if one exists, or 404 if it does not exist yet
- POST /patients — body {dob, address, medicalHistory, emergencyContact} — NO userId field, it's derived server-side from the JWT — creates a Patient profile for the currently authenticated user
- GET /doctors/by-user/{userId} — returns 200 with the Doctor profile if one exists, or 404 if not
- POST /doctors — body {specialization, qualification, experienceYears, consultationFee} — NO userId field — creates a Doctor profile for the currently authenticated user
- GET /doctors?specialization= — search doctors
- GET /doctors/{id}/slots — list a doctor's available (unbooked) slots
- POST /doctors/{id}/slots — body {date, startTime, endTime} in 24-hour HH:mm format — adds a slot
- POST /appointments — body {doctorId, slotId, reason} → books an appointment for the current patient
- PUT /appointments/{id}/status — body {status} → approve/reject/cancel
- GET /appointments/patient/{userId} and GET /appointments/doctor/{userId} — list appointments

PROJECT SETUP:
- pom.xml: spring-boot-starter-web, spring-boot-starter-thymeleaf, thymeleaf-layout-dialect, lombok
- application.yml on port 8085, property gateway.base-url = http://localhost:8080
- A RestTemplate @Bean with a ClientHttpRequestInterceptor that reads "jwt" from the current HttpSession and attaches it as "Authorization: Bearer <token>" on every outgoing request, except calls to /auth/**
- Base Thymeleaf layout (layout.html, Layout Dialect) styled with Tailwind CSS via CDN, using this theme: colors canvas #F5F7F7, surface #FFFFFF, ink #14322F, brand #0F6B60, brand-dark #0A4C44, accent #E2A33B, status-success #3F8F6F, status-pending #E2A33B, status-danger #B5544A; fonts via Google Fonts — "Fraunces" for headings, "IBM Plex Sans" for body text, "IBM Plex Mono" for times/dates. Rounded-2xl cards, generous padding, a role-aware sidebar/navbar fragment, and a separate centered auth-layout.html for login/register with no navbar.
- A GlobalControllerAdvice that catches RestClientException and renders a themed error.html instead of a stack trace.

===========================================================
AUTHENTICATION, DASHBOARD ROUTING, AND FIRST-LOGIN PROFILE COMPLETION
(this is the most important part — get this flow exactly right)
===========================================================

AuthController:
- GET /register — shows register.html (name, email, password, confirmPassword, role dropdown limited to PATIENT/DOCTOR)
- POST /register — calls POST /auth/register, on success redirects to /login with a flash success message, on failure (400/409) re-renders register.html with the error shown in a themed banner
- GET /login — shows login.html (email, password)
- POST /login — this is the critical logic:
  1. Call POST /auth/login. On failure (401), re-render login.html with an error banner.
  2. On success, store jwt, role, and userId as HttpSession attributes.
  3. Branch by role:
     - If role is PATIENT: call GET /patients/{userId}. If it returns 200, set session attribute profileComplete=true and redirect to /patient/dashboard. If it returns 404 (catch this specific exception — RestTemplate throws HttpClientErrorException.NotFound for a 404, this is expected and normal, not an error to show the user), set profileComplete=false and redirect to /patient/complete-profile.
     - If role is DOCTOR: same logic but call GET /doctors/by-user/{userId}, redirecting to /doctor/dashboard or /doctor/complete-profile.
     - If role is ADMIN: redirect straight to /admin/dashboard (admins have no profile step).
- GET /logout — invalidates the session, redirects to /login

AuthInterceptor (HandlerInterceptor, registered via WebMvcConfigurer):
- Blocks any request to /patient/**, /doctor/**, /admin/** if the session has no jwt attribute — redirect to /login.
- Additionally blocks access to /patient/dashboard and everything else under /patient/** EXCEPT /patient/complete-profile if the session's profileComplete attribute is false — redirect to /patient/complete-profile instead. Apply the same rule for /doctor/dashboard and /doctor/** (except /doctor/complete-profile) using the doctor equivalent. This is what stops someone from reaching a broken/empty dashboard before finishing their profile, including if they bookmark or refresh the dashboard URL directly.
- Also blocks a PATIENT session from reaching any /doctor/** path and vice versa, redirecting to their own dashboard instead.

PatientProfileController:
- GET /patient/complete-profile — shows patient/complete-profile.html, a form with: date of birth (date picker), address, medical history (textarea), emergency contact. Do NOT include a userId field anywhere in this form — there is none to fill in.
- POST /patient/complete-profile — submits the form to POST /patients on the gateway. On success, set the session's profileComplete=true and redirect to /patient/dashboard with a success flash message. On failure (e.g. validation error), re-render the form with the error shown inline per field.

DoctorProfileController:
- GET /doctor/complete-profile — shows doctor/complete-profile.html, a form with: specialization (dropdown: Cardiology, Dermatology, Orthopedics, ENT, General Physician, Pediatrics, Gynecology, Neurology), qualification, years of experience, consultation fee. No userId field.
- POST /doctor/complete-profile — submits to POST /doctors on the gateway. On success, set profileComplete=true, redirect to /doctor/dashboard with a success flash message. On failure, re-render with inline errors.

===========================================================
PATIENT SECTION (only reachable once profileComplete=true)
===========================================================

PatientController:
- GET /patient/dashboard — calls GET /appointments/patient/{userId}, renders patient/dashboard.html as a list of themed cards (rounded-2xl, shadow-sm), each showing doctor name, date/time, a status pill badge colored by status (pending=amber, approved=green, rejected/cancelled=red), and a "Cancel" button on PENDING/APPROVED cards posting to /patient/appointments/{id}/cancel
- GET /patient/search-doctors — shows a specialization dropdown filter; on submit, calls GET /doctors?specialization=X and renders results as a responsive card grid (name, specialization, experience, fee, a "Book" button linking to /patient/book/{doctorId})
- GET /patient/book/{doctorId} — calls GET /doctors/{doctorId}/slots, renders patient/book-appointment.html with a 3-step stepper header (Search → Select slot → Confirm, step 2 highlighted), available slots shown as a wrap-grid of rounded-full pill buttons grouped by date (clicking a pill selects it, storing the chosen slotId in a hidden form field via a small inline script, visually highlighting the selected pill with bg-brand text-white while unselected pills stay outline-only), and a reason textarea below
- POST /patient/book/{doctorId} — submits the selected slotId and reason to POST /appointments with doctorId included, on success redirects to /patient/dashboard with a success flash message, on failure (e.g. slot already taken) re-renders the page with an error banner and refreshed slot list
- POST /patient/appointments/{id}/cancel — calls PUT /appointments/{id}/status with status=CANCELLED, redirects back to /patient/dashboard

===========================================================
DOCTOR SECTION (only reachable once profileComplete=true)
===========================================================

DoctorController:
- GET /doctor/dashboard — calls GET /appointments/doctor/{userId}, renders doctor/dashboard.html with requests split into a "Pending" section (Approve/Reject buttons) and an "Upcoming/History" section, each appointment as a themed card with a status pill
- POST /doctor/appointments/{id}/approve — PUT /appointments/{id}/status status=APPROVED, redirect back with a flash message
- POST /doctor/appointments/{id}/reject — PUT /appointments/{id}/status status=REJECTED, redirect back with a flash message
- GET /doctor/manage-slots — calls GET /doctors/{doctorId}/slots, lists existing slots as pill badges grouped by date (booked ones shown grayed out and disabled), plus an "Add Slot" form
- POST /doctor/manage-slots — validates and submits a new slot to POST /doctors/{doctorId}/slots, redirects back with a success flash message

SLOT CREATION FORM — SPECIFIC CONSTRAINTS (get these exactly right, this is a common bug spot):
- Date field: an HTML <input type="date"> with its min attribute set server-side via Thymeleaf to today's date (th:min="${T(java.time.LocalDate).now()}") so the date picker itself refuses to show past dates. ALSO validate server-side in the controller (not just the browser) that the submitted date is not before today, rejecting with a clear inline error if someone bypasses the browser control.
- Time fields (start time and end time): do NOT use a native <input type="time">, since its AM/PM vs 24-hour display is inconsistent across browsers/locales. Instead build each time field from three explicit dropdowns: hour (1–12), minute (00/15/30/45), and an AM/PM select. Before submitting the form, convert both start and end time from 12-hour+AM/PM into 24-hour "HH:mm" strings (e.g. 2:30 PM → "14:30") using a small inline JavaScript function, and put the converted values into hidden form fields that actually get submitted — the visible dropdowns are just for the doctor's convenience, the hidden 24-hour values are what the API receives.
- Validate (both client-side before submit, and again server-side in the controller as a backstop) that the computed end time is after the computed start time — show an inline error if not, don't just silently reject.

===========================================================
ADMIN SECTION (build this LAST, only after everything above is fully working)
===========================================================

AdminController:
- GET /admin/dashboard — calls GET /doctors (no filter), renders admin/dashboard.html with a themed stat card row (total doctors, total pending/approved/completed appointments — aggregate client-side from available data for this scope) and a table of all doctors below. Read-only for now.

===========================================================
FINAL REQUIREMENTS
===========================================================
- Every form must show field-level validation errors inline when the backend returns a 400.
- Every RestTemplate call that can 404/400/500 must be wrapped so it never surfaces a raw stack trace to the browser — always a themed error state instead.
- Test the entire flow yourself mentally before finishing: register as PATIENT → login → land on complete-profile (not a broken dashboard) → submit profile → land on dashboard (empty state, no appointments yet) → search doctors → book → confirm it appears on the dashboard as PENDING. Do the same for DOCTOR in a second pass: register → login → complete-profile → dashboard → add a slot with today-or-later date and a valid AM/PM time range → confirm it's visible to a patient searching that specialization.

Package base: com.appointment.web
```

---

## Why this is one prompt, not several

The earlier 6-sub-prompt breakdown broke specifically because the "does a profile exist yet?" check is cross-cutting — it touches login, session state, routing, and two separate controllers at once. Splitting that logic across disconnected prompts is exactly how it fell apart. Keeping the whole `POST /login` branching logic — and the interceptor rules that depend on it — in one prompt is what makes it actually hang together.

## The one thing worth double-checking after Copilot generates this

Confirm it correctly catches `HttpClientErrorException.NotFound` around the profile-existence checks (`GET /patients/{userId}` and `GET /doctors/by-user/{userId}`) rather than treating a 404 there as a fatal error. "A 404 here just means go show the profile form" is a slightly unusual instruction to follow correctly, and it's the single most likely spot for something to still be wrong even with the consolidated prompt.
