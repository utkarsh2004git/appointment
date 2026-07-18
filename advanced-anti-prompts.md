# AI Agent Generation Prompts — Online Appointment Management System

> Copy each prompt section below and give it to an AI coding agent to recreate the exact codebase. Each prompt is self-contained with full schemas, business logic, endpoint contracts, and configuration.

---

## PROMPT 0: System Architecture Overview (Give This First)

```
You are building an "Online Appointment Management System" as a Spring Boot microservices
application. The entire system consists of 7 independent Spring Boot projects inside a
single parent directory. Each service is a separate Maven project (NOT a multi-module build).

### Services & Ports:
| Service              | Port | spring.application.name | Database (PostgreSQL) |
|----------------------|------|-------------------------|-----------------------|
| eureka-server        | 8761 | eureka-server           | None                  |
| auth-service         | 8081 | auth-service            | auth_db               |
| api-gateway          | 8080 | api-gateway             | None                  |
| patient-service      | 8082 | patient-service         | patient_db            |
| doctor-service       | 8083 | doctor-service          | doctor_db             |
| appointment-service  | 8084 | appointment-service     | appointment_db        |
| web-client           | 8085 | web-client              | None (Thymeleaf UI)   |

### Technology Stack:
- Java 25, Spring Boot 3.x, Spring Cloud (Eureka, Gateway, OpenFeign)
- PostgreSQL with Spring Data JPA + Hibernate (ddl-auto: update)
- Lombok (@Data, @Builder, @Getter, @Setter, @NoArgsConstructor, @AllArgsConstructor, @RequiredArgsConstructor)
- Jakarta Validation API
- JWT (jjwt-api, jjwt-impl, jjwt-jackson) for authentication
- Thymeleaf + Thymeleaf Layout Dialect for web-client
- Resilience4j for circuit breakers in appointment-service
- BCrypt password encoding

### Security Model:
1. Users register/login via auth-service which returns a JWT token.
2. The web-client stores the JWT in HttpSession (server-side session, NOT cookie-based JWT).
3. All REST API calls from web-client go through the api-gateway at http://localhost:8080.
4. The api-gateway has a GlobalFilter (JwtValidationFilter) that:
   - Skips validation for /auth/register and /auth/login paths.
   - Extracts the Bearer token from the Authorization header.
   - Parses JWT claims to extract userId and role.
   - Mutates the downstream request to inject X-User-Id and X-User-Role headers.
   - Returns 401 if token is missing/invalid/expired.
5. Downstream services (patient, doctor, appointment) NEVER validate JWT directly.
   They only read X-User-Id and X-User-Role headers injected by the gateway.
6. Each downstream service has a validateAccess() helper that checks X-User-Id matches
   the resource owner, unless X-User-Role is ADMIN.

### Package Structure Convention:
Each service follows: com.appointment.<service-name>
Sub-packages: controller, service, entity, dto, repository, exception
(plus config, security, client, middleware, filter as needed)
```

---

## PROMPT 1: Eureka Server

```
Generate the eureka-server Spring Boot application.

### pom.xml Dependencies:
- spring-cloud-starter-netflix-eureka-server

### Main Class:
- Package: com.appointment.eureka
- Annotations: @SpringBootApplication, @EnableEurekaServer

### application.yml:
server:
  port: 8761
eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/

This is the simplest service. No entities, no controllers.
```

---

## PROMPT 2: auth-service

