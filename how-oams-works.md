# How OAMS Works — Entities, Roles, and UI Flow Explained

This is a plain-language walkthrough of the mechanics: which entity connects to which and how, what actually happens when someone signs up, and what the user sees on screen at each step. Read this alongside the Project Planning Document — that one describes *what* gets built, this one explains *how the pieces move together*.

---

## 1. The Core Mental Model

Every entity in this system lives in exactly one service's database. Nothing is ever joined across services at the database level — instead, one entity holds the plain numeric ID of another entity that lives elsewhere, and if a service needs the actual data behind that ID, it asks the owning service directly over the network (via OpenFeign).

Think of it like five separate spreadsheets kept by five separate people. If the "Appointments" spreadsheet needs to know a doctor's name, it doesn't get to open the "Doctors" spreadsheet itself — it has to ask the person holding it. That's the entire reason cross-service calls (Feign) exist.

---

## 2. Entity Relationship Map

### 2.1 Real relationships (within the same database)

Only **one** relationship in the whole system is a real, database-enforced JPA relationship — because it's the only pair of entities that live in the same database:

```
doctor_db
┌──────────────┐        ┌──────────────┐
│    Doctor     │ 1 ──── * │     Slot     │
│  id           │        │  id           │
│  userId       │        │  doctorId (FK)│
│  specialization│        │  date         │
│  qualification │        │  startTime    │
│  experienceYears│       │  endTime      │
│  consultationFee│       │  isBooked     │
└──────────────┘        └──────────────┘
```

A `Doctor` has many `Slot`s. Delete a doctor, and (with cascade configured) their slots go with them. This is a normal, boring, textbook `@OneToMany` / `@ManyToOne` pair — nothing unusual here.

### 2.2 Logical relationships (across services, no database link)

Everything else is a **plain ID field**, not a JPA relationship. The database has no idea these numbers mean anything to each other — the meaning only exists in application code.

```
auth_db                patient_db              doctor_db             appointment_db
┌──────────┐            ┌──────────┐            ┌──────────┐          ┌──────────────┐
│   User    │◄─ ─ ─ ─ ─ │ Patient   │            │  Doctor   │◄─ ─ ─ ─ │  Appointment  │
│ id        │  (userId) │ id        │            │  id       │(doctorId)│ id            │
│ name      │           │ userId    │            │  userId   │         │ patientId     │
│ email     │◄─ ─ ─ ─ ─ ┼───────────┤            │           │         │ doctorId      │
│ password  │  (userId) │ dob       │            │           │◄─ ─ ─ ─ ┼ slotId        │
│ role      │           │ address   │            │           │(via Slot)│ status        │
└──────────┘            └──────────┘            └──────────┘          │ reason        │
                                                        ▲               └──────────────┘
                                                        │                      │
                                                   ┌──────────┐                │
                                                   │   Slot    │◄─ ─ ─ ─ ─ ─ ─ ┘
                                                   │ id        │   (slotId)
                                                   │ doctorId  │
                                                   │ isBooked  │
                                                   └──────────┘
```

The dashed arrows (`◄─ ─ ─`) are *not* foreign keys. They're just a number that happens to match. Here's what each one means in practice:

| From | Field | Points to | How it's actually resolved |
|---|---|---|---|
| Patient.userId | Long | Auth Service's User.id | Not resolved automatically — set once at profile creation time, trusted afterward via the JWT session. |
| Doctor.userId | Long | Auth Service's User.id | Same as above. |
| Appointment.patientId | Long | Patient Service's Patient.id (or userId, depending on what the frontend sends) | Taken from the logged-in patient's session when they book — not independently re-verified against Patient Service in the current scope. |
| Appointment.doctorId | Long | Doctor Service's Doctor.id | Verified via an OpenFeign call to Doctor Service **at booking time** — this one *is* actively checked, because the booking must confirm the doctor and slot really exist and are available. |
| Appointment.slotId | Long | Doctor Service's Slot.id | Same Feign call — Appointment Service asks Doctor Service "is this slot still open?" and "mark it booked" before it saves anything. |

This is the single most important design idea in the whole system: **only the reference that actually needs to be trustworthy at the moment of use gets actively verified.** The Appointment→Doctor/Slot link is checked live because a double-booking bug would be a real, visible failure. The Patient→User link isn't re-checked on every request because the JWT already proved who's logged in — re-checking it would be redundant work for no added safety.

---

## 3. User Creation & Role Allocation — Step by Step

This is the part that trips people up, because "creating a user" and "creating a patient/doctor profile" are two *separate* steps, in two *separate* services, and that's intentional.

### Step 1 — Registration creates a `User`, and only a `User`

When someone fills out the registration form, this happens:

