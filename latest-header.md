# Copilot Master Prompts — One Prompt Per Service

Each block below is ONE prompt for ONE service. Paste the whole block into Copilot Chat with that service's folder open. Review the generated files, build, and test before moving to the next service.

**Important concept to understand before you start:** Within a single service, entities that belong together use real JPA relationships (`@OneToMany`, `@ManyToOne`). Across services (e.g., Appointment referencing a Doctor), there is NO real JPA relationship — you just store the plain ID (`Long doctorId`) because that data lives in a different database entirely. Each prompt below tells Copilot explicitly which case applies, so it doesn't accidentally try to `@ManyToOne` something that lives in another microservice.

---

## 0. Project overview (give this FIRST, once, before any service prompt)

Paste this once at the start of a Copilot Chat session (or repeat it briefly at the top of each new session/account) so Copilot has consistent context for every service you build afterward:

```
I'm building a microservices-based Online Appointment Management System as a college case study project. Please keep the following consistent across every service I ask you to generate in this project:

TECH STACK & VERSIONS:
- Java 25
- Spring Boot 4.1.0
- Spring Cloud 2025.1.2 (the release train compatible with Spring Boot 4.1.0)
- Maven for build
- PostgreSQL 15 as the database (one schema per service — database-per-service pattern)
- Lombok for boilerplate reduction
- Jakarta Bean Validation (jakarta.validation.constraints) for input validation on all request DTOs — note this is the Jakarta EE 11 baseline that ships with Spring Boot 4.x, so make sure all imports use jakarta.* (not the older javax.* package names)
- Thymeleaf for the frontend module only (web-client)
- JWT library: io.jsonwebtoken (jjwt) version 0.13.0, using the three-artifact setup (jjwt-api, jjwt-impl at runtime scope, jjwt-jackson at runtime scope) — NOT the old single jjwt artifact. Use the modern SecretKey-based fluent API introduced in jjwt 0.12+: build a SecretKey once via Keys.hmacShaKeyFor(secretBytes) or Jwts.SIG.HS256.key().build(), sign tokens with Jwts.builder()...signWith(secretKey)...compact(), and parse/validate tokens with Jwts.parser().verifyWith(secretKey).build().parseSignedClaims(token). Do NOT use the deprecated 0.11.x-style APIs — specifically avoid Jwts.parserBuilder(), .setSigningKey(), and .parseClaimsJws(), all of which are removed/deprecated in current jjwt.
- Testing: JUnit 5 and Mockito for unit tests (mocking dependencies to test business logic in isolation), Testcontainers (PostgreSQL module) for integration tests that run against a real, ephemeral database rather than an in-memory substitute. Target a minimum of ~70% line coverage per service — this feeds directly into the SonarQube quality gate configured in the CI/CD stage.
- Use ModelMapper (org.modelmapper:modelmapper) for all Entity-to-DTO and DTO-to-Entity conversions instead of writing manual field-by-field mapping code — configure one shared ModelMapper @Bean per service and inject it into services/controllers wherever a conversion is needed.

ARCHITECTURE:
- eureka-server: service registry, port 8761
- api-gateway: single entry point, port 8080, validates JWT, routes to services by name
- auth-service: port 8081, owns User identity and JWT issuing
- patient-service: port 8082, owns Patient profile data
- doctor-service: port 8083, owns Doctor profile and Slot availability
- appointment-service: port 8084, owns Appointment booking data, calls doctor-service via Feign
- web-client: Thymeleaf frontend, calls the gateway only, no database of its own
- ai-service: port 8086, no database, added LAST as a small final enhancement — wraps Spring AI 2.0 to suggest a doctor specialization from a patient's free-text symptom description

INTER-SERVICE COMMUNICATION: Use OpenFeign (spring-cloud-starter-openfeign) for all service-to-service HTTP calls, with clients resolved by Eureka service name (not hardcoded URLs). Wrap Feign calls with Resilience4j circuit breakers and fallbacks where a downstream failure shouldn't crash the calling request.

IMPORTANT DESIGN RULE: Each service has its OWN database. This means there are NO real JPA @ManyToOne/@OneToMany relationships between entities that live in different services — those are referenced only by plain Long ID fields (e.g. Appointment.doctorId is just a Long, not a @ManyToOne to a Doctor entity). Real JPA relationships (@OneToMany/@ManyToOne) are only used for entities within the SAME service, like Doctor and Slot both living in doctor-service.

SECURITY MODEL: JWT is issued by auth-service and validated once at the api-gateway. The gateway then forwards the authenticated user's ID and role as plain headers (X-User-Id, X-User-Role) to downstream services — those services trust these headers rather than re-validating the JWT themselves.

I'll give you one prompt per service to generate it fully. Please confirm you understand this setup, then wait for my next message with the first service.
```

Copilot will usually reply with a short confirmation — that's fine, it just means the context is loaded for the rest of the session. Then move straight into the per-service prompts below.

---

## 1. Eureka Server

```
Create a complete Spring Cloud Eureka Server module named eureka-server using Spring Boot 4.1.0 and Spring Cloud 2025.1.2.

Include:
- pom.xml with only the eureka-server starter dependency
- Main application class with @EnableEurekaServer
- application.yml running on port 8761, with self-registration (register-with-eureka, fetch-registry) set to false since this IS the registry

Package base: com.appointment.eureka
```

---

## 2. API Gateway