```
Generate the auth-service Spring Boot application.

### pom.xml Dependencies:
- spring-boot-starter-web
- spring-boot-starter-data-jpa
- spring-boot-starter-validation
- spring-boot-starter-security
- spring-cloud-starter-netflix-eureka-client
- postgresql driver
- lombok
- jjwt-api (0.12.6), jjwt-impl, jjwt-jackson

### Package: com.appointment.auth

### Entity — User (table: "users"):
| Field     | Type          | Constraints                              |
|-----------|---------------|------------------------------------------|
| id        | Long          | @Id @GeneratedValue(IDENTITY)            |
| name      | String        | @Column(nullable=false)                  |
| email     | String        | @Column(unique=true, nullable=false)     |
| password  | String        | @Column(nullable=false)                  |
| role      | UserRole enum | @Enumerated(STRING) @Column(nullable=false) |
| createdAt | LocalDateTime | @Column(updatable=false) @Builder.Default = LocalDateTime.now() |

### Enum — UserRole:
PATIENT, DOCTOR, ADMIN

### DTO — RegisterRequest:
| Field    | Type     | Validation                                     |
|----------|----------|-------------------------------------------------|
| name     | String   | @NotBlank, @Size(min=2, max=100)                |
| email    | String   | @NotBlank, @Email                               |
| password | String   | @NotBlank, @Size(min=6, max=100)                |
| role     | UserRole | @NotNull                                        |

### DTO — LoginRequest:
| Field    | Type   | Validation         |
|----------|--------|--------------------|
| email    | String | @NotBlank, @Email  |
| password | String | @NotBlank          |

### DTO — AuthResponse:
| Field  | Type     |
|--------|----------|
| token  | String   |
| role   | UserRole |
| userId | Long     |

### Repository — UserRepository:
- extends JpaRepository<User, Long>
- Optional<User> findByEmail(String email)

### Service — AuthService:
- **register(RegisterRequest dto)**:
  1. Check if email already exists → throw EmailAlreadyExistsException.
  2. Build User with BCrypt-encoded password.
  3. Save and return User.
- **login(LoginRequest dto)**:
  1. Find user by email → throw InvalidCredentialsException if not found.
  2. Verify password with passwordEncoder.matches() → throw InvalidCredentialsException if wrong.
  3. Generate JWT token using JwtUtil.
  4. Return AuthResponse with token, role, userId.

### Security — JwtUtil:
- @Value("${jwt.secret}") private String secret
- **generateToken(User user)**: Creates JWT with claims: email, role (as name()), userId.
  Subject = String.valueOf(user.getId()). Expiration = 1 hour. Signed with HMAC-SHA key.
- **validateToken(String token)**: Parses and checks expiration. Returns boolean.
- **extractClaims(String token)**: Returns Claims object.
- Signing key: Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8))

### Config — SecurityConfig:
- @EnableWebSecurity
- BCryptPasswordEncoder bean
- SecurityFilterChain: disable CSRF, STATELESS sessions, permit /auth/register and /auth/login,
  authenticate all other requests.

### Controller — AuthController (@RequestMapping("/auth")):
| Method | Path      | Request Body     | Response              | Notes                          |
|--------|-----------|------------------|-----------------------|--------------------------------|
| POST   | /register | RegisterRequest  | User (201 CREATED)    | Clear password in response     |
| POST   | /login    | LoginRequest     | AuthResponse (200 OK) |                                |

### Exception Classes:
- EmailAlreadyExistsException extends RuntimeException
- InvalidCredentialsException extends RuntimeException
- GlobalExceptionHandler with @ControllerAdvice handling both + MethodArgumentNotValidException

### application.yml:
server.port: 8081
spring.application.name: auth-service
spring.datasource.url: jdbc:postgresql://localhost:5432/auth_db
spring.datasource.username: postgres
spring.datasource.password: 
spring.jpa.hibernate.ddl-auto: update
spring.jpa.properties.hibernate.dialect: org.hibernate.dialect.PostgreSQLDialect
spring.jpa.show-sql: true
eureka.client.service-url.defaultZone: http://localhost:8761/eureka/
eureka.instance.prefer-ip-address: true
jwt.secret: ""
```

---

## PROMPT 3: api-gateway

```
Generate the api-gateway Spring Boot application using Spring Cloud Gateway (reactive/WebFlux).

### pom.xml Dependencies:
- spring-cloud-starter-gateway (reactive gateway)
- spring-cloud-starter-netflix-eureka-client
- spring-boot-starter-security (WebFlux security)
- jjwt-api (0.12.6), jjwt-impl, jjwt-jackson
- spring-boot-starter-actuator

### Package: com.appointment.gateway

### Main Class:
- @SpringBootApplication, @EnableDiscoveryClient

### Config — SecurityConfig:
- @EnableWebFluxSecurity (NOT @EnableWebSecurity — this is reactive!)
- SecurityWebFilterChain bean: disable CSRF, permit ALL exchanges.

### Filter — JwtValidationFilter (implements GlobalFilter, Ordered):
- @Value("${jwt.secret}") private String secret
- getOrder() returns -1 (execute early)
- filter() logic:
  1. Extract path from exchange.getRequest().getURI().getPath().
  2. SKIP validation if path is "/auth/register" or "/auth/login" → chain.filter(exchange).
  3. Extract Authorization header. If null or doesn't start with "Bearer " → return 401 JSON error.
  4. Parse token substring(7) with Jwts.parser().verifyWith(getSigningKey()).build().parseSignedClaims().
  5. Extract userId from claims: try "userId" key, then "user_id", then subject. Convert to String.
  6. Extract role from claims: try "role" key, then "roles", then "userRole". Convert to String.
  7. Mutate the request to add headers "X-User-Id" and "X-User-Role".
  8. chain.filter(mutatedExchange).
  9. On any exception → return 401 JSON error body.
- onError helper: Sets response status, Content-Type JSON, writes error body as DataBuffer.

### application.yml:
server.port: 8080
spring.application.name: api-gateway
spring.cloud.gateway.server.webflux.routes:
  - id: auth-service
    uri: lb://auth-service
    predicates: Path=/auth/**
  - id: patient-service
    uri: lb://patient-service
    predicates: Path=/patients/**
  - id: doctor-service
    uri: lb://doctor-service
    predicates: Path=/doctors/**
  - id: appointment-service
    uri: lb://appointment-service
    predicates: Path=/appointments/**
eureka.client.service-url.defaultZone: http://localhost:8761/eureka/
eureka.instance.prefer-ip-address: true
management.endpoints.web.exposure.include: "*"
logging.level.org.springframework.cloud.gateway: TRACE
jwt.secret: ""
```

---

## PROMPT 4: patient-service

