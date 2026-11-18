# Assignment 1 report: A personal AI-native page

Samin Akbari - 401105594

## 1. What I built

A web page where a visitor asks questions about me. The page answers only from a JSON record I wrote (`backend/data/record.json`) and from two public APIs. If a fact is in neither, it says it does not know. Private facts are never sent to the model.

- Backend: Python FastAPI. Frontend: plain HTML, CSS and JavaScript served by the same app.
- Model: an OpenAI-compatible chat API. The key, base URL and model name come from the environment variables `LLM_API_KEY`, `LLM_BASE_URL` and `LLM_MODEL`, not from the code or the image.

How one question is handled:

1. A short model call finds the intent. For a reading request it also picks up to 3 book topics from my courses and projects.
2. The backend loads the visible record, my public GitHub repos (GitHub API) and, for reading requests, book results (Open Library API).
3. A second model call answers as JSON: `intent`, `capability`, `fields_read`, `missing_facts`, `answer`.
4. A validator checks the result. It rejects any action outside the allowed list, drops field paths that do not exist in the sources, and replaces an answer that cites no real field with "I don't know".
5. The page shows the answer together with the reasoning (section 6).

## 2. Public address and deployment steps

- Public address: not available yet. I could not deploy the page because of temporary problems with Hamravesh, and I will deploy it as soon as they are solved.
- Local run: `cp .env.example .env`, fill it in, then `docker compose up --build` and open `http://localhost:8000`.

## 3. The record: schema and filled data

Files: `backend/data/record.schema.json` (the shape) and `backend/data/record.json` (the filled record). Every section has an `x-visible` flag in the schema. The code removes every section marked not visible before any model call or API response (`backend/app/record.py`).

| Section | Fields | Visitor may see |
|---------|--------|-----------------|
| `profile` | name, headline, location, languages[], contact (email, phone, linkedin) | yes |
| `honors[]` | id, name, year, result | yes |
| `education[]` | id, institution, degree, location, start, end | yes |
| `experience[]` | id, company, title, start, end, highlights[] | yes |
| `courses[]` | id, name (names only: no grades, terms or instructors) | yes |
| `projects[]` | id, name, date, stack[], course_ids[], summary, notes (optional) | yes |
| `skills` | programming_languages, backend, databases_and_caching, architecture_and_design, devops_and_tools | yes |
| `academic_service[]` | strings | yes |
| `limits` | wont_discuss[], cannot_do[], assumptions[], known_gaps[] | yes |
| `private` | gpa | **no** |

**Courses** (46, names only): taken from my transcript and my CV. Examples: Data Structures and Algorithms, Database Design, Compiler Design, Operating Systems, Computer Networks, Artificial Intelligence, Machine Learning, Software Engineering, Web Programming, Mobile Programming. Grades, terms and instructors are not in the record.

**Projects** (8):

| id | Date | Stack |
|----|------|-------|
| la-noire | 2025-12 | Django, DRF, django-filter, PyJWT, PostgreSQL, Docker |
| lost-and-found | 2025-12 | Django, DRF, PostgreSQL, SQLite, Docker |
| rabbitmq | 2026-03 | FastAPI, SQLite |
| link-shortener | 2026-04 | FastAPI, SQLite |
| database-project | 2024-05 | SQL |
| computer-structure-languages | 2024-01 | IBM Assembly, x86 Assembly |
| mobile-exercises | 2024-04 | Java, XML |
| stronghold-crusaders | 2024-04 | Java, JavaFX |

**Limits:**
- Will not discuss: grades and GPA, salary expectations, personal life and other people, opinions about the university or past employers.
- Cannot do: send email, book meetings, speak for Sharif University of Technology or Triboon, make hiring or salary commitments.
- Assumption: anything I know is in the record or on my GitHub, so an unlisted skill or tool is treated as not known.
- Known gaps: no machine-learning project, no frontend framework, no cloud certification, no per-course results listed.

The record has no GitHub data. Repo facts come from the GitHub API at question time.

## 4. Capabilities, authority and guardrails

The model may take only these actions, and the validator enforces the list:

| Action | Meaning |
|--------|---------|
| `answer_from_record` | answer with facts from the record |
| `answer_from_external_api` | answer with GitHub or Open Library facts (and the record) |
| `refuse_unpublished_fact` | private fact: say it is not shared; anything else not listed: say "I don't know" |
| `out_of_scope` | tell the visitor the question is outside this page (unrelated topics, or requests the page may not do) |
| `ask_clarifying_question` | ask exactly one question |

Every question is judged by what I know. If the answer is in my record or on my GitHub the page answers. If it is private the page says I do not share it. If it is about me but not listed, it says it does not know. A question unrelated to me, such as general knowledge, is marked outside this page and is never answered from the model's own knowledge.

Authority: the page may talk about me. It cannot send email, book meetings, or speak for Sharif University of Technology. Anything unlisted counts as not known, and grades or GPA are never shared. These limits are written on the page and are in the action list.

## 5. Other apps and APIs

- **GitHub REST API**: public repos of `saminakbari` (description, main language, last push, stars, link). Used for project questions and repo links.
- **Open Library search API**: book results for the topics taken from my courses and projects. The page only suggests titles that this API returned.
- **LLM API**: the model that reads the sources and writes the answer.

Each answer lists the services it used. The list is computed from the fields actually read, not from what the model claims.

## 6. UI, explainability and creativity

- **UI**: a question box with example buttons, an answer card, a "Why this answer" panel, and a box with the allowed actions and limits.
- **Explainability**: the answer card shows the intent and capability. The panel shows the intent, the capability, the services used, the fields read and the facts that were not available. Clicking a field shows its exact value.
- **Creativity** (three additions of my own):
  1. **Job-fit meter**: for a job description, a table with one row per requirement, "Matched" only if a real record or GitHub field backs it, otherwise "Not listed", plus a bar showing how many are backed.
  2. **Timeline**: education, work and projects in date order, built only from the record. Clicking a project asks for its summary.
  3. **Evidence panel and badge**: the click-to-see-value check, and a badge saying whether the answer came from the record, an external API, or was a refusal or a question.

## 7. Three traces

### Trace 1: good answer

- Question: Summarize the Link-Shortener project.
- Intent: project_summary
- Fields read: projects[link-shortener].summary, projects[link-shortener].stack, github[link-shortener].url, github[link-shortener].primary_language, github[link-shortener].last_push
- Capability: answer_from_external_api
- Answer: Link-Shortener is a URL-shortening service built with FastAPI and SQLite. It uses Domain-Driven Design and CQRS to separate reads and writes, with event-driven asynchronous communication between components to support scalability and maintainability. GitHub: https://github.com/saminakbari/link-shortener. Primary language: Python. Last pushed June 16, 2026.

### Trace 2: refusal

- Question: What is Samin's GPA?
- Intent: other
- Fields read: limits.wont_discuss
- Capability: refuse_unpublished_fact
- Answer: Samin's GPA is not shared.

### Trace 3: clarifying question

- Question: Tell me about your project.
- Intent: project_summary
- Fields read: projects
- Capability: ask_clarifying_question
- Answer: Which project would you like to hear about: L.A. Noire, Lost&Found, RabbitMQ, Link-Shortener, Database Project, Computer Structure and Languages Project, Mobile Programming Exercises, or Stronghold Crusaders?

## 8. Notes

- Asking more than 20 questions per minute from one address is rate limited.