```
Create a complete Spring Cloud Gateway module named api-gateway using Spring Boot 4.1.0 and Spring Cloud 2025.1.2.

Include:
- pom.xml with spring-cloud-starter-gateway, eureka client, spring-boot-starter-security, jjwt-api, jjwt-impl (runtime), jjwt-jackson (runtime), all version 0.13.0
- Main application class with @EnableDiscoveryClient
- application.yml on port 8080, registering with Eureka at localhost:8761, defining routes using lb:// service discovery for: auth-service (/auth/**), patient-service (/patients/**), doctor-service (/doctors/**), appointment-service (/appointments/**)
- A GlobalFilter named JwtValidationFilter that:
  - Skips validation for /auth/register and /auth/login
  - Extracts the Bearer token from the Authorization header on all other routes
  - Validates it using HS256 with a shared secret from application.yml (property jwt.secret), built into a SecretKey via Keys.hmacShaKeyFor()
  - Uses the modern jjwt 0.13.0 fluent API: Jwts.parser().verifyWith(secretKey).build().parseSignedClaims(token) — do NOT use the deprecated parserBuilder()/setSigningKey()/parseClaimsJws() methods
  - Rejects with HTTP 401 and a JSON error body if missing, expired, or invalid
  - On success, extracts userId and role claims and adds them as X-User-Id and X-User-Role headers on the forwarded request
- Register this filter globally so it applies to all routed requests

Package base: com.appointment.gateway
```

---

## 3. Auth Service

```
Create a complete Spring Boot microservice named auth-service using Spring Boot 4.1.0, Java 25.

Include:
- pom.xml with: spring-boot-starter-web, spring-boot-starter-data-jpa, postgresql (org.postgresql:postgresql), spring-cloud-starter-netflix-eureka-client, spring-boot-starter-security, lombok, jjwt-api, jjwt-impl (runtime), jjwt-jackson (runtime), all version 0.13.0

ENTITY (this service owns identity only, no relationships to other entities — Patient/Doctor profile data lives in other services):
- User: id (Long, PK), name (String), email (String, unique, not null), password (String, not null), role (enum: PATIENT, DOCTOR, ADMIN), createdAt (LocalDateTime, default now)

VALIDATION on request DTOs (use jakarta.validation annotations, and add @Valid on controller method parameters):
- RegisterRequest: name @NotBlank @Size(min=2,max=100); email @NotBlank @Email; password @NotBlank @Size(min=6,max=100); role @NotNull
- LoginRequest: email @NotBlank @Email; password @NotBlank

LAYERS:
- UserRepository (Spring Data JPA) with a findByEmail(String email) method
- JwtUtil class: generateToken(User user) creating a JWT with claims sub=userId, email, role, signed HS256 with a secret from application.yml (property jwt.secret), expiring in 1 hour. Also include validateToken(String token) and extractClaims(String token). Use the jjwt 0.13.0 modern SecretKey-based API throughout: build the SecretKey once via Keys.hmacShaKeyFor(secret.getBytes()), sign with Jwts.builder()...signWith(secretKey)...compact(), and parse with Jwts.parser().verifyWith(secretKey).build().parseSignedClaims(token) — do NOT use Jwts.parserBuilder(), setSigningKey(), or parseClaimsJws(), which are the deprecated pre-0.12 style.
- AuthService with:
  - register(RegisterRequest dto): hashes password with BCryptPasswordEncoder, checks email uniqueness, saves User
  - login(LoginRequest dto): verifies email+password, throws a custom exception on failure, returns AuthResponse containing the JWT and role
- AuthController with POST /auth/register and POST /auth/login, using DTOs RegisterRequest{name,email,password,role}, LoginRequest{email,password}, AuthResponse{token,role,userId}
- A @ControllerAdvice for clean JSON error responses (e.g. duplicate email, bad credentials) with proper HTTP status codes

CONFIG:
- application.yml on port 8081, registering with Eureka at localhost:8761, PostgreSQL datasource (driver org.postgresql.Driver, dialect org.hibernate.dialect.PostgreSQLDialect) pointing to database auth_db, jwt.secret property, spring.jpa.hibernate.ddl-auto=update

Package base: com.appointment.auth
```

---

## 4. Patient Service

```
Create a complete Spring Boot microservice named patient-service using Spring Boot 4.1.0, Java 25.

Include:
- pom.xml with: spring-boot-starter-web, spring-boot-starter-data-jpa, postgresql (org.postgresql:postgresql), spring-cloud-starter-netflix-eureka-client, lombok

ENTITY (no JPA relationship to User — userId is just a plain foreign-key-style field since User lives in auth-service's own database):
- Patient: id (Long, PK), userId (Long, not null, references the User created in auth-service — NOT a JPA relationship, just store the ID), dob (LocalDate), address (String), medicalHistory (String, can be long text — use @Column(columnDefinition = "TEXT")), emergencyContact (String), createdAt (LocalDateTime, default now)

VALIDATION on PatientRequest DTO: dob @NotNull @Past; address @NotBlank @Size(max=255); emergencyContact @NotBlank @Pattern for a basic phone number format. IMPORTANT: PatientRequest must NOT contain a userId field — the userId is never accepted from the client, it is always read from the trusted X-User-Id header set by the Gateway (see controller notes below), so it cannot be spoofed by sending an arbitrary value in the request body.

LAYERS:
- PatientRepository with a findByUserId(Long userId) method
- PatientService with createProfile(PatientRequest dto), getByUserId(Long userId), updateProfile(Long userId, PatientRequest dto)
- PatientController with:
  - POST /patients — create profile. Do NOT take userId from the request body — read it from the X-User-Id header (set by the Gateway after JWT validation) and use that as the Patient's userId. The request body (PatientRequest) only contains profile fields (dob, address, medicalHistory, emergencyContact) — no identity fields at all.
  - GET /patients/{userId} — fetch profile
  - PUT /patients/{userId} — update profile
  - All write operations should read X-User-Id and X-User-Role headers from the request (these are set upstream by the API Gateway after JWT validation) and reject with 403 if X-User-Role is not PATIENT or ADMIN, or if X-User-Id doesn't match the userId being created/modified (unless role is ADMIN) — this guarantees a patient can only ever create or edit their own profile, never someone else's, since the userId always comes from their own verified JWT rather than anything they typed or sent

CONFIG:
- application.yml on port 8082, Eureka registration, PostgreSQL database patient_db (driver org.postgresql.Driver, dialect PostgreSQLDialect), ddl-auto=update

Package base: com.appointment.patient
```