```
Generate the patient-service Spring Boot application.

### pom.xml Dependencies:
- spring-boot-starter-web
- spring-boot-starter-data-jpa
- spring-boot-starter-validation
- spring-cloud-starter-netflix-eureka-client
- postgresql driver
- lombok

### Package: com.appointment.patient

### Entity — Patient (table: "patients"):
| Field          | Type          | Column Annotation                                     |
|----------------|---------------|-------------------------------------------------------|
| id             | Long          | @Id @GeneratedValue(IDENTITY)                         |
| userId         | Long          | @Column(name="user_id", nullable=false, unique=true)  |
| name           | String        | @Column(name="name")                                  |
| contactNumber  | String        | @Column(name="contact_number")                        |
| dob            | LocalDate     | @Column(nullable=false)                               |
| address        | String        | @Column(nullable=false)                               |
| medicalHistory | String        | @Column(name="medical_history", columnDefinition="TEXT") |
| createdAt      | LocalDateTime | @Column(updatable=false) @Builder.Default = LocalDateTime.now() |

### DTO — PatientRequest:
| Field          | Type      | Validation                                               |
|----------------|-----------|----------------------------------------------------------|
| name           | String    | @NotBlank(message="Name is required")                    |
| contactNumber  | String    | @NotBlank, @Pattern(regexp="^[+]?[0-9\\s\\-()]{7,20}$") |
| dob            | LocalDate | @NotNull, @Past(message="Date of birth must be in the past") |
| address        | String    | @NotBlank, @Size(max=255)                                |
| medicalHistory | String    | (optional, no validation)                                |

### DTO — PatientResponse:
All fields from Patient entity mapped 1:1: id, userId, name, contactNumber, dob, address,
medicalHistory, createdAt.

### Repository — PatientRepository:
- extends JpaRepository<Patient, Long>
- Optional<Patient> findByUserId(Long userId)

### Service — PatientService:
- **createProfile(Long userId, PatientRequest request)**:
  1. Check if profile already exists for userId → throw IllegalArgumentException.
  2. Build Patient entity from request fields + userId.
  3. Save and return mapped PatientResponse.
- **getByUserId(Long userId)**: Find by userId or throw ResourceNotFoundException.
- **getById(Long id)**: Find by id or throw ResourceNotFoundException.
- **updateProfile(Long userId, PatientRequest request)**:
  1. Find by userId or throw ResourceNotFoundException.
  2. Set all fields from request onto the entity.
  3. Save and return mapped PatientResponse.
- **mapToResponse(Patient)**: Maps entity to PatientResponse using builder pattern.

### Controller — PatientController (@RequestMapping("/patients")):
| Method | Path          | Headers Required        | Body            | Response               |
|--------|---------------|-------------------------|-----------------|------------------------|
| POST   | /patients     | X-User-Id, X-User-Role  | PatientRequest  | PatientResponse (201)  |
| GET    | /{userId}     | None                    | —               | PatientResponse (200)  |
| GET    | /profile/{id} | None                    | —               | PatientResponse (200)  |
| PUT    | /{userId}     | X-User-Id, X-User-Role  | PatientRequest  | PatientResponse (200)  |

- **validateAccess(headerUserId, headerRole, targetUserId)** helper:
  1. If headerUserId or headerRole is null → throw ForbiddenException.
  2. Must be ADMIN or PATIENT role → else throw ForbiddenException.
  3. If PATIENT, parse headerUserId as Long, compare to targetUserId → throw ForbiddenException on mismatch.

### Exception Classes:
- ResourceNotFoundException extends RuntimeException (single String constructor)
- ForbiddenException extends RuntimeException (single String constructor)
- GlobalExceptionHandler (@ControllerAdvice) with handlers for:
  - ResourceNotFoundException → 404 JSON body
  - ForbiddenException → 403 JSON body
  - IllegalArgumentException → 400 JSON body
  - MethodArgumentNotValidException → 400 JSON body with field errors map
  - Exception → 500 JSON body
  Each response body is Map<String, Object> with keys: timestamp, status, error, message.

### application.yml:
server.port: 8082
spring.application.name: patient-service
spring.jackson.datatype.datetime.write-dates-as-timestamps: false
spring.datasource.url: jdbc:postgresql://localhost:5432/patient_db
spring.datasource.username: postgres
spring.datasource.password: 
spring.jpa.hibernate.ddl-auto: update
spring.jpa.properties.hibernate.dialect: org.hibernate.dialect.PostgreSQLDialect
spring.jpa.show-sql: true
eureka.client.service-url.defaultZone: http://localhost:8761/eureka/
eureka.instance.prefer-ip-address: true
```

---

## PROMPT 5: doctor-service

