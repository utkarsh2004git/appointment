Act as a technical writer creating a formal project case study document for a college/corporate training submission. Write a complete, structured Word document titled "[YOUR PROJECT NAME] — High-Level Project Planning Document" with the following sections, in this order:

1. Executive Summary — 3 short paragraphs summarizing what the project is, why it was built this way, and how this document is organized.
2. Project Overview — what the system does, its objectives as a bulleted list, and a technology stack table (Layer | Technology) covering every framework, language, database, and tool used.
3. Problem Statement — describe the real-world problem this project solves, the limitations of the current/manual approach, and how the proposed system addresses each limitation.
4. User Roles — a table of every user role in the system with their description and key capabilities.
5. Core Features — grouped by user role, as bulleted lists under each role's own subheading, plus a "Platform & Security Features" subsection for cross-cutting capabilities.
6. Non-Functional Requirements — a table covering Security, Scalability, Availability/Resilience, Performance, Maintainability, and Usability requirements.
7. Architecture Overview — describe the system's architecture (monolith, microservices, layered, etc.), including a services/components table (Component | Responsibility | Key Technology) and a description of how security, data flow, and inter-component communication work.
8. Database Design — for each major data-owning component, list its entities and fields in a table, and clearly state which relationships are enforced at the database level versus which are just referenced by ID across boundaries.
9. Core Workflow — describe the primary end-to-end user journey as a numbered sequence of steps, naming which component is responsible at each step.
10. API/Interface Planning — grouped by service or module, list every endpoint or interface with its method/trigger, required authorization, and a one-line description.
11. CI/CD & Quality Overview — describe the build, test, static analysis, and deployment pipeline stages and what quality gates are enforced.
12. Prompt Engineering / AI-Assisted Development Methodology — since this project was built using an AI coding agent, document: (a) how prompts were structured and sequenced, (b) at least one concrete before/after example where refining a prompt fixed an incorrect or outdated AI output, and (c) how iteration and context were managed across multiple AI sessions.
13. Challenges & Lessons Learned — a table pairing each significant challenge with how it was addressed.
14. Future Enhancements — a bulleted list of scoped-out features or improvements for a future version.
15. Conclusion — a closing paragraph tying the technical outcome back to the project's original objectives.

Formatting requirements: use proper Word heading styles (Heading 1 for main sections, Heading 2 for subsections) so a Table of Contents can be auto-generated, use real Word tables (not markdown-style text tables) wherever a table is specified above, include a title page before Section 1, and keep prose paragraphs concise (3-5 sentences) — this is a technical planning document, not a narrative essay.

I will provide the specific project details (name, tech stack, entities, features, and API endpoints) in my next message — confirm you understand this structure first, then wait for that information before writing.
