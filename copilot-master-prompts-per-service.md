# Copilot Master Prompts — One Prompt Per Service

Each block below is ONE prompt for ONE service. Paste the whole block into Copilot Chat with that service's folder open. Review the generated files, build, and test before moving to the next service.

**Important concept to understand before you start:** Within a single service, entities that belong together use real JPA relationships (`@OneToMany`, `@ManyToOne`). Across services (e.g., Appointment referencing a Doctor), there is NO real JPA relationship — you just store the plain ID (`Long doctorId`) because that data lives in a different database entirely. Each prompt below tells Copilot explicitly which case applies, so it doesn't accidentally try to `@ManyToOne` something that lives in another microservice.

---

## 1. Eureka Server

```
Create a complete Spring Cloud Eureka Server module named eureka-server using Spring Boot 3.x and Spring Cloud 2023.x.

Include:
- pom.xml with only the eureka-server starter dependency
- Main application class with @EnableEurekaServer
- application.yml running on port 8761, with self-registration (register-with-eureka, fetch-registry) set to false since this IS the registry

Package base: com.appointment.eureka
```

---

## 2. API Gateway

```
Create a complete Spring Cloud Gateway module named api-gateway using Spring Boot 3.x and Spring Cloud 2023.x.

Include:
- pom.xml with spring-cloud-starter-gateway, eureka client, spring-boot-starter-security, jjwt (io.jsonwebtoken) 0.11.x
- Main application class with @EnableDiscoveryClient
- application.yml on port 8080, registering with Eureka at localhost:8761, defining routes using lb:// service discovery for: auth-service (/auth/**), patient-service (/patients/**), doctor-service (/doctors/**), appointment-service (/appointments/**)
- A GlobalFilter named JwtValidationFilter that:
  - Skips validation for /auth/register and /auth/login
  - Extracts the Bearer token from the Authorization header on all other routes
  - Validates it using HS256 with a shared secret from application.yml (property jwt.secret)
  - Rejects with HTTP 401 and a JSON error body if missing, expired, or invalid
  - On success, extracts userId and role claims and adds them as X-User-Id and X-User-Role headers on the forwarded request
- Register this filter globally so it applies to all routed requests

Package base: com.appointment.gateway
```

---

## 3. Auth Service

```
Create a complete Spring Boot microservice named auth-service using Spring Boot 3.x, Java 17.

Include:
- pom.xml with: spring-boot-starter-web, spring-boot-starter-data-jpa, mysql-connector-j, spring-cloud-starter-netflix-eureka-client, spring-boot-starter-security, lombok, jjwt (io.jsonwebtoken) 0.11.x for api/impl/jackson

ENTITY (this service owns identity only, no relationships to other entities — Patient/Doctor profile data lives in other services):
- User: id (Long, PK), name (String), email (String, unique, not null), password (String, not null), role (enum: PATIENT, DOCTOR, ADMIN), createdAt (LocalDateTime, default now)

LAYERS:
- UserRepository (Spring Data JPA) with a findByEmail(String email) method
- JwtUtil class: generateToken(User user) creating a JWT with claims sub=userId, email, role, signed HS256 with a secret from application.yml (property jwt.secret), expiring in 1 hour. Also include validateToken(String token) and extractClaims(String token).
- AuthService with:
  - register(RegisterRequest dto): hashes password with BCryptPasswordEncoder, checks email uniqueness, saves User
  - login(LoginRequest dto): verifies email+password, throws a custom exception on failure, returns AuthResponse containing the JWT and role
- AuthController with POST /auth/register and POST /auth/login, using DTOs RegisterRequest{name,email,password,role}, LoginRequest{email,password}, AuthResponse{token,role,userId}
- A @ControllerAdvice for clean JSON error responses (e.g. duplicate email, bad credentials) with proper HTTP status codes

CONFIG:
- application.yml on port 8081, registering with Eureka at localhost:8761, MySQL datasource pointing to schema auth_db, jwt.secret property, spring.jpa.hibernate.ddl-auto=update

Package base: com.appointment.auth
```

---

## 4. Patient Service

```
Create a complete Spring Boot microservice named patient-service using Spring Boot 3.x, Java 17.

Include:
- pom.xml with: spring-boot-starter-web, spring-boot-starter-data-jpa, mysql-connector-j, spring-cloud-starter-netflix-eureka-client, lombok

ENTITY (no JPA relationship to User — userId is just a plain foreign-key-style field since User lives in auth-service's own database):
- Patient: id (Long, PK), userId (Long, not null, references the User created in auth-service — NOT a JPA relationship, just store the ID), dob (LocalDate), address (String), medicalHistory (String, can be long text — use @Column(columnDefinition = "TEXT")), emergencyContact (String), createdAt (LocalDateTime, default now)

LAYERS:
- PatientRepository with a findByUserId(Long userId) method
- PatientService with createProfile(PatientRequest dto), getByUserId(Long userId), updateProfile(Long userId, PatientRequest dto)
- PatientController with:
  - POST /patients — create profile
  - GET /patients/{userId} — fetch profile
  - PUT /patients/{userId} — update profile
  - All write operations should read X-User-Id and X-User-Role headers from the request (these are set upstream by the API Gateway after JWT validation) and reject with 403 if X-User-Role is not PATIENT or ADMIN, or if X-User-Id doesn't match the userId being modified (unless role is ADMIN)

CONFIG:
- application.yml on port 8082, Eureka registration, MySQL schema patient_db, ddl-auto=update

Package base: com.appointment.patient
```

---

## 5. Doctor Service