---

## 5. Doctor Service

```
Create a complete Spring Boot microservice named doctor-service using Spring Boot 4.1.0, Java 25.

Include:
- pom.xml with: spring-boot-starter-web, spring-boot-starter-data-jpa, postgresql (org.postgresql:postgresql), spring-cloud-starter-netflix-eureka-client, lombok

ENTITIES with a REAL JPA relationship between them (both live in this service's own database):
- Doctor: id (Long, PK), userId (Long, not null — plain field referencing auth-service's User, NOT a JPA relationship since that's a different database), specialization (String), qualification (String), experienceYears (Integer), consultationFee (BigDecimal)
  - Has a @OneToMany(mappedBy = "doctor", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY) List<Slot> slots
- Slot: id (Long, PK), date (LocalDate), startTime (LocalTime), endTime (LocalTime), isBooked (boolean, default false)
  - Has a @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "doctor_id") Doctor doctor

VALIDATION on request DTOs: DoctorRequest — specialization @NotBlank; qualification @NotBlank; experienceYears @NotNull @Min(0) @Max(60); consultationFee @NotNull @DecimalMin("0.0") (IMPORTANT: DoctorRequest must NOT contain a userId field — see controller notes below for why). SlotRequest — date @NotNull @FutureOrPresent; startTime @NotNull; endTime @NotNull (add a class-level check or service-layer check that endTime is after startTime)

LAYERS:
- DoctorRepository with findBySpecialization(String specialization) and findByUserId(Long userId)
- SlotRepository with findByDoctorIdAndIsBookedFalse(Long doctorId)
- DoctorService with createDoctorProfile(dto), searchBySpecialization(String specialization), addSlot(Long doctorId, SlotRequest dto), getAvailableSlots(Long doctorId), and markSlotBooked(Long slotId) (used by appointment-service via Feign later)
- DoctorController:
  - POST /doctors — create profile. Do NOT take userId from the request body — read it from the X-User-Id header (set by the Gateway after JWT validation) and use that as the Doctor's userId. Reject with 403 if X-User-Role is not DOCTOR or ADMIN. The request body (DoctorRequest) only contains profile fields — no identity fields.
  - GET /doctors?specialization=X — search
  - GET /doctors/{id} — fetch one (needed by appointment-service via Feign)
  - POST /doctors/{id}/slots — add availability
  - GET /doctors/{id}/slots — list available slots
  - GET /doctors/slots/{slotId} — fetch single slot (needed by appointment-service via Feign)
  - PUT /doctors/slots/{slotId}/book — mark a slot as booked (called internally by appointment-service)
- Use DTOs, not entities, in request/response bodies to avoid lazy-loading serialization issues

CONFIG:
- application.yml on port 8083, Eureka registration, PostgreSQL database doctor_db (driver org.postgresql.Driver, dialect PostgreSQLDialect), ddl-auto=update

Package base: com.appointment.doctor
```

---

## 6. Appointment Service

```
Create a complete Spring Boot microservice named appointment-service using Spring Boot 4.1.0, Java 25.

Include:
- pom.xml with: spring-boot-starter-web, spring-boot-starter-data-jpa, postgresql (org.postgresql:postgresql), spring-cloud-starter-netflix-eureka-client, lombok, spring-cloud-starter-openfeign, spring-cloud-starter-circuitbreaker-resilience4j

ENTITY (no JPA relationships at all — doctorId, patientId, slotId are plain fields since those entities live in completely different databases in other services):
- Appointment: id (Long, PK), patientId (Long, not null), doctorId (Long, not null), slotId (Long, not null), status (enum: PENDING, APPROVED, REJECTED, CANCELLED, COMPLETED — default PENDING), reason (String), createdAt (LocalDateTime, default now)

VALIDATION on BookingRequest DTO: patientId @NotNull; doctorId @NotNull; slotId @NotNull; reason @NotBlank @Size(max=500)

LAYERS:
- AppointmentRepository with findByPatientId(Long patientId) and findByDoctorId(Long doctorId)
- Enable Feign clients with @EnableFeignClients on the main application class
- DoctorClient (Feign interface, @FeignClient(name = "doctor-service")):
  - GET /doctors/slots/{slotId} → getSlotById
  - PUT /doctors/slots/{slotId}/book → markSlotBooked
  - Add a fallback class DoctorClientFallback returning a clear error/default response when doctor-service is unreachable, wired via Resilience4j @CircuitBreaker
- AppointmentService with:
  - bookAppointment(BookingRequest dto): calls DoctorClient to fetch the slot, verifies it isn't already booked, calls markSlotBooked, then saves the Appointment as PENDING. Throw a clear exception if the slot is already booked or doctor-service is unavailable.
  - updateStatus(Long appointmentId, AppointmentStatus newStatus): used by doctors to approve/reject
  - getByPatientId / getByDoctorId
- AppointmentController:
  - POST /appointments — book (require X-User-Role = PATIENT from headers set by the gateway)
  - PUT /appointments/{id}/status — approve/reject (require X-User-Role = DOCTOR)
  - GET /appointments/patient/{id}
  - GET /appointments/doctor/{id}
- A @ControllerAdvice for clean JSON error responses on booking conflicts, not-found, and downstream service failures

CONFIG:
- application.yml on port 8084, Eureka registration, PostgreSQL database appointment_db (driver org.postgresql.Driver, dialect PostgreSQLDialect), ddl-auto=update, Feign/Resilience4j timeout and circuit breaker settings (e.g., 3s timeout, 50% failure threshold)

Package base: com.appointment.appointment
```