```
Generate the doctor-service Spring Boot application.

### pom.xml Dependencies:
- spring-boot-starter-web
- spring-boot-starter-data-jpa
- spring-boot-starter-validation
- spring-cloud-starter-netflix-eureka-client
- postgresql driver
- lombok

### Package: com.appointment.doctor

### Entity — Doctor (table: "doctors"):
| Field           | Type         | Column                                        |
|-----------------|--------------|-----------------------------------------------|
| id              | Long         | @Id @GeneratedValue(IDENTITY)                 |
| userId          | Long         | @Column(name="user_id", nullable=false, unique=true) |
| name            | String       | @Column(name="name")                          |
| contactNumber   | String       | @Column(name="contact_number")                |
| specialization  | String       | @Column(nullable=false)                       |
| qualification   | String       | @Column(nullable=false)                       |
| experienceYears | Integer      | @Column(name="experience_years", nullable=false) |
| consultationFee | BigDecimal   | @Column(name="consultation_fee", nullable=false) |
| slots           | List<Slot>   | @OneToMany(mappedBy="doctor", cascade=ALL, orphanRemoval=true, fetch=LAZY) @Builder.Default = new ArrayList<>() |

### Entity — Slot (table: "slots"):
| Field     | Type      | Column                                         |
|-----------|-----------|------------------------------------------------|
| id        | Long      | @Id @GeneratedValue(IDENTITY)                  |
| date      | LocalDate | @Column(nullable=false)                        |
| startTime | LocalTime | @Column(name="start_time", nullable=false)     |
| endTime   | LocalTime | @Column(name="end_time", nullable=false)       |
| isBooked  | boolean   | @Column(name="is_booked", nullable=false) @Builder.Default = false |
| doctor    | Doctor    | @ManyToOne(fetch=LAZY) @JoinColumn(name="doctor_id", nullable=false) |

### DTO — DoctorRequest:
| Field           | Type       | Validation                                       |
|-----------------|------------|--------------------------------------------------|
| specialization  | String     | @NotBlank                                        |
| qualification   | String     | @NotBlank                                        |
| experienceYears | Integer    | @NotNull, @Min(0), @Max(60)                      |
| consultationFee | BigDecimal | @NotNull, @DecimalMin("0.0")                     |
| name            | String     | (optional)                                       |
| contactNumber   | String     | (optional)                                       |

### DTO — DoctorResponse:
id, userId, specialization, qualification, experienceYears, consultationFee, name, contactNumber.

### DTO — SlotRequest:
| Field     | Type      | Validation                                    |
|-----------|-----------|-----------------------------------------------|
| date      | LocalDate | @NotNull, @FutureOrPresent                    |
| startTime | LocalTime | @NotNull                                      |
| endTime   | LocalTime | @NotNull                                      |

### DTO — SlotResponse:
id, date, startTime, endTime, isBooked (with @JsonProperty("isBooked")),
doctorId, isExpired (with @JsonProperty("isExpired")).
IMPORTANT: Both boolean fields need @com.fasterxml.jackson.annotation.JsonProperty annotations
to prevent Jackson from serializing them as "booked" and "expired" (stripping the "is" prefix).

### Repository — DoctorRepository:
- extends JpaRepository<Doctor, Long>
- List<Doctor> findBySpecializationIgnoreCase(String specialization)
- Optional<Doctor> findByUserId(Long userId)

### Repository — SlotRepository:
- extends JpaRepository<Slot, Long>
- List<Slot> findByDoctorIdAndIsBookedFalse(Long doctorId)

### Service — DoctorService:
- **createDoctorProfile(Long userId, DoctorRequest request)**:
  1. Check if profile exists → throw IllegalArgumentException.
  2. Build Doctor, save, return DoctorResponse.
- **updateDoctor(Long id, DoctorRequest request)**: ← THIS IS THE PUT UPDATE ENDPOINT
  1. Find by id or throw ResourceNotFoundException.
  2. Set ALL fields: specialization, qualification, experienceYears, consultationFee, name, contactNumber.
  3. Save and return DoctorResponse.
- **searchBySpecialization(String specialization)**:
  If null/empty → findAll, else findBySpecializationIgnoreCase. Map to list of DoctorResponse.
- **getDoctorById(Long id)**: Find by id or throw.
- **getDoctorByUserId(Long userId)**: Find by userId or throw.
- **addSlot(Long doctorId, SlotRequest request)**:
  1. Find doctor by id.
  2. Validate date is not in the past.
  3. If date is TODAY, validate startTime is not before (now - 15 minutes) → throw IllegalArgumentException.
  4. Validate endTime > startTime.
  5. Build Slot with isBooked=false, save, return SlotResponse.
- **getAvailableSlots(Long doctorId)**: Find unbooked slots. Map with isExpired calculation.
- **getSlotById(Long slotId)**: Find slot or throw.
- **markSlotBooked(Long slotId)**: Find slot, check not already booked, set booked=true, save.
- **editSlot(Long slotId, SlotRequest request)**:
  1. Find slot or throw.
  2. If already booked → throw IllegalArgumentException("Cannot edit a slot that is already booked").
  3. Same date/time validation as addSlot (past date check, today's time check with 15-min buffer).
  4. Validate endTime > startTime.
  5. Update date, startTime, endTime. Save.
- **deleteSlot(Long slotId)**: (IMPORTANT — also implement this)
  1. Find slot or throw.
  2. If booked → throw IllegalArgumentException("Cannot delete a slot that is already booked").
  3. slotRepository.delete(slot).
- **mapToSlotResponse(Slot)**: Calculate isExpired dynamically:
  isExpired = slot.date.isBefore(today) || (slot.date.isEqual(today) && LocalTime.now().isAfter(slot.endTime))

### Controller — DoctorController (@RequestMapping("/doctors")):
| Method | Path              | Headers              | Body          | Response             |
|--------|-------------------|----------------------|---------------|----------------------|
| POST   | /doctors          | X-User-Id, X-User-Role | DoctorRequest | DoctorResponse (201) |
| GET    | /doctors          | None                 | ?specialization | List<DoctorResponse> |
| GET    | /{id}             | None                 | —             | DoctorResponse       |
| PUT    | /{id}             | X-User-Id, X-User-Role | DoctorRequest | DoctorResponse (200) |
| GET    | /user/{userId}    | None                 | —             | DoctorResponse       |
| POST   | /{id}/slots       | X-User-Id, X-User-Role | SlotRequest   | SlotResponse (201)   |
| GET    | /{id}/slots       | None                 | —             | List<SlotResponse>   |
| GET    | /slots/{slotId}   | None                 | —             | SlotResponse         |
| PUT    | /slots/{slotId}/book | None              | —             | SlotResponse         |
| PUT    | /slots/{slotId}   | X-User-Id, X-User-Role | SlotRequest   | SlotResponse         |
| DELETE | /slots/{slotId}   | X-User-Id, X-User-Role | —             | 204 No Content       |

- **validateAccess(headerUserId, headerRole, targetUserId, requiredRole)** helper:
  1. Null check on both headers → ForbiddenException.
  2. Must be ADMIN or the requiredRole → ForbiddenException.
  3. If requiredRole matches headerRole, parse headerUserId as Long, compare to targetUserId.
  This is reusable — the requiredRole parameter makes it work for both DOCTOR and PATIENT contexts.

### Exception Classes:
Same pattern as patient-service: ResourceNotFoundException, ForbiddenException, GlobalExceptionHandler.

### application.yml:
server.port: 8083
spring.application.name: doctor-service
spring.jackson.datatype.datetime.write-dates-as-timestamps: false
spring.datasource.url: jdbc:postgresql://localhost:5432/doctor_db
(rest same as patient-service)
```

