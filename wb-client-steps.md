# web-client — Sequential Prompts (paste one at a time, in order)

Five prompts. Build and check each one in your browser before moving to the next. **Prompt 2 is the critical one and must NOT be split further** — the login→profile-completion logic broke last time specifically because it was divided across separate generations. Everything else is safe to split.

---

## Prompt 1 — Project setup, base layout, theme

```
Create the foundation of a Spring Boot 4.1.0, Java 25 module named web-client, using Thymeleaf as the server-rendered view layer (no JavaScript framework, no SPA). This module has NO database of its own — it only calls the API Gateway at http://localhost:8080 over HTTP, using a JWT stored server-side in HttpSession.

Include:
- pom.xml: spring-boot-starter-web, spring-boot-starter-thymeleaf, thymeleaf-layout-dialect, lombok
- Main application class
- application.yml on port 8085, property gateway.base-url = http://localhost:8080
- A RestTemplate @Bean with a ClientHttpRequestInterceptor that reads "jwt" from the current HttpSession and attaches it as "Authorization: Bearer <token>" on every outgoing request, except calls to /auth/**
- Base Thymeleaf layout (layout.html, Layout Dialect) styled with Tailwind CSS via CDN, using this theme: colors canvas #F5F7F7, surface #FFFFFF, ink #14322F, brand #0F6B60, brand-dark #0A4C44, accent #E2A33B, status-success #3F8F6F, status-pending #E2A33B, status-danger #B5544A; fonts via Google Fonts — "Fraunces" for headings, "IBM Plex Sans" for body text, "IBM Plex Mono" for times/dates. Rounded-2xl cards, generous padding, a role-aware sidebar/navbar fragment (th:fragment="navbar", branching its links based on a session role attribute — Home/Search/My Appointments for PATIENT, Dashboard/Manage Slots for DOCTOR, Dashboard/Doctors for ADMIN, plus Logout for all), and a footer fragment.
- A separate auth-layout.html for login/register — centered card, no navbar, just the product name in the display font.
- error.html — a themed error page.
- A GlobalControllerAdvice that catches RestClientException and renders error.html instead of a stack trace.

Package base: com.appointment.web
```

---

## Prompt 2 — Authentication + first-login profile completion (DO NOT split this into smaller prompts)

```
Continuing the web-client module (package com.appointment.web). Assume these backend endpoints already exist and work correctly — call them, don't recreate them:
- POST /auth/register — body {name, email, password, role} → 201
- POST /auth/login — body {email, password} → 200 with {token, role, userId}
- GET /patients/{userId} — 200 with the Patient profile if it exists, 404 if it doesn't exist yet
- POST /patients — body {dob, address, medicalHistory, emergencyContact} — no userId field, derived server-side from the JWT
- GET /doctors/by-user/{userId} — 200 with the Doctor profile if it exists, 404 if not
- POST /doctors — body {specialization, qualification, experienceYears, consultationFee} — no userId field

Build the complete authentication and first-login profile-completion flow:

AuthController:
- GET /register — shows register.html (name, email, password, confirmPassword, role dropdown limited to PATIENT/DOCTOR)
- POST /register — calls POST /auth/register, on success redirects to /login with a flash success message, on failure re-renders register.html with the error in a themed banner
- GET /login — shows login.html (email, password)
- POST /login — THIS IS THE CRITICAL LOGIC, implement it exactly as described:
  1. Call POST /auth/login. On failure (401), re-render login.html with an error banner.
  2. On success, store jwt, role, and userId as HttpSession attributes.
  3. Branch by role:
     - PATIENT: call GET /patients/{userId}. If 200, set session attribute profileComplete=true, redirect to /patient/dashboard. If 404 (catch HttpClientErrorException.NotFound specifically — this is an EXPECTED, NORMAL case, not an error to show the user), set profileComplete=false, redirect to /patient/complete-profile.
     - DOCTOR: identical logic using GET /doctors/by-user/{userId}, redirecting to /doctor/dashboard or /doctor/complete-profile.
     - ADMIN: redirect straight to /admin/dashboard (no profile step for admins).
- GET /logout — invalidates the session, redirects to /login

AuthInterceptor (HandlerInterceptor, registered via WebMvcConfigurer):
- Blocks any request to /patient/**, /doctor/**, /admin/** if the session has no jwt attribute — redirect to /login.
- Blocks access to anything under /patient/** EXCEPT /patient/complete-profile if the session's profileComplete is false — redirect to /patient/complete-profile instead. Same rule for /doctor/** (except /doctor/complete-profile). This must work even if the user bookmarks or refreshes the dashboard URL directly, not just right after login.
- Blocks a PATIENT session from reaching any /doctor/** path and vice versa, redirecting to their own dashboard.

PatientProfileController:
- GET /patient/complete-profile — shows patient/complete-profile.html: date of birth (date picker), address, medical history (textarea), emergency contact. No userId field anywhere on this form.
- POST /patient/complete-profile — submits to POST /patients. On success, set session profileComplete=true, redirect to /patient/dashboard with a success flash message. On failure, re-render with inline field errors.

DoctorProfileController:
- GET /doctor/complete-profile — shows doctor/complete-profile.html: specialization dropdown (Cardiology, Dermatology, Orthopedics, ENT, General Physician, Pediatrics, Gynecology, Neurology), qualification, years of experience, consultation fee. No userId field.
- POST /doctor/complete-profile — submits to POST /doctors. On success, set profileComplete=true, redirect to /doctor/dashboard with a success flash message. On failure, re-render with inline errors.

Before finishing, trace through this mentally: register as PATIENT → login → land on complete-profile (NOT a broken/empty dashboard) → submit → land on dashboard. Confirm the 404-means-no-profile-yet case is handled as a normal branch, not an exception page.
```