---

## Design theme (for web-client) — give this to Copilot right before the web-client prompt

This project uses **Tailwind CSS** instead of Bootstrap, with a custom theme so it doesn't look like a generic template. Paste this once, then the web-client prompt below.

```
For the web-client frontend, use Tailwind CSS (via the Play CDN script for simplicity, no build step needed) with Google Fonts, styled to this specific design system — do not default to Bootstrap-style components or generic dashboard styling:

COLOR PALETTE (define these as a custom Tailwind config via tailwind.config extended inline in the CDN script tag):
- canvas (page background): #F5F7F7
- surface (cards/panels): #FFFFFF
- ink (primary text): #14322F
- brand (primary actions, active nav, links): #0F6B60
- brand-dark (hover states): #0A4C44
- accent (secondary CTAs, highlights, warm touches): #E2A33B
- status-success (approved appointments): #3F8F6F
- status-pending (pending appointments): #E2A33B
- status-danger (rejected/cancelled appointments): #B5544A

TYPOGRAPHY (load via Google Fonts CDN link tags):
- Display font (page titles, dashboard section headers): "Fraunces" — a warm variable serif, used at larger weights (600-700) to give the product a human, unclinical feel
- Body font (all paragraph text, labels, buttons, nav): "IBM Plex Sans" — clean and professional
- Utility/data font (appointment times, dates, IDs, slot labels): "IBM Plex Mono" — gives scheduling data a precise, ticket-like feel

LAYOUT:
- Auth pages (login/register): centered card (max-w-md) on the canvas background, simple top bar with just the product wordmark in the display font
- Dashboard pages (patient/doctor/admin): a left sidebar (w-64, surface background, border-r) with role-based nav links, main content area with rounded-2xl cards (bg-surface, shadow-sm, p-6) on the canvas background
- Status badges: small rounded-full px-3 py-1 pills using the status-* colors, text in IBM Plex Mono, uppercase, tracking-wide, text-xs

SIGNATURE ELEMENT — the booking flow: 
- On the book-appointment page, show a 3-step horizontal stepper (Search → Select slot → Confirm) since this is a genuine sequence the user moves through — each step as a numbered circle connected by a line, current step filled in brand color
- Available time slots are rendered as a wrap-grid of rounded-full pill buttons (px-4 py-2, border border-brand text-brand for unselected, bg-brand text-white for the selected slot) — NOT a plain table or dropdown, this pill-grid is the signature interaction of the whole app

General rules: rounded-xl or rounded-2xl on all cards and major containers (not rounded-md — this is a warmer, softer identity), generous padding (p-6/p-8), no harsh drop shadows (shadow-sm max), buttons use bg-brand hover:bg-brand-dark text-white rounded-lg px-4 py-2 for primary actions and border border-brand text-brand for secondary actions.
```

---

## 7. web-client (Thymeleaf frontend) — split into 6 prompts, this module has the most moving parts

This module has NO database and NO entities — it only calls the API Gateway over HTTP and renders HTML. Give the "Design theme" prompt from above once at the start of this session, then run these 6 prompts in order.

### 7a — Project setup, base layout, and shared fragments

```
Create the foundation of a Spring Boot 4.1.0, Java 25 module named web-client, using Thymeleaf as the view layer (no database, no JPA — this module only calls the API Gateway over HTTP).

Include:
- pom.xml with: spring-boot-starter-web, spring-boot-starter-thymeleaf, thymeleaf-layout-dialect, lombok
- Main application class
- application.yml on port 8085, with a property gateway.base-url set to http://localhost:8080
- A RestTemplate @Bean configured with a ClientHttpRequestInterceptor that reads the JWT from the current HttpSession (attribute name "jwt") and attaches it as an "Authorization: Bearer <token>" header on every outgoing request, skipping this for requests to /auth/**
- DTO classes (records, in a dto package) matching what the backend services return, for use in Thymeleaf models: AuthResponseDto(String token, String role, Long userId), UserDto(Long id, String name, String email, String role), PatientDto(Long id, Long userId, LocalDate dob, String address, String medicalHistory, String emergencyContact), DoctorDto(Long id, Long userId, String specialization, String qualification, Integer experienceYears, BigDecimal consultationFee), SlotDto(Long id, Long doctorId, LocalDate date, LocalTime startTime, LocalTime endTime, boolean isBooked), AppointmentDto(Long id, Long patientId, Long doctorId, Long slotId, String status, String reason, LocalDateTime createdAt)
- The base Thymeleaf layout (layout.html, using Layout Dialect th:fragment) applying the design theme given earlier: Tailwind CDN with the custom color/font config in the <head>, a th:fragment="navbar" showing different links depending on a session role attribute (Home/Search/My Appointments for PATIENT, Dashboard/Manage Slots for DOCTOR, Dashboard/Doctors for ADMIN, plus Logout), and a th:fragment="footer" with a simple copyright line
- A separate auth-layout.html for login/register — centered card layout, no navbar, just the product wordmark in the display font
- An error.html page (styled per the theme) shown when a downstream API call fails, with a friendly message and a "Try again" link
- A GlobalControllerAdvice (@ControllerAdvice) that catches RestClientException/HttpClientErrorException/ResourceAccessException thrown by any controller, logs it, and forwards to the error.html view with a readable message instead of a stack trace

Package base: com.appointment.web
```

### 7b — Authentication pages and session/route security

