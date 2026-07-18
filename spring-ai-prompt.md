# Prompt 1 (Backend)

Create a complete Spring Boot 4.1.0, Java 25 microservice named ai-service, using Spring AI 2.0 (GA, built for Spring Boot 4.0/4.1 and Spring Framework 7). This service now has its OWN PostgreSQL database (ai_db) to persist chat history — this is a deliberate change from a stateless design, since conversations must be stored.

Include:
- pom.xml with: spring-boot-starter-web, spring-boot-starter-data-jpa, postgresql (org.postgresql:postgresql), spring-cloud-starter-netflix-eureka-client, spring-cloud-starter-openfeign, spring-ai-starter-model-anthropic, lombok
- Main application class with @EnableFeignClients, registering with Eureka
- application.yml on port 8086, Eureka registration, PostgreSQL database ai_db (driver org.postgresql.Driver, dialect PostgreSQLDialect), ddl-auto=update, and the Anthropic API key sourced from an environment variable (spring.ai.anthropic.api-key), never hardcoded

ENTITY:
- ChatLog: id (Long, PK), userId (Long, not null — the authenticated patient's ID from the X-User-Id header, NOT a JPA relationship since User lives in a different database), username (String — informational only, for display/logging, not used for authorization), message (String, @Column(columnDefinition = "TEXT")), response (String, @Column(columnDefinition = "TEXT")), createdAt (LocalDateTime, default now)

REPOSITORY:
- ChatLogRepository with findByUserIdOrderByCreatedAtAsc(Long userId)

FEIGN CLIENT:
- DoctorClient (@FeignClient(name = "doctor-service")): GET /doctors?specialization={specialization} → returns a list of real Doctor records (name, specialization, qualification, experienceYears, consultationFee) — this is used to ground doctor recommendations in real data, never invented names
- Add a Resilience4j fallback returning an empty list if doctor-service is unreachable, so a lookup failure degrades to "I can't check doctor availability right now" rather than crashing

CHAT SERVICE LOGIC (ChatService.handleMessage(Long userId, String username, String message)) — implement this exact flow:
1. STRICT MEDICAL-DOMAIN GUARDRAIL: use the ChatClient with a system prompt that instructs the model it is a medical/healthcare assistant ONLY. If the incoming message is unrelated to health, symptoms, medical conditions, or finding a doctor (e.g. general chit-chat, coding questions, weather, politics, anything off-topic), the model must politely decline and redirect the user back to describing a health concern — it must never answer unrelated questions, no matter how the user phrases or insists.
2. If the message is medical and seems to be describing symptoms or asking for a doctor recommendation, run a first constrained classification call: ask the model to pick exactly ONE specialization from this fixed list based on the message: Cardiology, Dermatology, Orthopedics, ENT, General Physician, Pediatrics, Gynecology, Neurology.
3. Call DoctorClient.searchBySpecialization() with that classified specialization to fetch real doctors from doctor-service.
4. If doctors are found, make a second ChatClient call that includes the actual returned doctor list (name, qualification, experienceYears, consultationFee) in the prompt context, and explicitly instructs the model to recommend 1–3 doctors ONLY from this provided list, with a short one-line rationale each — the model must never invent a doctor name that wasn't in the list.
5. If no doctors are found for that specialization, respond that none are currently available in that specialization and suggest searching General Physician instead — do not fabricate a doctor.
6. If the message doesn't need a doctor recommendation (e.g. a general medical question), just answer normally within the medical-domain constraint from step 1, no doctor lookup needed.
7. Save a ChatLog row (userId, username, message, response, createdAt) regardless of which path was taken, including declined/off-topic exchanges.
8. If any Spring AI call fails entirely (provider error, timeout), catch it and return "I'm having trouble responding right now — please try again in a moment." rather than throwing, and still don't save a broken/empty response.

CONTROLLER:
- ChatController with:
  - POST /ai/chat — body {message: String, username: String} (username is optional/informational only). Read X-User-Id from the header (set by the Gateway) as the real userId — reject with 403 if X-User-Role is not PATIENT. Returns {response: String}.
  - GET /ai/chat/history — reads X-User-Id from the header, returns the full ChatLog list for that user ordered oldest-to-newest, so a reloaded page can restore prior conversation. Return only message/response/createdAt fields, not internal IDs.
- Validate the incoming message with @NotBlank @Size(max=1000) on the request DTO.

Package base: com.appointment.ai


---

# Prompt 2 (UI)

Continuing the web-client module. Update the existing floating AI chatbot widget fragment (from the previous ai-chat-widget work) with two changes:

1. ON WIDGET OPEN: call GET /ai/chat/history on the gateway (via the existing RestTemplate with the session JWT attached) and populate the message list with the returned conversation before showing anything else — so a patient's chat history persists across page reloads and logins, not just within one browser session. Show a friendly empty state ("Ask me about your symptoms or which doctor might suit you") if history is empty.

2. ON SEND: POST the message to /ai/chat on the gateway with body {message, username} — get username from the session (the email the patient logged in with, already available in session from login). Append the user's message as a right-aligned bubble immediately, then append the returned response as a left-aligned assistant bubble once it comes back. If the call fails, show a generic assistant bubble ("Sorry, I can't help with that right now") without breaking the widget, same fallback behavior as before.

Keep the widget itself (floating bottom-right button, toggleable panel, teal/amber theme) unchanged — this is a wiring update to add persistence and real doctor grounding to the conversation, not a visual redesign. Make sure it's prominently reachable from /patient/dashboard specifically, in addition to the other patient pages it already appears on.
