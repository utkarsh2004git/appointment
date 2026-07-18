# AI Agent Generation Prompts for Online Appointment Microservices

This document contains a structured set of instructions and technical specifications that you can copy and provide to any AI coding assistant to generate the exact microservice codebases implemented in this workspace.

---

## Part 1: Overall System Architecture & Security Baseline

Provide this prompt to set up the system-wide context:

```markdown
You are generating a secure Online Appointment Management System built on Spring Boot microservices. 

### Architecture Specifications:
1. Service Discovery: Netflix Eureka Discovery Client registered under local Spring Cloud Eureka Server.
2. Security Model: Downstream microservices must NOT check sessions or DB tokens directly. All requests pass through an API Gateway which authenticates and injects headers:
   - `X-User-Id` (the authenticated user's ID)
   - `X-User-Role` (the authenticated user's role: PATIENT, DOCTOR, or ADMIN)
3. Authorization Rules: Downstream services must validate that the `X-User-Id` matches the owner of the resource being modified or accessed, unless `X-User-Role` is `ADMIN`.
4. Technology Stack: Spring Boot, Spring Web, Spring Data JPA, H2 database (or PostgreSQL), Lombok, Validation API.
```

---

## Part 2: Generating `patient-service`

```markdown
Please generate the `patient-service` microservice for the Online Appointment Management System.

### Requirements:
1. **Entity Schema (`Patient`)**:
   - `id` (Long, auto-generated PK)
   - `userId` (Long, maps to authentication service user id)
   - `name` (String, required)
   - `contactNumber` (String, required)
   - `dateOfBirth` (LocalDate)
   - `address` (String)
   - `medicalHistory` (String)
   - Note: Do NOT include `emergencyContact`.

2. **Endpoints (in `PatientController`)**:
   - `POST /patients`: Complete profile (requires body of `PatientRequest`).
   - `GET /patients/{userId}`: Retrieve patient details by authentication userId.
   - `PUT /patients/{userId}`: Update patient details (requires body of `PatientRequest`).

3. **Security Headers Validation**:
   - Every read or write endpoint must parse `@RequestHeader("X-User-Id") String headerUserId` and `@RequestHeader("X-User-Role") String headerRole`.
   - Validate that `headerUserId` equals the target patient's `userId`, unless the role is `ADMIN`. Return `403 Forbidden` on mismatch.
```

---

## Part 3: Generating `doctor-service`

```markdown
Please generate the `doctor-service` microservice for the Online Appointment Management System.

### Requirements:
1. **Doctor Schema**:
   - `id`, `userId`, `name`, `contactNumber`, `specialization`, `qualification`, `experienceYears`, `consultationFee`.
2. **Availability Slot Schema (`Slot`)**:
   - `id`, `date` (LocalDate), `startTime` (LocalTime), `endTime` (LocalTime), `isBooked` (boolean).
   - Dynamic transient boolean `isExpired` which returns true if:
     `date.isBefore(today) || (date.isEqual(today) && LocalTime.now().isAfter(endTime))`

3. **Slot Operations**:
   - `POST /doctors/{doctorId}/slots`: Add slot. Throw error if start time is in the past (allow a 15-minute clock-drift buffer for slot start times on the current day).
   - `PUT /doctors/slots/{slotId}`: Edit slot. Apply same clock-drift past-time validations.
   - `DELETE /doctors/slots/{slotId}`: Delete slot if it is not booked. Return `400 Bad Request` if already booked.

4. **Authorization**:
   - Validate `X-User-Id` headers match the target doctor's `userId` for adding, editing, or deleting slots.
```

---

## Part 4: Generating `appointment-service`

```markdown
Please generate the `appointment-service` microservice.

### Requirements:
1. **Appointment Schema**:
   - `id`, `patientId`, `doctorId`, `slotId`, `date` (LocalDate), `startTime`, `endTime`, `status` (PENDING, APPROVED, REJECTED).
2. **Business Rules**:
   - When booking (`POST /appointments`), fetch the slot details from `doctor-service` first.
   - Reject booking if the slot is expired or already booked.
   - Update slot status in `doctor-service` to booked on success.
3. **Endpoints**:
   - `POST /appointments`: Book appointment.
   - `GET /appointments/patient/{patientId}`: View patient appointments.
   - `GET /appointments/doctor/{doctorId}`: View doctor consultations.
   - Both filters must support optional parameters: `startDate` and `endDate` for range filtering.
   - `PUT /appointments/{appointmentId}/status`: Approve or reject appointment.
```

---

## Part 5: Generating `web-client` (Frontend App)

```markdown
Please generate the Thymeleaf-based `web-client` frontend.

### Requirements:
1. **Auth & Redirection Interceptor (`AuthInterceptor`)**:
   - Checks if user details are present in HTTP session (`token`, `userId`, `role`).
   - For logged-in users, check if they have completed their profile in the database by calling `patient-service` or `doctor-service`.
   - If profile is not complete, redirect to `/patient/complete-profile` or `/doctor/complete-profile` and block access to any dashboard or slot management paths.

2. **Dashboard & Booking Views**:
   - **Index page (`/`)**: A modern landing page using custom styling (Teal/Gold design) with a hero illustration and navigation options.
   - **Doctor Slots Screen (`/doctor/manage-slots`)**:
     - Left: list of slots displaying an explicit "Expired" badge if `isExpired` is true. Remove "Edit" button for expired or booked slots.
     - Right: static "Add Availability Slot" form.
     - Click on "Edit" on a slot pops open a fixed modal overlay containing edit forms (pre-populated) and a red "Delete Slot" button side-by-side with "Save Changes".
     - Date limits must be initialized dynamically on load via JavaScript.
     - Do NOT use ES6 backtick strings with `${}` inside `<script th:inline="javascript">` as they conflict with Thymeleaf's expression parser.

3. **Appointments filter**:
   - Patient and Doctor dashboards must include a "My Appointments" sub-page with inline Date Pickers (`startDate`, `endDate`) to dynamically filter consultations.
```