---

## PROMPT 6: appointment-service

```
Generate the appointment-service Spring Boot application.

### pom.xml Dependencies:
- spring-boot-starter-web
- spring-boot-starter-data-jpa
- spring-boot-starter-validation
- spring-cloud-starter-netflix-eureka-client
- spring-cloud-starter-openfeign
- spring-cloud-starter-circuitbreaker-resilience4j
- postgresql driver
- lombok

### Package: com.appointment.appointment

### Main Class:
- @SpringBootApplication, @EnableDiscoveryClient, @EnableFeignClients

### Enum — AppointmentStatus:
PENDING, APPROVED, REJECTED, CANCELLED, COMPLETED

### Entity — Appointment (table: "appointments"):
| Field     | Type              | Column                                            |
|-----------|-------------------|---------------------------------------------------|
| id        | Long              | @Id @GeneratedValue(IDENTITY)                     |
| patientId | Long              | @Column(name="patient_id", nullable=false)        |
| doctorId  | Long              | @Column(name="doctor_id", nullable=false)         |
| slotId    | Long              | @Column(name="slot_id", nullable=false)           |
| status    | AppointmentStatus | @Enumerated(STRING) @Column(nullable=false) @Builder.Default = PENDING |
| reason    | String            | @Column(columnDefinition="TEXT")                  |
| createdAt | LocalDateTime     | @Column(updatable=false) @Builder.Default = LocalDateTime.now() |

### DTO — BookingRequest:
| Field     | Type   | Validation                            |
|-----------|--------|---------------------------------------|
| patientId | Long   | @NotNull                              |
| doctorId  | Long   | @NotNull                              |
| slotId    | Long   | @NotNull                              |
| reason    | String | @NotBlank, @Size(max=500)             |

### DTO — AppointmentResponse:
id, patientId, doctorId, slotId, status (AppointmentStatus enum), reason, createdAt.

### DTO — SlotResponse (LOCAL COPY in dto package — mirrors doctor-service):
id, date (LocalDate), startTime (LocalTime), endTime (LocalTime),
isBooked (boolean, @JsonProperty("isBooked")), doctorId (Long),
isExpired (boolean, @JsonProperty("isExpired")).

### Feign Client — DoctorClient:
@FeignClient(name = "doctor-service", fallback = DoctorClientFallback.class)
- @GetMapping("/doctors/slots/{slotId}") → SlotResponse getSlotById(@PathVariable Long slotId)
- @PutMapping("/doctors/slots/{slotId}/book") → SlotResponse markSlotBooked(@PathVariable Long slotId)

### Fallback — DoctorClientFallback (@Component implements DoctorClient):
Both methods throw DownstreamServiceUnavailableException with descriptive message.

### Repository — AppointmentRepository:
- extends JpaRepository<Appointment, Long>
- List<Appointment> findByPatientId(Long patientId)
- List<Appointment> findByDoctorId(Long doctorId)

### Service — AppointmentService:
- **bookAppointment(BookingRequest request)**:
  1. Fetch slot from doctor-service via DoctorClient.getSlotById(request.slotId).
  2. If null → throw ResourceNotFoundException.
  3. Validate doctorId matches slot.doctorId → throw IllegalArgumentException on mismatch.
  4. If slot.isBooked → throw BookingConflictException.
  5. EXPIRATION CHECK: Calculate if slot is expired using LocalDate.now() and LocalTime.now().
     If date is before today, or date equals today and now is after endTime → throw BookingConflictException.
  6. Mark slot as booked via DoctorClient.markSlotBooked(slotId).
  7. Build Appointment entity with status=PENDING, save, return AppointmentResponse.
- **updateStatus(Long appointmentId, AppointmentStatus newStatus)**:
  Find by id, set status, save, return.
- **getByPatientId(Long patientId)**: Return list mapped to AppointmentResponse.
- **getByDoctorId(Long doctorId)**: Return list mapped to AppointmentResponse.

### Controller — AppointmentController (@RequestMapping("/appointments")):
| Method | Path                | Headers             | Params/Body       | Response                    |
|--------|---------------------|---------------------|-------------------|-----------------------------|
| POST   | /appointments       | X-User-Id, X-User-Role | BookingRequest  | AppointmentResponse (201)   |
| PUT    | /{id}/status        | X-User-Id, X-User-Role | ?status=APPROVED | AppointmentResponse (200)  |
| GET    | /patient/{id}       | None                | —                 | List<AppointmentResponse>   |
| GET    | /doctor/{id}        | None                | —                 | List<AppointmentResponse>   |

- **requireRole(headerRole, requiredRole)** helper:
  If null → ForbiddenException. If not ADMIN and not requiredRole → ForbiddenException.
- POST /appointments requires PATIENT role.
- PUT /{id}/status requires DOCTOR role.

### Exception Classes:
- ResourceNotFoundException, ForbiddenException, BookingConflictException,
  DownstreamServiceUnavailableException
- GlobalExceptionHandler handles all + MethodArgumentNotValidException

### application.yml:
server.port: 8084
spring.application.name: appointment-service
spring.jackson.datatype.datetime.write-dates-as-timestamps: false
spring.datasource.url: jdbc:postgresql://localhost:5432/appointment_db
spring.cloud.openfeign.circuitbreaker.enabled: true
spring.cloud.openfeign.client.config.default.connectTimeout: 3000
spring.cloud.openfeign.client.config.default.readTimeout: 3000
resilience4j.circuitbreaker.configs.default:
  slidingWindowSize: 10
  minimumNumberOfCalls: 5
  failureRateThreshold: 50
  waitDurationInOpenState: 5000
  permittedNumberOfCallsInHalfOpenState: 3
resilience4j.circuitbreaker.instances.doctorClient.baseConfig: default
```

