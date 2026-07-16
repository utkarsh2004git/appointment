# Project Context — Online Appointment Management System

Paste everything below as one message to Copilot 365, after it confirms it understood the structure prompt.

```
Here is all the project detail. Use this to write the full document per the structure you confirmed.

PROJECT NAME: Online Appointment Management System (OAMS)

WHAT IT DOES: A microservices-based web platform where patients search for doctors by specialization, view real-time slot availability, and book appointments online. Doctors manage their own availability and approve or reject incoming booking requests. Admins get a read-only view of all doctors and system-wide appointment activity. This was built as a case study for a Capgemini training program on Prompt Engineering and Agentic AI — the entire codebase was generated through structured, iterative prompts given to GitHub Copilot rather than written manually, so document the AI-assisted development process itself as a first-class part of this case study, not just the resulting application.

TECHNOLOGY STACK:
- Language: Java 25
- Application framework: Spring Boot 4.1.0
- Microservices framework: Spring Cloud 2025.1.2
- Service discovery: Netflix Eureka
- API Gateway: Spring Cloud Gateway
- Security: Spring Security + JWT, using io.jsonwebtoken (jjwt) version 0.13.0 with its modern SecretKey-based fluent API
- Inter-service communication: OpenFeign, clients resolved via Eureka service name, wrapped with Resilience4j circuit breakers and fallbacks
- Database: PostgreSQL 15, one separate database per microservice (database-per-service pattern)
- Frontend: Thymeleaf (server-rendered, not a SPA) styled with Tailwind CSS, using a custom design theme (deep teal primary color, warm amber accent, Fraunces display font, IBM Plex Sans body font, IBM Plex Mono for times/data)
- Build tool: Maven
- CI/CD: Jenkins declarative pipeline
- Code quality: SonarQube with JaCoCo test coverage
- Containerization: Docker, orchestrated locally with Docker Compose

USER ROLES:
1. Patient — registers/logs in, searches doctors by specialization, views a doctor's available slots, books an appointment with a reason, views their appointment history and status, can cancel a pending/approved appointment.
2. Doctor — registers/logs in, maintains their profile (specialization, qualification, experience, consultation fee), defines available time slots, views incoming appointment requests, approves or rejects them.
3. Admin — views a directory of all registered doctors, views system-wide appointment statistics. (Read-only in current scope; full user/doctor management identified as a future enhancement.)

ARCHITECTURE — 7 SERVICES:
1. Eureka Server, port 8761 — service registry, all other services register here and discover each other by name, no database.
2. API Gateway, port 8080 — single entry point for all client traffic, validates JWTs on every request except /auth/login and /auth/register, routes to backend services via Eureka-based service discovery, no database.
3. Auth Service, port 8081, database auth_db — owns user identity: registration, login, password hashing (BCrypt), JWT issuing.
4. Patient Service, port 8082, database patient_db — owns patient profile data.
5. Doctor Service, port 8083, database doctor_db — owns doctor profile data and slot availability (this is the one service with a real internal JPA relationship, Doctor to Slot).
6. Appointment Service, port 8084, database appointment_db — owns appointment booking data; calls Doctor Service via OpenFeign to verify and reserve a slot before confirming a booking.
7. web-client, port 8085 — Thymeleaf frontend module, no database of its own, calls only the API Gateway over HTTP.

IMPORTANT ARCHITECTURAL RULE TO DOCUMENT: because each service has its own database, there are no real foreign-key/JPA relationships between entities owned by different services — those are referenced only by plain ID fields (e.g. an Appointment's doctorId is just a number, resolved at call time via a Feign call, not a database join). Real JPA relationships only exist between entities within the same service's database (Doctor and Slot both live in doctor_db).

DATA MODEL / ENTITIES (group these into a Database Design section, one subsection per service):
- auth_db → User: id, name, email (unique), password (BCrypt-hashed), role (enum: PATIENT, DOCTOR, ADMIN), createdAt
- patient_db → Patient: id, userId (references User.id logically, not a JPA relation), dob, address, medicalHistory, emergencyContact, createdAt
- doctor_db → Doctor: id, userId (logical reference), specialization, qualification, experienceYears, consultationFee — has a real one-to-many relationship to Slot
- doctor_db → Slot: id, doctorId (real foreign key to Doctor within this same database), date, startTime, endTime, isBooked (boolean)
- appointment_db → Appointment: id, patientId, doctorId, slotId (all three are plain reference fields, not JPA relations, since Patient/Doctor/Slot live in other services' databases), status (enum: PENDING, APPROVED, REJECTED, CANCELLED, COMPLETED), reason, createdAt

SECURITY MODEL (JWT):
- Auth Service is the only service that issues tokens. On successful login it builds a JWT containing claims sub (userId), email, and role, signed with HS256 using a shared secret, expiring in 1 hour.
- The API Gateway is the only service that validates tokens. It checks every incoming request's Authorization: Bearer header, verifies the signature and expiry, and rejects invalid requests with HTTP 401 before they reach any backend service.
- After successful validation, the Gateway extracts the userId and role from the token and forwards them as plain headers (X-User-Id, X-User-Role) to whichever backend service the request is routed to. Backend services trust these headers rather than re-validating the JWT themselves — this is a deliberate "validate once at the edge" design decision worth explaining in the architecture section.

INTER-SERVICE COMMUNICATION: All service-to-service calls (not client-to-service, which goes through the Gateway) use OpenFeign, with clients resolved dynamically by Eureka service name rather than hardcoded URLs, and wrapped with Resilience4j circuit breakers so a downstream outage degrades gracefully with a fallback response instead of crashing the calling request. The concrete example: when a patient books an appointment, Appointment Service calls Doctor Service via a Feign client to confirm the chosen slot is still open and to mark it as booked, before persisting the new Appointment record as PENDING.

CORE WORKFLOW (appointment booking, describe as a numbered sequence):
1. Patient registers and logs in — Auth Service authenticates and issues a JWT.
2. Patient searches and selects a doctor — queries Doctor Service (through the Gateway) filtered by specialization.
3. Patient views that doctor's available slots — Doctor Service returns only currently unbooked slots.
4. Patient books an appointment, choosing a slot and providing a reason — Appointment Service calls Doctor Service via Feign to atomically verify and reserve the slot, then saves the appointment as PENDING.
5. The doctor reviews the request from their dashboard and approves or rejects it. A rejection returns the patient to slot selection to choose a different time.
6. On approval, the appointment status updates and is reflected in the patient's dashboard as confirmed.

API ENDPOINTS (group by service, list Method | Endpoint | Auth requirement | Description):

Auth Service (/auth/**):
- POST /auth/register — Public — registers a new user (name, email, password, role); password is BCrypt-hashed
- POST /auth/login — Public — verifies credentials, returns a signed JWT with role and userId

Patient Service (/patients/**):
- POST /patients — PATIENT/ADMIN — creates a patient profile linked to a User
- GET /patients/{userId} — Owner/ADMIN — fetches a patient's profile
- PUT /patients/{userId} — Owner/ADMIN — updates a patient's profile

Doctor Service (/doctors/**):
- POST /doctors — DOCTOR/ADMIN — creates a doctor profile
- GET /doctors?specialization= — Any authenticated — searches doctors by specialization
- GET /doctors/{id} — Any authenticated + internal (Feign) — fetches one doctor
- GET /doctors/by-user/{userId} — Owner/ADMIN — looks up a doctor's profile by their userId
- POST /doctors/{id}/slots — DOCTOR owner/ADMIN — adds a new availability slot
- GET /doctors/{id}/slots — Any authenticated — lists a doctor's available slots
- GET /doctors/slots/{slotId} — Internal (Feign) — fetches a single slot, called by Appointment Service
- PUT /doctors/slots/{slotId}/book — Internal (Feign) — marks a slot booked, called by Appointment Service

Appointment Service (/appointments/**):
- POST /appointments — PATIENT — books an appointment, verifying/reserving the slot via Feign first
- PUT /appointments/{id}/status — DOCTOR (approve/reject) or owning PATIENT (cancel) — updates status
- GET /appointments/patient/{patientId} — Owner/ADMIN — lists a patient's appointments
- GET /appointments/doctor/{doctorId} — Owner/ADMIN — lists a doctor's appointments

API Gateway routing summary:
- /auth/** → auth-service (JWT check skipped only for login/register)
- /patients/** → patient-service (JWT required)
- /doctors/** → doctor-service (JWT required)
- /appointments/** → appointment-service (JWT required)

CI/CD PIPELINE (Jenkins, describe as numbered stages):
1. Checkout — pulls source from git
2. Build — runs mvn clean package -DskipTests for each of the 7 modules
3. Unit Test — runs mvn test per module, publishes JUnit results
4. SonarQube Analysis — runs mvn sonar:sonar per module, uploading static analysis + JaCoCo coverage
5. Quality Gate — pipeline waits for and enforces SonarQube's quality gate; a failing gate aborts the pipeline
6. Docker Build — builds a tagged Docker image per service from a multi-stage Dockerfile
7. Docker Push — optional, pushes to a registry (skipped in the local/demo setup)
8. Deploy — runs docker-compose up -d --build to redeploy the full stack locally

PROMPT ENGINEERING METHODOLOGY (document this as its own detailed section — this is central to the training):
- A single context-setting overview prompt was reused at the start of every AI session, fixing shared decisions (versions, architecture, database-per-service rule, security model) once so every later prompt stayed consistent without repeating those constraints.
- Each backend service was generated through one comprehensive, self-contained prompt specifying its entities, field validation, JPA relationship rules (real relationships only within the same service, plain ID fields across services), layered structure, and configuration — scoped tightly enough to review before moving on.
- The Thymeleaf frontend, being the most component-heavy module, was deliberately split into 6 focused sub-prompts (setup/layout, auth pages, patient pages, doctor pages, admin pages, final polish) instead of one large prompt, because a single large frontend prompt produced inconsistent styling and incomplete pages.
- CI/CD configuration was generated last, only after every service was confirmed working locally by hand — pointing a pipeline at unverified code makes it impossible to tell whether a failure is a pipeline problem or an application problem.
- CONCRETE EXAMPLE TO INCLUDE: an early prompt for JWT handling only specified a library version ("use jjwt 0.13.x") without constraining the API style. The AI agent defaulted to the deprecated pre-0.12 pattern (Jwts.parserBuilder().setSigningKey(...).parseClaimsJws(...)) from its more common training examples, even though the newer version was specified. The fix was a revised prompt that explicitly named both the desired modern method chain (Jwts.parser().verifyWith(secretKey).build().parseSignedClaims(...)) and the specific deprecated methods to avoid. Lesson: naming a version is necessary but not sufficient — the prompt must also constrain the exact API surface expected, especially for libraries with a recent breaking-change history.
- Follow-up corrections during development were scoped as small, targeted asks (e.g. regenerate only one class) rather than full re-generations, to keep each iteration cheap and independently reviewable.

CHALLENGES FACED (pair each with how it was resolved, for the Challenges & Lessons Learned section):
- AI agent defaulting to outdated library APIs → resolved by explicitly naming both the desired modern calls and the deprecated ones to avoid, not just a version number.
- Maintaining architectural consistency across 7 independently generated modules → resolved by re-establishing the overview/context prompt every session and restating cross-cutting rules in every relevant service prompt rather than assuming they'd carry over.
- Frontend generation producing inconsistent styling across pages → resolved by splitting into scoped sub-prompts and providing one explicit design token specification up front.
- Coordinating JWT issuance (Auth Service) and validation (API Gateway) as two separately generated services → resolved by documenting the shared contract (same secret, same claim structure) once in the overview prompt.
- Not knowing whether a CI/CD failure was a code problem or a pipeline problem → resolved by manually verifying each piece (build a Docker image by hand, run mvn sonar:sonar by hand) before wiring it into Jenkins.

FUTURE ENHANCEMENTS TO MENTION:
- A Notification Service for async email/SMS confirmations via a message broker
- Distributed tracing and centralized logging across services
- Stronger cross-service identity verification (validating patientId against Patient Service at booking time)
- RSA-based JWT signing instead of a shared HMAC secret
- Full admin capabilities (doctor onboarding/deactivation, user management)
- Kubernetes orchestration instead of Docker Compose for production-realistic deployment

Write the full document now using this information, following the structure and formatting rules from your previous confirmation.
```