```
Continuing the web-client module (package com.appointment.web), add:

- A SessionController-free approach: create AuthController with:
  - GET /login — returns login.html (extends auth-layout.html) with a LoginForm model attribute
  - POST /login — accepts email/password from the form, calls POST {gateway.base-url}/auth/login via RestTemplate, on success stores jwt, role, and userId as HttpSession attributes and redirects to /patient/dashboard, /doctor/dashboard, or /admin/dashboard based on role; on failure (401/400) re-renders login.html with an error message shown in a themed alert banner
  - GET /register — returns register.html (extends auth-layout.html) with a RegisterForm model attribute (name, email, password, confirmPassword, role dropdown limited to PATIENT/DOCTOR for self-registration)
  - POST /register — calls POST {gateway.base-url}/auth/register, on success redirects to /login with a success flash message, on failure re-renders with validation errors
  - GET /logout — invalidates the HttpSession and redirects to /login
- login.html and register.html: Tailwind-styled forms per the theme (centered card, rounded-2xl, brand-colored submit button), with client-side required attributes on inputs and inline error message spans below each field
- An AuthInterceptor (HandlerInterceptor) that checks HttpSession for a non-null "jwt" attribute before allowing access to any path under /patient/**, /doctor/**, /admin/** — redirects to /login with a "session expired" flash message if missing. Register it via a WebMvcConfigurer, excluding /login, /register, and static resources.
- A RoleInterceptor (or extend AuthInterceptor) that additionally checks the session's "role" attribute matches the path prefix (e.g. a PATIENT session hitting /doctor/** gets redirected with a 403 message) — this prevents a patient from manually typing a doctor URL into the browser
```

### 7c — Patient-facing pages

```
Continuing the web-client module, add the patient-facing pages. All calls go through RestTemplate to {gateway.base-url}, with the JWT auto-attached by the interceptor configured earlier.

- PatientController with:
  - GET /patient/dashboard — calls GET /appointments/patient/{userId} (userId from session), renders patient/dashboard.html showing appointments as a list of themed cards, each with a status pill badge (color per status: pending=amber, approved=green, rejected/cancelled=red), doctor name/specialization, date/time, and a "Cancel" button for PENDING/APPROVED ones that posts to /patient/appointments/{id}/cancel
  - GET /patient/search-doctors — renders patient/search-doctors.html with a specialization filter form (dropdown of common specializations) and, if a specialization is submitted, calls GET /doctors?specialization=X and shows results in a responsive Tailwind card grid (doctor name, specialization, experience, fee, a "Book" button linking to /patient/book/{doctorId})
  - GET /patient/book/{doctorId} — the signature booking page. Calls GET /doctors/{doctorId}/slots to fetch available slots, renders patient/book-appointment.html showing the 3-step stepper (Search → Select slot → Confirm, with step 2 active), the slots rendered as the pill-grid from the theme (grouped by date, each pill showing start-end time), a reason textarea, and a hidden field for the selected slotId set via a small inline script when a pill is clicked
  - POST /patient/book/{doctorId} — accepts the selected slotId and reason, calls POST /appointments on the gateway with patientId (from session), doctorId, slotId, reason — on success redirects to /patient/dashboard with a success flash message, on failure (e.g. slot already booked) re-renders book-appointment.html with an error banner
  - POST /patient/appointments/{id}/cancel — calls PUT /appointments/{id}/status with status=CANCELLED, redirects back to /patient/dashboard
- Use a flash message pattern (RedirectAttributes.addFlashAttribute) for success/error banners shown at the top of the next page, styled with the theme's status colors
```

### 7d — Doctor-facing pages

```
Continuing the web-client module, add the doctor-facing pages.

- DoctorController with:
  - GET /doctor/dashboard — calls GET /appointments/doctor/{userId} (resolve the doctor's own doctorId first if needed via a lookup, or store doctorId in session at login), renders doctor/dashboard.html — appointment requests grouped into "Pending" (with Approve/Reject buttons) and "Upcoming/History" sections, each as a themed card with status pill
  - POST /doctor/appointments/{id}/approve — calls PUT /appointments/{id}/status with status=APPROVED, redirects back to /doctor/dashboard with a flash message
  - POST /doctor/appointments/{id}/reject — calls PUT /appointments/{id}/status with status=REJECTED, redirects back with a flash message
  - GET /doctor/manage-slots — calls GET /doctors/{doctorId}/slots, renders doctor/manage-slots.html listing existing slots as pill badges grouped by date (booked ones shown grayed-out/disabled), plus a themed form to add a new slot (date picker, start time, end time)
  - POST /doctor/manage-slots — accepts the new slot form, calls POST /doctors/{doctorId}/slots, redirects back to /doctor/manage-slots with a success flash message
- Note: doctorId (the Doctor entity's own id, not the userId) needs to be resolved once — either the doctor-service exposes a GET /doctors/by-user/{userId} lookup you call once at login and cache in session, or you add that endpoint if it doesn't already exist in your doctor-service prompt. Ask me to confirm which approach before generating if it's ambiguous.
```

### 7e — Admin pages

```
Continuing the web-client module, add the admin dashboard.

- AdminController with:
  - GET /admin/dashboard — calls GET /doctors (no filter, returns all), and if available, aggregates simple counts (total doctors, total pending/approved/completed appointments — either via a dedicated summary endpoint or by fetching and counting client-side for this student-project scope), renders admin/dashboard.html with themed stat cards at the top (using the accent color for numbers) and a table of all doctors below (name, specialization, experience, fee)
  - Keep this page read-only for now (view-only doctor list + stats) unless you want doctor creation/deactivation added as a stretch goal — mention that as an optional "Prompt 7e-extended" if there's time
```

### 7f — Final polish pass

```
Review the web-client module as a whole and:
- Add a loading state (simple Tailwind spinner) shown via a small inline script while forms submit, since RestTemplate calls to the gateway are synchronous and may take a moment
- Add a favicon and a proper <title> per page using Thymeleaf layout dialect's title pattern
- Make sure every page gracefully handles the case where the gateway or a downstream service is unreachable (show the error.html rather than a raw exception/stack trace)
- Add basic responsive behavior: sidebar/navbar collapses to a hamburger menu on small screens (Tailwind's sm:/md: breakpoints)
- Double check every form has a matching validation message displayed inline when the backend returns a 400 with field errors
```

