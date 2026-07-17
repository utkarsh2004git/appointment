# API Endpoints Reference — Online Appointment Management System

All external traffic enters through the **API Gateway** at `http://localhost:8080`. The paths below are what you call **through the gateway** (add the gateway's base URL in front). Direct service ports are listed too, for local debugging with Postman when you want to bypass the gateway during development.

Every endpoint except `/auth/register` and `/auth/login` requires a header:
```
Authorization: Bearer <jwt>
```
The gateway validates this and forwards `X-User-Id` and `X-User-Role` to the backend services — you don't send those headers yourself, the gateway adds them.

---

## Auth Service
Direct port: `8081` · Gateway prefix: `/auth/**`

| Method | Endpoint | Auth required | Request body | Response | Description |
|---|---|---|---|---|---|
| POST | `/auth/register` | No (public) | `{ "name": "string", "email": "string", "password": "string", "role": "PATIENT \| DOCTOR" }` | `201 Created` — created user summary | Registers a new user, hashes password with BCrypt |
| POST | `/auth/login` | No (public) | `{ "email": "string", "password": "string" }` | `200 OK` — `{ "token": "string", "role": "string", "userId": number }` | Verifies credentials, issues a JWT (1 hour expiry) |

---

## Patient Service
Direct port: `8082` · Gateway prefix: `/patients/**`

| Method | Endpoint | Auth required | Request body | Response | Description |
|---|---|---|---|---|---|
| POST | `/patients` | Yes — PATIENT or ADMIN | `{ "dob": "YYYY-MM-DD", "address": "string", "medicalHistory": "string", "emergencyContact": "string" }` (userId is NOT sent in the body — it's derived from the X-User-Id header the Gateway sets from the caller's JWT) | `201 Created` — Patient profile | Creates a patient profile for the currently authenticated user |
| GET | `/patients/{userId}` | Yes — owner (matching X-User-Id) or ADMIN | — | `200 OK` — Patient profile | Fetches a patient's profile |
| PUT | `/patients/{userId}` | Yes — owner or ADMIN | Same shape as POST | `200 OK` — updated Patient profile | Updates a patient's profile |

---

## Doctor Service
Direct port: `8083` · Gateway prefix: `/doctors/**`

| Method | Endpoint | Auth required | Request body | Response | Description |
|---|---|---|---|---|---|
| POST | `/doctors` | Yes — DOCTOR or ADMIN | `{ "specialization": "string", "qualification": "string", "experienceYears": number, "consultationFee": number }` (userId is NOT sent in the body — it's derived from the X-User-Id header) | `201 Created` — Doctor profile | Creates a doctor profile for the currently authenticated user |
| GET | `/doctors?specialization={value}` | Yes — any authenticated role | — (query param) | `200 OK` — list of Doctor profiles | Searches doctors by specialization (omit param for all doctors) |
| GET | `/doctors/{id}` | Yes — any authenticated role (also called internally via Feign) | — | `200 OK` — Doctor profile | Fetches one doctor by their Doctor entity id |
| GET | `/doctors/by-user/{userId}` | Yes — owner or ADMIN | — | `200 OK` — Doctor profile | Looks up a doctor's own profile by their userId (used by web-client after login) |
| POST | `/doctors/{id}/slots` | Yes — DOCTOR (owner) or ADMIN | `{ "date": "YYYY-MM-DD", "startTime": "HH:mm", "endTime": "HH:mm" }` | `201 Created` — Slot | Adds a new availability slot for a doctor |
| GET | `/doctors/{id}/slots` | Yes — any authenticated role | — | `200 OK` — list of available (unbooked) Slots | Lists a doctor's open slots |
| GET | `/doctors/slots/{slotId}` | Internal (called by appointment-service via Feign) | — | `200 OK` — Slot | Fetches a single slot by id |
| PUT | `/doctors/slots/{slotId}/book` | Internal (called by appointment-service via Feign) | — | `200 OK` — updated Slot (isBooked=true) | Marks a slot as booked; has a Resilience4j fallback in the caller |

---

## Appointment Service
Direct port: `8084` · Gateway prefix: `/appointments/**`

| Method | Endpoint | Auth required | Request body | Response | Description |
|---|---|---|---|---|---|
| POST | `/appointments` | Yes — PATIENT | `{ "patientId": number, "doctorId": number, "slotId": number, "reason": "string" }` | `201 Created` — Appointment (status PENDING) | Books an appointment; calls doctor-service via Feign to verify + mark the slot booked |
| PUT | `/appointments/{id}/status` | Yes — DOCTOR (for APPROVED/REJECTED) or PATIENT (for CANCELLED, owner only) | `{ "status": "APPROVED \| REJECTED \| CANCELLED \| COMPLETED" }` | `200 OK` — updated Appointment | Changes an appointment's status |
| GET | `/appointments/patient/{patientId}` | Yes — owner or ADMIN | — | `200 OK` — list of Appointments | All appointments for a given patient |
| GET | `/appointments/doctor/{doctorId}` | Yes — owner (matching doctor) or ADMIN | — | `200 OK` — list of Appointments | All appointments for a given doctor |

---

## API Gateway (routing only, not a business API)
Port: `8080`

| Path prefix | Routes to | Notes |
|---|---|---|
| `/auth/**` | auth-service | JWT check skipped for `/auth/login`, `/auth/register` |
| `/patients/**` | patient-service | JWT required |
| `/doctors/**` | doctor-service | JWT required |
| `/appointments/**` | appointment-service | JWT required |

---

## web-client (Thymeleaf, not a REST API — page routes for reference)
Port: `8085`

| Method | Route | Who can access | Calls (via gateway) |
|---|---|---|---|
| GET/POST | `/login` | Public | `/auth/login` |
| GET/POST | `/register` | Public | `/auth/register` |
| GET | `/logout` | Any logged-in session | — (clears session) |
| GET | `/patient/dashboard` | PATIENT | `/appointments/patient/{userId}` |
| GET | `/patient/search-doctors` | PATIENT | `/doctors?specialization=` |
| GET/POST | `/patient/book/{doctorId}` | PATIENT | `/doctors/{id}/slots`, `/appointments` |
| POST | `/patient/appointments/{id}/cancel` | PATIENT (owner) | `/appointments/{id}/status` |
| GET | `/doctor/dashboard` | DOCTOR | `/appointments/doctor/{doctorId}` |
| POST | `/doctor/appointments/{id}/approve` | DOCTOR | `/appointments/{id}/status` |
| POST | `/doctor/appointments/{id}/reject` | DOCTOR | `/appointments/{id}/status` |
| GET/POST | `/doctor/manage-slots` | DOCTOR | `/doctors/{id}/slots` |
| GET | `/admin/dashboard` | ADMIN | `/doctors`, appointment counts |

---

## Quick Postman testing order

1. `POST /auth/register` (role PATIENT) → then again for a doctor (role DOCTOR)
2. `POST /auth/login` with the patient's credentials → copy the token
3. `POST /patients` with the patient's userId, using the token
4. `POST /auth/login` with the doctor's credentials → copy that token
5. `POST /doctors` with the doctor's userId, using the doctor's token
6. `POST /doctors/{id}/slots` to add availability
7. Switch back to the patient's token → `GET /doctors?specialization=` → `POST /appointments` with the returned slotId
8. Switch to the doctor's token → `PUT /appointments/{id}/status` with `APPROVED`
9. `GET /appointments/patient/{id}` to confirm the status updated