---

## PROMPT 7: web-client (Thymeleaf Frontend)

```
Generate the web-client Spring Boot application — the Thymeleaf UI frontend.

### pom.xml Dependencies:
- spring-boot-starter-web
- spring-boot-starter-thymeleaf
- thymeleaf-layout-dialect (nz.net.ultraq.thymeleaf)
- spring-cloud-starter-netflix-eureka-client
- lombok
- jackson-datatype-jsr310

### Package: com.appointment.web

### IMPORTANT: This service has NO database, NO JPA, NO security.
It uses RestTemplate to call the api-gateway at http://localhost:8080 for ALL API operations.
It uses Thymeleaf templates for server-side HTML rendering.
User session is managed via HttpSession (storing token, userId, role as session attributes).

---

### Config — RestTemplateConfig:
- @Bean RestTemplate with SessionTokenInterceptor added to interceptors list.
- Register JavaTimeModule on the ObjectMapper of MappingJackson2HttpMessageConverter.

### Config — SessionTokenInterceptor (implements ClientHttpRequestInterceptor):
- In intercept(): Get the current HttpServletRequest from RequestContextHolder.
- Extract "token" from HttpSession.
- If token exists, call request.getHeaders().setBearerAuth(token).
- This automatically injects the Authorization: Bearer <token> header on ALL RestTemplate calls.

### Config — WebConfig (implements WebMvcConfigurer):
- Register AuthInterceptor for path patterns: "/patient/**", "/doctor/**", "/admin/**".

---

### Middleware — AuthInterceptor (implements HandlerInterceptor):
- Injected: RestTemplate
- preHandle() logic:
  1. If session null or no "token" → redirect to /login, return false.
  2. Extract path, role, userId from session.
  3. Role-based route checks:
     - /patient/** requires PATIENT or ADMIN
     - /doctor/** requires DOCTOR or ADMIN
     - /admin/** requires ADMIN
  4. PROFILE COMPLETION CHECK (critical feature):
     - For PATIENT role: If not on /patient/complete-profile page AND session "profileCompleted"
       is null or false → call http://localhost:8080/patients/{userId} via RestTemplate.
       If 404 → redirect to /patient/complete-profile. If success → set profileCompleted=true.
     - Same logic for DOCTOR role using http://localhost:8080/doctors/user/{userId}.

---

### DTOs (lightweight copies in web.dto package — NO validation annotations):
- LoginRequest: email, password
- RegisterRequest: name, email, password, role (UserRole enum)
- UserRole: PATIENT, DOCTOR, ADMIN
- AuthResponse: token, role (UserRole), userId
- PatientRequest: name, contactNumber, dob (LocalDate), address, medicalHistory
- PatientResponse: id, userId, name, contactNumber, dob, address, medicalHistory, createdAt
- DoctorRequest: specialization, qualification, experienceYears (Integer),
  consultationFee (BigDecimal), name, contactNumber — use @Builder
- DoctorResponse: id, userId, specialization, qualification, experienceYears,
  consultationFee, name, contactNumber
- SlotRequest: date (LocalDate), startTime (LocalTime), endTime (LocalTime)
- SlotResponse: id, date, startTime, endTime, isBooked (@JsonProperty("isBooked")),
  doctorId, isExpired (@JsonProperty("isExpired"))
- BookingRequest: patientId, doctorId, slotId, reason
- AppointmentResponse: id, patientId, doctorId, slotId, status (AppointmentStatus),
  reason, createdAt, PLUS transient display fields: patientName, doctorName,
  slotDate (String), slotTime (String)
- AppointmentStatus enum: PENDING, APPROVED, REJECTED, CANCELLED, COMPLETED

---

### Controller — AuthController (NO @RequestMapping prefix):
| Method | Path      | What it does                                                    |
|--------|-----------|-----------------------------------------------------------------|
| GET    | /         | If session has token → redirect to role-specific dashboard. Else render "index" template. |
| GET    | /login    | Show login.html with LoginRequest model attribute. Handle ?error and ?registered params. |
| POST   | /login    | POST to http://localhost:8080/auth/login. Store token/userId/role in session. Redirect to dashboard. |
| GET    | /register | Show register.html with RegisterRequest model attribute.        |
| POST   | /register | POST to http://localhost:8080/auth/register. Redirect to /login?registered=true. |
| GET    | /logout   | Invalidate session. Redirect to /login.                         |

### Controller — PatientController (@RequestMapping("/patient")):
| Method | Path              | What it does                                                    |
|--------|-------------------|-----------------------------------------------------------------|
| GET    | /dashboard        | Fetch patient profile, upcoming appointments, enrich with doctor/slot details. |
| GET    | /complete-profile | Show complete-profile.html form.                                |
| POST   | /complete-profile | POST PatientRequest to http://localhost:8080/patients. Set profileCompleted=true. |
| GET    | /search-doctors   | Fetch all doctors (optional ?specialization filter). Show search results. |
| GET    | /book/{doctorId}  | Fetch doctor details and available slots. Filter out expired slots. Show booking form. |
| POST   | /book             | POST BookingRequest to http://localhost:8080/appointments.       |
| GET    | /appointments     | Fetch patient appointments with startDate/endDate filters. Enrich with doctor/slot info. |
| GET    | /profile          | Show profile view/edit form.                                    |
| POST   | /profile          | PUT PatientRequest to http://localhost:8080/patients/{userId}.   |

### Controller — DoctorController (@RequestMapping("/doctor")):
| Method | Path                       | What it does                                               |
|--------|----------------------------|------------------------------------------------------------|
| GET    | /dashboard                 | Fetch doctor profile, appointments (enriched), slots.      |
| POST   | /appointments/{id}/status  | PUT to http://localhost:8080/appointments/{id}/status?status=X |
| GET    | /manage-slots              | Fetch slots list. Show add-slot form.                      |
| POST   | /manage-slots              | POST SlotRequest to http://localhost:8080/doctors/{id}/slots. |
| POST   | /manage-slots/edit/{slotId}| PUT SlotRequest to http://localhost:8080/doctors/slots/{slotId}. |
| POST   | /manage-slots/delete/{slotId} | DELETE to http://localhost:8080/doctors/slots/{slotId}.  |
| GET    | /complete-profile          | Show complete-profile.html form.                           |
| POST   | /complete-profile          | POST DoctorRequest to http://localhost:8080/doctors.        |
| GET    | /appointments              | Fetch doctor appointments with date filters. Enrich.       |
| GET    | /profile                   | Show profile view/edit form.                               |
| POST   | /profile                   | PUT DoctorRequest to http://localhost:8080/doctors/{id}.    |

IMPORTANT: The web-client uses form POST even for edit/delete operations because HTML forms
only support GET/POST. The controller then converts these to PUT/DELETE RestTemplate calls
to the API gateway.

### Enrichment Pattern (used in dashboard and appointments):
For each AppointmentResponse:
1. Fetch PatientResponse from http://localhost:8080/patients/profile/{patientId}
   → Set app.patientName = name + " (Contact: " + contactNumber + ")"
2. Fetch SlotResponse from http://localhost:8080/doctors/slots/{slotId}
   → Set app.slotDate and app.slotTime as formatted strings.

---

### Templates (Thymeleaf):
The following templates exist. All dashboard/inner pages use Thymeleaf Layout Dialect
with layout:decorate="~{layout}" and layout:fragment="content".

1. **layout.html** — Master layout with sidebar navigation and header:
   - Sidebar links: Dashboard, Manage Slots (doctor), Search Doctors (patient),
     My Appointments, Profile.
   - Header: Role badge, "My Profile" link, Gateway status indicator, Logout button.
   - Uses Google Fonts (Inter), custom CSS variables for colors.

2. **index.html** — Standalone landing page (does NOT use layout.html):
   - Hero section, features grid, statistics, CTA buttons to /login and /register.

3. **login.html** — Standalone login form with email/password fields.

4. **register.html** — Standalone register form with name/email/password/role dropdown.

5. **patient/dashboard.html** — Shows upcoming appointments table with doctor name, slot info.

6. **patient/complete-profile.html** — Form: name, contact, DOB, address, medical history.

7. **patient/search-doctors.html** — Specialization filter + doctor cards grid.

8. **patient/book.html** — Doctor info card + available slots list + booking form.

9. **patient/appointments.html** — Date-filtered appointments table.

10. **patient/profile.html** — View/edit patient profile form.

11. **doctor/dashboard.html** — Appointments table with approve/reject buttons + slots overview.

12. **doctor/complete-profile.html** — Form: name, contact, specialization, qualification,
    experience, consultation fee.

13. **doctor/manage-slots.html** — Two-panel layout:
    - Left: List of slot pill cards. Each shows date, time range, status badges
      (Expired in gray, Booked in amber). Non-expired/non-booked slots show Edit button.
    - Right: "Add Availability Slot" form with date, start time, end time inputs.
    - EDIT MODAL: Fixed-position overlay popup that opens when Edit is clicked.
      Pre-populates date/startTime/endTime. Has "Save Changes" and "Delete Slot" buttons.
    - CRITICAL: All date min attributes are set dynamically via JavaScript on page load,
      NOT via Thymeleaf expressions. This avoids TemplateInputException errors.
    - CRITICAL: Do NOT use ES6 template literals (backtick ${}) inside
      <script th:inline="javascript"> blocks — Thymeleaf interprets ${} as server expressions.
      Use string concatenation instead.
    - Use HTML5 data-* attributes on Edit buttons (th:data-id, th:data-date, etc.)
      and read them with element.dataset in JavaScript.

14. **doctor/appointments.html** — Date-filtered appointments table with approve/reject.

15. **doctor/profile.html** — View/edit doctor profile form.

16. **admin/dashboard.html** — Admin overview (basic).

### application.yml:
server.port: 8085
spring.application.name: web-client
spring.thymeleaf.cache: false
spring.thymeleaf.prefix: classpath:/templates/
spring.thymeleaf.suffix: .html
spring.jackson.datatype.datetime.write-dates-as-timestamps: false
eureka.client.service-url.defaultZone: http://localhost:8761/eureka/
eureka.instance.prefer-ip-address: true
```