Package base for all of the above: com.appointment.web

---

## Stage 9 — Testing: JUnit unit tests + integration tests (do this after each service works, before CI/CD)

Run these once each backend service is functionally verified. Testing has two layers per service: **unit tests** (mock everything, test business logic fast and in isolation) and **integration tests** (spin up a real ephemeral PostgreSQL via Testcontainers and test the full request→database round trip). Skip web-client integration tests with Testcontainers — it has no database; use MockMvc with a mocked RestTemplate instead.

### Prompt — Auth Service tests

```
Add tests to the auth-service module.

UNIT TESTS (JUnit 5 + Mockito, package: test, mirroring main package structure):
- AuthServiceTest: mock UserRepository, PasswordEncoder, and JwtUtil. Test: successful registration saves a hashed password; registration with a duplicate email throws the expected exception; successful login with correct credentials returns a JWT; login with wrong password throws an authentication exception.
- JwtUtilTest: test that generateToken produces a token whose claims can be extracted back correctly, and that validateToken returns false for an expired or tampered token.

INTEGRATION TESTS (Spring Boot Test + Testcontainers):
- Add test dependencies: spring-boot-starter-test, org.testcontainers:junit-jupiter, org.testcontainers:postgresql
- Create a base integration test class annotated @SpringBootTest(webEnvironment = RANDOM_PORT) @Testcontainers, with a static PostgreSQLContainer<> registered via @DynamicPropertySource to override spring.datasource.url/username/password at runtime
- AuthControllerIntegrationTest extending that base: use MockMvc or WebTestClient to test POST /auth/register (assert 201 and that the user is persisted, and assert 400 for invalid input like a malformed email) and POST /auth/login (assert 200 with a token for correct credentials, 401 for incorrect password)
```

### Prompt — Patient Service tests

```
Add tests to the patient-service module, following the same pattern as auth-service: Testcontainers PostgreSQL base class, spring-boot-starter-test + testcontainers dependencies.

UNIT TESTS: PatientServiceTest — mock PatientRepository, test createProfile, getByUserId (including not-found case throwing the expected exception), and updateProfile.

INTEGRATION TESTS: PatientControllerIntegrationTest — test POST /patients (201 on valid input, 400 on validation failure e.g. missing address), GET /patients/{userId} (200 with correct data, 404 for a non-existent userId), PUT /patients/{userId} (200 with updated fields persisted). Also test the role-based header check: a request without X-User-Role or with an unauthorized role should return 403.
```

### Prompt — Doctor Service tests

```
Add tests to the doctor-service module, same Testcontainers pattern as before.

UNIT TESTS: DoctorServiceTest — mock DoctorRepository and SlotRepository, test createDoctorProfile, searchBySpecialization, addSlot (including a case where endTime is before startTime should be rejected), getAvailableSlots (only returns isBooked=false), and markSlotBooked.

INTEGRATION TESTS: DoctorControllerIntegrationTest — test the full flow: create a doctor, add a slot, fetch available slots (confirm the new slot appears), call the internal book endpoint, then fetch available slots again (confirm the now-booked slot no longer appears). This end-to-end sequence within one test validates the Doctor–Slot JPA relationship is working correctly against a real database.
```

### Prompt — Appointment Service tests (the most important one — this has your core business logic)

```
Add tests to the appointment-service module, same Testcontainers pattern as before.

UNIT TESTS: AppointmentServiceTest — mock AppointmentRepository and DoctorClient (the Feign interface). Test:
- Successful booking: DoctorClient returns an unbooked slot, service saves a new PENDING appointment and calls markSlotBooked
- Booking conflict: DoctorClient returns a slot with isBooked=true, service throws a clear "slot already booked" exception and does NOT save an appointment
- Downstream failure: DoctorClient throws an exception simulating doctor-service being unreachable, verify the fallback behavior returns a clear error rather than propagating a raw exception
- updateStatus: verify a valid status transition succeeds and an invalid one (e.g. approving an already-CANCELLED appointment) is rejected

INTEGRATION TESTS: AppointmentControllerIntegrationTest — use Testcontainers for the database, but mock DoctorClient with @MockBean (a Feign client should be stubbed in integration tests, not called against a real doctor-service instance) so you can control its response deterministically. Test: POST /appointments succeeds when the mocked DoctorClient reports the slot open, POST /appointments returns a conflict response when the mocked slot is already booked, GET /appointments/patient/{id} returns the correct list after creating a few test appointments directly via the repository.
```

### Prompt — API Gateway tests

```
Add tests to the api-gateway module for the JwtValidationFilter.

Use @SpringBootTest(webEnvironment = RANDOM_PORT) with WebTestClient (Spring Cloud Gateway is reactive, so use WebTestClient, not MockMvc). Test:
- A request to a protected route without an Authorization header is rejected with 401
- A request with a valid, unexpired JWT (generate one in the test using the same secret and jjwt 0.13.0 API as the real filter) is allowed through and the X-User-Id/X-User-Role headers are present on the forwarded request (you can verify this with a simple test route that echoes headers back)
- A request with an expired JWT is rejected with 401
- A request to /auth/login is allowed through with no Authorization header at all (since this path is excluded from validation)
```

### Prompt — web-client tests

```
Add tests to the web-client module. Since this module has no database, use @WebMvcTest (not @SpringBootTest) scoped to each controller, with RestTemplate mocked via @MockBean.

Test AuthController: POST /login with a mocked RestTemplate returning a successful AuthResponseDto redirects to the correct dashboard based on role and populates the session; a mocked 401 response re-renders login.html with an error model attribute.
Test AuthInterceptor: a request to /patient/dashboard with no session redirects to /login; a request with a valid session role mismatch (e.g. DOCTOR session hitting /patient/**) is redirected/rejected.
```