1. The form (on web-client) sends `{ name, email, password, role }` to `POST /auth/login`... actually `POST /auth/register`, through the Gateway, to **Auth Service**.
2. Auth Service hashes the password with BCrypt and saves a row in `auth_db.User` — with whatever `role` the person picked at signup: `PATIENT` or `DOCTOR`. (Admin accounts aren't self-service — there's no "sign up as Admin" option; that's a deliberate gap, see note below.)
3. **At this point, that's it.** No `Patient` row and no `Doctor` row exists yet — just a `User` with a role label attached. The role lives permanently on that User record from here on.

### Step 2 — Login issues a JWT carrying that role

1. The person logs in with `POST /auth/login`.
2. Auth Service checks the password, then builds a JWT containing three claims: `sub` (the User's id), `email`, and `role` (copied straight from the User row).
3. That JWT is handed back to web-client, which stores it in the browser session. From this point on, **the role is baked into the token** — nothing has to look the role up again for the rest of that session.

### Step 3 — Role allocation "gates" what happens next, but doesn't create anything by itself

The role on the JWT is what decides which dashboard the person lands on after login (`/patient/dashboard` vs `/doctor/dashboard`), but it does **not** automatically create a Patient or Doctor profile. That's a separate, explicit action:

- A person with role `PATIENT` who logs in for the first time will typically be prompted (or the frontend will just quietly do it) to create their `Patient` profile — `POST /patients` with their `userId` from the JWT, plus DOB/address/etc. Until that exists, they have an *account* but no *profile*.
- A person with role `DOCTOR` similarly needs a `Doctor` profile created — `POST /doctors` with their `userId`, specialization, fee, etc. — before they can add slots or appear in patient searches.

So the real sequence for a doctor is: **register (User, role=DOCTOR) → login (get JWT) → create Doctor profile (now searchable) → add slots.** Four separate steps across two services, tied together only by the same `userId` number.

### A note on the Admin role

There's no self-registration path to become an Admin in the current design — if you want an Admin account, the simplest approach is inserting one directly into `auth_db.User` with `role = ADMIN` (or a one-off seed/migration script), since exposing "register as Admin" as a public form field would let anyone grant themselves admin access. This is worth mentioning explicitly if your report gets asked "how does someone become an Admin?"

---

## 4. How the UI Reflects All of This

### 4.1 Session, not just a token

web-client is a traditional server-rendered app, not a JavaScript SPA, so it doesn't hand the JWT to the browser to manage. Instead:

1. After login, the JWT + role + userId are stored in the **HttpSession** on the web-client server itself.
2. Every subsequent page request from that browser carries a session cookie; web-client reads the JWT back out of the session and attaches it as the `Authorization: Bearer` header on any call it makes to the Gateway.
3. The browser itself never sees the raw JWT — it only ever sees an opaque session cookie. This is why an interceptor (not JavaScript) is what protects pages.

### 4.2 What decides what you see

| Session state | What happens when you visit a page |
|---|---|
| No session at all | `AuthInterceptor` redirects any `/patient/**`, `/doctor/**`, `/admin/**` request straight to `/login`. |
| Logged in as PATIENT, visiting `/doctor/dashboard` | Rejected/redirected — the role in session doesn't match the path prefix, even though they're authenticated. |
| Logged in as PATIENT, visiting `/patient/dashboard` | Allowed. The navbar fragment renders only Patient-relevant links (Search, My Appointments, Logout). |
| Logged in as DOCTOR | Same idea — navbar shows Dashboard/Manage Slots instead, and `/patient/**` is off-limits. |

The navbar itself is a single shared Thymeleaf fragment that branches on the session's role attribute (`th:if="${session.role == 'PATIENT'}"` style logic) — there's one navbar template, not three separate ones.

### 4.3 A concrete walkthrough — Sarah books with Dr. Mehta

To make all of the above concrete, here's an actual path through the system with made-up but realistic IDs:

1. **Sarah registers** → `POST /auth/register` → Auth Service creates `User{id: 101, email: sarah@mail.com, role: PATIENT}`.
2. **Sarah logs in** → `POST /auth/login` → JWT issued with `sub: 101, role: PATIENT`. web-client stores this in session and redirects Sarah to `/patient/dashboard`.
3. **Sarah's dashboard is empty** (no appointments yet) since no `Patient` profile or `Appointment` exists for her yet. She fills in her profile → `POST /patients {userId: 101, dob: ..., address: ...}` → Patient Service creates `Patient{id: 55, userId: 101, ...}`.
4. **Dr. Mehta** went through the same registration/login dance earlier, ending up with `User{id: 77, role: DOCTOR}` and then `Doctor{id: 12, userId: 77, specialization: "Dermatology"}`, plus a `Slot{id: 900, doctorId: 12, date: 2026-07-20, startTime: 10:00, isBooked: false}` he added himself.
5. **Sarah searches** "Dermatology" → `GET /doctors?specialization=Dermatology` → Doctor Service returns Dr. Mehta's profile (Doctor id 12).
6. **Sarah views his slots** → `GET /doctors/12/slots` → returns Slot 900 as available.
7. **Sarah books it** → `POST /appointments {patientId: 55, doctorId: 12, slotId: 900, reason: "skin rash"}`. Appointment Service calls Doctor Service via Feign: "is slot 900 still open?" → yes → "mark it booked" → Doctor Service flips `Slot{900}.isBooked = true`. Appointment Service then saves `Appointment{id: 400, patientId: 55, doctorId: 12, slotId: 900, status: PENDING}`.
8. **Dr. Mehta logs in**, sees the pending request on his dashboard (`GET /appointments/doctor/12`), and approves it → `PUT /appointments/400/status {status: APPROVED}`.
9. **Sarah refreshes her dashboard** → `GET /appointments/patient/55` → sees Appointment 400 now shows `APPROVED`, rendered as a green status pill.

Every number in that story (101, 55, 12, 77, 900, 400) is a plain ID sitting in a different database than the one reading it — and the only two moments any service actually *reached across* to check one was real were step 7 (verify + reserve the slot) and the implicit trust of the JWT at every step in between.

---

## 5. Quick Reference — "Which service owns this?"

| If you're wondering about... | It lives in... |
|---|---|
| Login credentials, password, role | Auth Service (`auth_db.User`) |
| A patient's medical history / address | Patient Service (`patient_db.Patient`) |
| A doctor's specialization / fee | Doctor Service (`doctor_db.Doctor`) |
| Whether a specific time slot is free | Doctor Service (`doctor_db.Slot`) |
| Whether an appointment was approved | Appointment Service (`appointment_db.Appointment`) |
| What page you're currently allowed to see | web-client's HttpSession (not a database at all) |
| The suggested specialization from symptoms | AI Service (stateless — nothing is stored) |