---

## Prompt 3 — Patient section

```
Continuing the web-client module. The authentication and profile-completion flow from the previous prompt already works — patient/dashboard.html is only ever reached once profileComplete=true in session. Build the rest of the patient experience:

PatientController:
- GET /patient/dashboard — calls GET /appointments/patient/{userId} (userId from session), renders patient/dashboard.html as a list of themed cards (rounded-2xl, shadow-sm), each showing doctor name, date/time, a status pill badge (pending=amber, approved=green, rejected/cancelled=red), and a "Cancel" button on PENDING/APPROVED cards posting to /patient/appointments/{id}/cancel. Show a friendly empty state if there are no appointments yet.
- GET /patient/search-doctors — a specialization dropdown filter form; on submit, calls GET /doctors?specialization=X, renders results as a responsive card grid (name, specialization, experience, fee, a "Book" button linking to /patient/book/{doctorId})
- GET /patient/book/{doctorId} — calls GET /doctors/{doctorId}/slots, renders patient/book-appointment.html with a 3-step stepper header (Search → Select slot → Confirm, step 2 highlighted), available slots as a wrap-grid of rounded-full pill buttons grouped by date (clicking a pill selects it via a small inline script, storing the slotId in a hidden field, highlighting the selected pill bg-brand text-white while others stay outline-only), and a reason textarea
- POST /patient/book/{doctorId} — submits the selected slotId, doctorId, and reason to POST /appointments, on success redirects to /patient/dashboard with a success flash message, on failure (e.g. slot already taken) re-renders with an error banner and a refreshed slot list
- POST /patient/appointments/{id}/cancel — calls PUT /appointments/{id}/status with status=CANCELLED, redirects to /patient/dashboard
```

---

## Prompt 4 — Doctor section

```
Continuing the web-client module. Build the doctor experience, which is only reachable once profileComplete=true in session (already handled by the interceptor from Prompt 2).

DoctorController:
- GET /doctor/dashboard — calls GET /appointments/doctor/{userId}, renders doctor/dashboard.html with requests split into "Pending" (Approve/Reject buttons) and "Upcoming/History" sections, each as a themed card with a status pill
- POST /doctor/appointments/{id}/approve — PUT /appointments/{id}/status status=APPROVED, redirect back with a flash message
- POST /doctor/appointments/{id}/reject — PUT /appointments/{id}/status status=REJECTED, redirect back with a flash message
- GET /doctor/manage-slots — calls GET /doctors/{doctorId}/slots, lists existing slots as pill badges grouped by date (booked ones grayed out/disabled), plus an "Add Slot" form
- POST /doctor/manage-slots — validates and submits a new slot to POST /doctors/{doctorId}/slots (body {date, startTime, endTime} in 24-hour HH:mm format), redirects back with a success flash message

SLOT CREATION FORM — get these exactly right:
- Date field: <input type="date"> with th:min="${T(java.time.LocalDate).now()}" so the picker itself won't show past dates. ALSO validate server-side in the controller that the submitted date isn't before today — reject with a clear inline error if the browser control is bypassed.
- Time fields: do NOT use a native <input type="time"> (its AM/PM vs 24-hour display is inconsistent across browsers). Instead, build each time field from three dropdowns: hour (1–12), minute (00/15/30/45), AM/PM. Before submitting, convert both start and end time from 12-hour+AM/PM into 24-hour "HH:mm" strings (e.g. 2:30 PM → "14:30") via a small inline JavaScript function, placing the converted values into hidden fields that are what actually get submitted.
- Validate, both client-side and server-side, that the computed end time is after the computed start time — show a clear inline error otherwise.
```

---

## Prompt 5 — Admin section (build this last)

```
Continuing the web-client module. Build the admin section, read-only for this scope.

AdminController:
- GET /admin/dashboard — calls GET /doctors (no filter), renders admin/dashboard.html with a themed stat card row (total doctors, total pending/approved/completed appointments — aggregate client-side from available data) and a table of all doctors below.
```

---

## After all five prompts

Walk through both full user journeys yourself in the browser:
- **Patient**: register → login → land on complete-profile (not a broken dashboard) → submit → empty dashboard → search doctors → book → appears as PENDING on dashboard
- **Doctor**: register → login → complete-profile → dashboard → add a slot with today-or-later date and a valid AM/PM range → confirm a patient can find and book it

If Prompt 2's flow is right, Prompts 3–5 rarely cause the kind of breakage you saw before — they only add pages *on top of* an already-correct foundation instead of redefining how login and routing work.