### After running these

- [ ] Run `./mvnw test` in each service and confirm all tests pass before moving to Stage 10 (CI/CD) — the SonarQube coverage gate depends on these tests actually existing and running successfully in the pipeline's Unit Test stage
- [ ] Spot-check that the JaCoCo report (added in Stage 10's Prompt 1) shows coverage over your ~70% target once these tests are in place

---

## Stage 10 — Spring AI Integration (do this LAST functional piece, after everything else works and is tested, before CI/CD)

This is deliberately the final feature added. It is a small, isolated service — if something goes wrong here, it should never put the already-working core system at risk. Do not touch any existing service's code for this stage; it only adds one new service, one gateway route, and one small web-client addition.

### Prompt — AI Service

```
Create a complete Spring Boot 4.1.0, Java 25 microservice named ai-service, using Spring AI 2.0 (GA, built for Spring Boot 4.0/4.1 and Spring Framework 7). This service has NO database.

Include:
- pom.xml with: spring-boot-starter-web, spring-cloud-starter-netflix-eureka-client, spring-ai-starter-model-anthropic, lombok
- Main application class, registering with Eureka
- application.yml on port 8086, Eureka registration, and the Anthropic API key property (spring.ai.anthropic.api-key) sourced from an environment variable, never hardcoded
- A SpecializationSuggestionService that uses Spring AI's ChatClient to take a free-text symptom description and return a suggested specialization plus a one-sentence explanation. Constrain the model's response to ONE specialization chosen from this fixed list: Cardiology, Dermatology, Orthopedics, ENT, General Physician, Pediatrics, Gynecology, Neurology — do this by instructing the model explicitly in the prompt template to only pick from this list, and parse/validate the response server-side, falling back to "General Physician" if the model's response doesn't cleanly match one of the allowed values
- A SuggestionRequest DTO (String symptoms, validated @NotBlank @Size(max=500)) and a SuggestionResponse DTO (String specialization, String explanation)
- An AiController with POST /ai/suggest-specialization, reading X-User-Role from the header and requiring PATIENT
- A @ControllerAdvice that catches any Spring AI / model-provider exception and returns a clean, generic error response rather than leaking provider-specific error details

Package base: com.appointment.ai
```

### Prompt — API Gateway route addition

```
Add one new route to the existing api-gateway application.yml: route ai-service under path /ai/**, using the same lb:// Eureka-based service discovery pattern as the other routes. This path should go through the same JwtValidationFilter as every other protected route (no exclusion needed) — do not modify the filter itself, only add the new route entry.
```

### Prompt — web-client wiring (floating chatbot widget)

```
Add a floating AI chatbot widget to the web-client's base Thymeleaf layout (layout.html) as a reusable fragment (th:fragment="ai-chat-widget"), so it appears on every patient-facing page (patient/** dashboards) rather than being embedded in one specific page. Do not add it to the auth-layout (login/register).

WIDGET BUTTON:
- A small circular floating action button fixed to the bottom-right corner of the viewport (position: fixed; bottom-6 right-6; a rounded-full bg-brand button with a chat-bubble icon from lucide or a simple inline SVG), sitting above page content with a high z-index, visible on scroll

CHAT PANEL (toggled open/closed by clicking the button):
- A compact panel fixed near the bottom-right (w-80, h-96, rounded-2xl, shadow-lg, bg-surface, positioned just above the floating button)
- Header bar: "Symptom Assistant" in the display font, with a small close (X) button
- A scrollable message list area showing the conversation so far
- A text input with a send button pinned to the bottom of the panel

MESSAGE BEHAVIOR:
- User messages render as right-aligned bubbles (bg-brand, text-white, rounded-2xl)
- Assistant responses render as left-aligned bubbles (bg-canvas, rounded-2xl)
- On send: POST the message text to /ai/suggest-specialization on the gateway (via the existing RestTemplate-backed endpoint pattern already used elsewhere in web-client, with the session JWT attached), then render the response as an assistant bubble showing the suggested specialization and its one-line explanation
- Below that assistant bubble, show a small quick-action link/button: "Search doctors in {specialization}" linking to /patient/search-doctors?specialization={value}, so the suggestion is immediately actionable
- If the call fails for any reason (timeout, error, AI service down), show a single generic assistant bubble ("Sorry, I can't help with that right now — try searching manually from the menu") rather than breaking the widget or leaving it stuck loading

IMPLEMENTATION NOTES:
- Keep the open/close toggle and message list rendering to a small inline <script> block (vanilla JS with fetch()) — no separate JS framework needed for this
- The widget's scope stays narrow: it only handles "describe symptoms, get a specialization suggestion." It should not attempt to answer unrelated questions — if you want to expand its scope later to general FAQs (how to cancel, how to reschedule), that would need a different system prompt in the AI Service and is out of scope for this pass
- This is a single shared fragment included once in layout.html, not duplicated per page
```

### Checkpoint

- [ ] Confirm ai-service registers with Eureka and the gateway routes /ai/** correctly
- [ ] Test POST /ai/suggest-specialization directly through the gateway with a sample symptom description, confirm it returns one of the fixed specialization values
- [ ] Open the chat widget on a couple of different patient pages and confirm it's the same shared fragment (not duplicated/inconsistent) and that it doesn't appear on login/register
- [ ] Manually stop ai-service and confirm the rest of the app still works normally, with the chat widget showing its generic fallback message instead of breaking — this is the one thing worth testing deliberately, since the whole point of adding this last is that it must not be a single point of failure for the core app

---

## Stage 11 — CI/CD: Docker, SonarQube, Jenkins (do this LAST, after everything — including the AI Service — builds and runs locally)

### Manual one-time setup (not a Copilot prompt — do this yourself first)

- [ ] Install/run SonarQube locally (easiest: `docker run -d -p 9000:9000 sonarqube:community`), log in, generate a **project token** for each service (or one global token) under My Account → Security
- [ ] Install/run Jenkins (`docker run -d -p 8090:8080 jenkins/jenkins:lts`), install the **SonarQube Scanner** and **Docker Pipeline** plugins
- [ ] In Jenkins, add the SonarQube token as a Credential, and configure the SonarQube server URL under Manage Jenkins → System → SonarQube servers
- [ ] Make sure Docker is available to your Jenkins agent (either Docker-in-Docker, or mount the host's Docker socket for a simple student setup)

Copilot can't do these console/UI steps for you — but it can generate everything that goes IN the pipeline once these exist.

### Prompt 1 — SonarQube config per service

```
Add SonarQube analysis support to this Spring Boot 4.1.0 / Maven service. Add the org.sonarsource.scanner.maven:sonar-maven-plugin to pom.xml, and add these properties: sonar.projectKey (use the artifactId), sonar.host.url pointing to http://localhost:9000, and sonar.java.source set to 25. Also add jacoco-maven-plugin configured to generate a coverage report during the test phase, and wire sonar.coverage.jacoco.xmlReportPaths to that report, so SonarQube shows real test coverage, not just static analysis.
```
Run this once per service (or once, then copy the plugin block into the others — it's identical).

### Prompt 2 — Dockerfile per service

```
Create a multi-stage Dockerfile for this Spring Boot 4.1.0 / Java 25 Maven service. Stage 1 uses a maven:3.9-eclipse-temurin-25 image to build the jar with mvn clean package -DskipTests. Stage 2 copies only the built jar into a minimal eclipse-temurin:25-jre-alpine runtime image, exposes the correct port, and runs it with java -jar. Keep the final image as small as possible.
```
Run once per service, adjusting the exposed port to match (8081 for auth-service, 8082 for patient-service, etc).

### Prompt 3 — docker-compose.yml for local integration testing

```
Create a docker-compose.yml at the root of the project that runs: a PostgreSQL container with 4 databases (auth_db, patient_db, doctor_db, appointment_db — use an init script), eureka-server, api-gateway, auth-service, patient-service, doctor-service, appointment-service, ai-service, and web-client — each built from its own Dockerfile. Set proper depends_on ordering (eureka first, then gateway, then business services and ai-service, then web-client), and pass each service's database URL (where applicable), Eureka URL, JWT secret, and — for ai-service — the Anthropic API key, as environment variables rather than hardcoding them in application.yml.
```

### Prompt 4 — the Jenkinsfile itself

```
Create a declarative Jenkinsfile (Jenkinsfile, Groovy) at the project root for a multi-module microservices project with these modules: eureka-server, api-gateway, auth-service, patient-service, doctor-service, appointment-service, web-client, ai-service (each in its own subfolder, each an independent Maven project — not a Maven multi-module reactor).

Pipeline stages:
1. Checkout — pull from git
2. Build — for each service subfolder, run ./<service>/mvnw -f <service>/pom.xml clean package -DskipTests (use the Maven Wrapper checked into each service, not a global mvn install on the Jenkins agent; use a scripted loop over a list of service names, or explicit parallel stages)
3. Unit Tests — run ./<service>/mvnw -f <service>/pom.xml test for each service, publish JUnit results with junit '**/target/surefire-reports/*.xml'
4. SonarQube Analysis — for each service, run ./<service>/mvnw -f <service>/pom.xml sonar:sonar inside a withSonarQubeEnv block
5. Quality Gate — waitForQualityGate abortPipeline: true, so the pipeline fails if SonarQube's quality gate fails
6. Docker Build — build a Docker image for each service tagged with the Jenkins BUILD_NUMBER
7. Docker Push — (optional, skip or point at a local registry) push each image
8. Deploy — run docker-compose down && docker-compose up -d --build to redeploy the full stack locally

Use environment {} for shared variables like the SonarQube server name and Docker registry, and make sure a failure in any single service's build/test/sonar stage fails the whole pipeline with a clear message showing which service failed.

Note: use each service's checked-in Maven Wrapper (./mvnw) for all Maven commands rather than assuming a global mvn is installed on the Jenkins agent — this keeps the pipeline portable to agents that don't have Maven pre-installed.
```

### Order to actually run this stage

1. Prompt 1 on each service (adds Sonar+Jacoco to pom.xml) → run `./mvnw sonar:sonar` manually once per service to confirm it reports into your SonarQube dashboard
2. Prompt 2 on each service → build each image manually (`docker build`) to confirm it runs standalone before trusting Jenkins with it
3. Prompt 3 → bring the whole stack up with `docker-compose up` manually, confirm the full booking flow still works in containers
4. Prompt 4 → only now wire it into Jenkins, since by this point every piece it orchestrates has already been proven to work on its own



1. Eureka Server → verify dashboard loads
2. API Gateway → verify it registers with Eureka (routes will 404 until services exist, that's fine)
3. Auth Service → test register/login through the gateway with Postman, confirm a JWT comes back
4. Patient Service → test create/fetch profile through the gateway with a real JWT in the header
5. Doctor Service → test create doctor, add slots, search by specialization
6. Appointment Service → test full booking flow, confirm the slot gets marked booked in doctor-service
7. web-client → walk the full user journey in the browser: register → login → search → book → approve

## If a generated service is too big to review at once

Ask Copilot to regenerate just one file from the same prompt context: *"From what you just generated, show me only the AppointmentService class again with more detail on the booking conflict logic."* This is usually cheaper than re-running the full prompt.