---

## Key Gotchas & Lessons Learned

```
1. JACKSON BOOLEAN SERIALIZATION: Fields named isBooked/isExpired in DTOs with Lombok
   get serialized as "booked"/"expired" by Jackson (it strips the "is" prefix for booleans).
   Fix: Add @JsonProperty("isBooked") and @JsonProperty("isExpired") annotations.

2. THYMELEAF + ES6 TEMPLATE LITERALS: Never use backtick strings with ${variable} inside
   <script th:inline="javascript">. Thymeleaf's parser interprets ${} as server-side
   expressions and crashes with TemplateInputException. Use string concatenation instead.

3. THYMELEAF ONCLICK BRACKETS: Don't use [[${slot.id}]] inside th:onclick attributes.
   It can produce raw unquoted values. Instead, put data on HTML5 data-* attributes
   and read them from JavaScript with element.dataset.

4. DATE INPUT MIN ATTRIBUTES: Don't use th:min="${#temporals.format(...)}" for dynamic
   date constraints. Set min attributes via JavaScript on page load instead.

5. SESSION-BASED AUTH IN WEB-CLIENT: The web-client stores JWT in HttpSession (server-side),
   and a SessionTokenInterceptor on RestTemplate automatically adds the Bearer token to
   all outgoing API calls. This is NOT cookie-based JWT.

6. PROFILE COMPLETION FLOW: First-time login for PATIENT or DOCTOR must redirect to
   /patient/complete-profile or /doctor/complete-profile. The AuthInterceptor checks if
   the profile exists by calling the downstream service, and caches the result in the session.

7. SLOT EXPIRATION: isExpired is calculated dynamically on every read (not stored in DB):
   slot.date.isBefore(today) || (slot.date.isEqual(today) && LocalTime.now().isAfter(slot.endTime))

8. FEIGN CLIENT FALLBACK: DoctorClientFallback throws a custom exception rather than
   returning null, ensuring the caller gets a clear error about service unavailability.
```