```
Create a complete Spring Boot microservice named doctor-service using Spring Boot 3.x, Java 17.

Include:
- pom.xml with: spring-boot-starter-web, spring-boot-starter-data-jpa, mysql-connector-j, spring-cloud-starter-netflix-eureka-client, lombok

ENTITIES with a REAL JPA relationship between them (both live in this service's own database):
- Doctor: id (Long, PK), userId (Long, not null — plain field referencing auth-service's User, NOT a JPA relationship since that's a different database), specialization (String), qualification (String), experienceYears (Integer), consultationFee (BigDecimal)
  - Has a @OneToMany(mappedBy = "doctor", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY) List<Slot> slots
- Slot: id (Long, PK), date (LocalDate), startTime (LocalTime), endTime (LocalTime), isBooked (boolean, default false)
  - Has a @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "doctor_id") Doctor doctor

LAYERS:
- DoctorRepository with findBySpecialization(String specialization) and findByUserId(Long userId)
- SlotRepository with findByDoctorIdAndIsBookedFalse(Long doctorId)
- DoctorService with createDoctorProfile(dto), searchBySpecialization(String specialization), addSlot(Long doctorId, SlotRequest dto), getAvailableSlots(Long doctorId), and markSlotBooked(Long slotId) (used by appointment-service via Feign later)
- DoctorController:
  - POST /doctors — create profile
  - GET /doctors?specialization=X — search
  - GET /doctors/{id} — fetch one (needed by appointment-service via Feign)
  - POST /doctors/{id}/slots — add availability
  - GET /doctors/{id}/slots — list available slots
  - GET /doctors/slots/{slotId} — fetch single slot (needed by appointment-service via Feign)
  - PUT /doctors/slots/{slotId}/book — mark a slot as booked (called internally by appointment-service)
- Use DTOs, not entities, in request/response bodies to avoid lazy-loading serialization issues

CONFIG:
- application.yml on port 8083, Eureka registration, MySQL schema doctor_db, ddl-auto=update

Package base: com.appointment.doctor
```

---

## 6. Appointment Service

```
Create a complete Spring Boot microservice named appointment-service using Spring Boot 3.x, Java 17.

Include:
- pom.xml with: spring-boot-starter-web, spring-boot-starter-data-jpa, mysql-connector-j, spring-cloud-starter-netflix-eureka-client, lombok, spring-cloud-starter-openfeign, spring-cloud-starter-circuitbreaker-resilience4j

ENTITY (no JPA relationships at all — doctorId, patientId, slotId are plain fields since those entities live in completely different databases in other services):
- Appointment: id (Long, PK), patientId (Long, not null), doctorId (Long, not null), slotId (Long, not null), status (enum: PENDING, APPROVED, REJECTED, CANCELLED, COMPLETED — default PENDING), reason (String), createdAt (LocalDateTime, default now)

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
- application.yml on port 8084, Eureka registration, MySQL schema appointment_db, ddl-auto=update, Feign/Resilience4j timeout and circuit breaker settings (e.g., 3s timeout, 50% failure threshold)

Package base: com.appointment.appointment
```

---

## 7. web-client (Thymeleaf frontend)

```
Create a complete Spring Boot module named web-client using Spring Boot 3.x, Java 17, with Thymeleaf as the view layer. This module has NO database and NO entities — it only calls the API Gateway over HTTP and renders HTML.

Include:
- pom.xml with: spring-boot-starter-web, spring-boot-starter-thymeleaf, thymeleaf-layout-dialect, lombok

SETUP:
- A base Thymeleaf layout (using Layout Dialect) with a shared navbar (shows different links based on session role: patient/doctor/admin) and footer, styled with Bootstrap 5 via CDN
- A RestTemplate bean with a ClientHttpRequestInterceptor that reads the JWT from the current HttpSession and attaches it as an "Authorization: Bearer <token>" header on every outgoing request
- A HandlerInterceptor named AuthInterceptor that checks HttpSession for a valid JWT before allowing access to /patient/**, /doctor/**, /admin/** paths — redirects to /login if missing, registered via WebMvcConfigurer

PAGES + CONTROLLERS (all calling the gateway at http://localhost:8080 via RestTemplate):
- /login (GET shows form, POST calls /auth/login on the gateway, stores JWT+role+userId in HttpSession, redirects by role)
- /register (GET shows form, POST calls /auth/register on the gateway)
- /patient/dashboard (GET calls /appointments/patient/{userId}, lists appointments with status badges)
- /patient/search-doctors (GET form filters by specialization, calls /doctors?specialization=, results in a Bootstrap card grid with a "Book" button per doctor)
- /patient/book/{doctorId} (GET calls /doctors/{id}/slots to show available slots, POST submits to /appointments on the gateway)
- /doctor/dashboard (GET calls /appointments/doctor/{userId}, shows pending requests with Approve/Reject buttons posting to /appointments/{id}/status)
- /doctor/manage-slots (GET lists existing slots via /doctors/{id}/slots, POST adds a new slot via /doctors/{id}/slots)
- /admin/dashboard (GET shows a simple table of doctors and basic counts)
- A GET /logout that invalidates the session and redirects to /login

Package base: com.appointment.web
```

---

## Order to run these in

1. Eureka Server → verify dashboard loads
2. API Gateway → verify it registers with Eureka (routes will 404 until services exist, that's fine)
3. Auth Service → test register/login through the gateway with Postman, confirm a JWT comes back
4. Patient Service → test create/fetch profile through the gateway with a real JWT in the header
5. Doctor Service → test create doctor, add slots, search by specialization
6. Appointment Service → test full booking flow, confirm the slot gets marked booked in doctor-service
7. web-client → walk the full user journey in the browser: register → login → search → book → approve

## If a generated service is too big to review at once

Ask Copilot to regenerate just one file from the same prompt context: *"From what you just generated, show me only the AppointmentService class again with more detail on the booking conflict logic."* This is usually cheaper than re-running the full prompt.
