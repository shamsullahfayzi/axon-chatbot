# Axon Chatbot — Progress and Build Order

## Project goal

Build an internal policy RAG chatbot that answers authorized employees using organizational documents, provides traceable citations, and acknowledges missing evidence. Prioritize correctness, security, maintainability, and simplicity.

## Working agreement

- The developer writes code, runs commands, tests, and makes architectural decisions.
- Codex provides mentoring, focused implementation guidance, debugging support, and reviews.
- File changes and command execution require specific authorization, except necessary read-only inspection.
- This file was created with explicit authorization. Future edits follow the mentoring agreement unless separately authorized.

## Completed milestones

- Created `D:\axon-chatbot`.
- Initialized Git.
- Established the project goal and mentoring workflow.

## Current milestone

Milestone 1: write a short fictional policy and three questions with expected answers. These will be our first checks of chatbot behavior.

## Architecture decisions

Starting architecture, subject to review before implementation:

- Frontend: Next.js, TypeScript, Tailwind CSS, shadcn/ui.
- Backend: NestJS modular monolith with REST APIs and DTO validation.
- Database: PostgreSQL with pgvector; Prisma where suitable and parameterized SQL for vector operations.
- AI: cloud LLM API and a dedicated embedding model.
- Deployment: Docker Compose on a Linux VPS.
- Keep AI credentials and model calls on the backend.
- Enforce document authorization before any retrieved content reaches generation.
- Treat document content as untrusted data.
- Begin with exact vector search. Add indexes, hybrid retrieval, and reranking only when measurements justify them.

Still undecided: model provider, embedding dimensions, document storage, authentication provider, initial supported file format, and deployment host.

## Known issues and verification status

- The repository contains no application code yet.
- No dependencies, database, or AI services are configured.
- No behavior has been tested. Production readiness has not been established.

## Build order: one working piece at a time

This is a map of the project, not a list to implement all at once. Start with milestone 1. When you finish its first task, share your work for review; we will choose the next small task together.

### 1. Decide what a correct answer looks like

**What and why:** Use a short fictional policy to define what the chatbot should answer. This gives us something concrete to test before connecting an AI model.

**Example policy:** “HR Policy, section 4.2: Employees receive 20 working days of annual leave per year. Leave requests require manager approval.”

**Your first task:** Create `evaluation/first-cases.md`. Copy this fictional policy or write your own. Add these three kinds of questions with expected answers and supporting passages:

| Kind | Example question | Expected behavior |
| --- | --- | --- |
| Direct | How many annual leave days do employees receive? | Answer 20 working days and cite section 4.2. |
| Different wording | How much paid holiday can I take each year? | Find the annual-leave passage; do not invent details about pay, which this policy does not specify. |
| Missing information | Can unused leave carry over to next year? | Say the provided policy does not specify this. |

**Done when:** Every answer is supported by the sample text, and the missing-information case has no supporting passage. No application code is needed yet.

### 2. Understand the journey from a document to an answer

**What and why:** Separate preparing documents from answering questions. This explains why each component exists.

**Example:** Preparing: policy text → smaller passages called chunks → numerical representations called embeddings → database. Answering: question → question embedding → find relevant permitted chunks → ask the LLM to answer using those chunks → return answer and source.

**Your first task:** In `docs/architecture.md`, describe these two flows in your own words. Identify where the backend checks who the employee is and which documents they may access.

**Done when:** You can explain that retrieval finds evidence, while the LLM writes an answer from that evidence. An embedding represents text numerically so we can compare meaning; it is not the final answer.

### 3. Start a small backend

**Objective:** Create a backend that starts predictably, answers a simple health request, and refuses to start with invalid required settings. Complete the following steps separately; do not implement them all in one attempt.

#### 3A. Create and understand the starter application

**Your task:** Scaffold a NestJS application under `backend/` using the official NestJS setup instructions and your chosen package manager. Run its development script from `backend/package.json`.

Read these generated files:

- `backend/src/main.ts`: the entry point that creates the application and starts listening for requests.
- `backend/src/app.module.ts`: the root module that connects application components.
- `backend/src/app.controller.ts`: the initial HTTP route handler.
- `backend/package.json`: available scripts and installed dependencies.

**Check:** Open the starter route using the address printed at startup. Confirm a response arrives. If startup fails, inspect the first relevant error before changing configuration or reinstalling packages.

#### 3B. Add a basic health endpoint

**Why:** A health endpoint lets you or an operator check whether the application can respond to HTTP requests.

**Your task:** In the existing controller, define a handler for `GET /health` that returns a small JSON object such as `{"status":"ok"}` with HTTP status 200. Follow the route-handler pattern already present in the starter. A new module or health-check dependency is unnecessary for this simple endpoint.

**Check:** Start the backend and request `/health` with your browser or API client. Inspect both the status code and response body. An automated HTTP check should later assert those same values.

This is a basic liveness check: it shows that the application responds. It does not prove that the database or AI provider works. Dependency readiness checks belong to later steps when those dependencies exist.

#### 3C. Define which settings the backend needs

**Important distinction:** A `.env` file is one way to supply settings. Docker or the operating system can supply environment variables directly. A missing `.env` file should not cause failure if all needed values are already available.

Start with a small configuration contract:

| Setting | Needed now? | Rule | If absent |
| --- | --- | --- | --- |
| `PORT` | Yes, but has a default | Integer from 1 to 65535 | Use 3000. |
| `DATABASE_URL` | Only when database integration is added | Nonempty connection URL using an accepted PostgreSQL scheme | Fail startup once database integration requires it. |
| Provider API key | Only when the selected AI integration is added | Nonempty value under the provider-specific name we choose | Fail startup once that integration requires it. |

Do not require a database URL or an AI key in the starter backend merely to demonstrate validation. Invalid `PORT` already gives us a meaningful failure case.

**Files you will create or update:**

- `backend/.env.example`: safe sample names and placeholder values. Commit this so another developer knows what to configure.
- `backend/.env`: your local values if you choose to use a local environment file. Keep secrets out of Git.
- The applicable `.gitignore`: ensure local secret-bearing environment files are ignored while `.env.example` is allowed.
- `backend/src/config/validate-environment.ts`: a focused function that receives loaded settings, validates them, and returns normalized settings or throws an error.
- `backend/src/app.module.ts`: connect configuration loading and validation during application initialization.
- `backend/src/main.ts`: read the validated port rather than separately parsing raw environment values.

These are planned files and responsibilities; they have not been created by this guide.

#### 3D. Validate settings before accepting requests

**How to approach it:** Read the official NestJS Configuration guide, specifically environment loading, custom validation, and reading settings through the configuration service. The standard `@nestjs/config` integration solves configuration loading; a small custom validator is enough initially. Check its current setup instructions before installing it yourself.

The validator should follow this sequence:

```text
receive the loaded environment settings
if PORT is absent, select the default 3000
otherwise, convert PORT to a number
reject blank, non-integer, or out-of-range PORT values
when a feature introduces a required setting, validate it here too
if validation fails, throw an error naming the setting and rule
otherwise, return the settings with normalized values
```

Preserve other loaded settings in the returned configuration. Ensure the application uses the normalized port. Register validation during application initialization so a thrown validation error stops startup before the HTTP listener opens.

**Example error:** `Invalid configuration: PORT must be an integer between 1 and 65535.` Later: `Invalid configuration: DATABASE_URL is required.` Never include secret values in errors.

Use an error when the application cannot operate correctly without a setting. Use a default for genuinely optional settings. A warning is suitable only when continuing is intentional and the reduced behavior is clearly defined; warning and continuing with unusable required settings hides the problem.

Configuration validation checks the shape and presence of values. A correctly shaped database URL does not prove the database is reachable, and a nonempty API key does not prove it is valid. Those need separate integration checks later.

#### 3E. Verify success and failure deliberately

Run each scenario separately, restarting the backend after changing startup settings:

| Scenario | Expected result |
| --- | --- |
| No PORT supplied | Starts on the default port 3000. |
| PORT is 4000 | Starts on port 4000; `/health` returns 200 and the expected JSON. |
| PORT is `hello`, blank, 0, 65536, or 3000.5 | Startup fails with a useful message; this process does not begin serving HTTP. |
| No `.env`, valid settings supplied through the process environment | Starts normally. |
| Later, database integration exists but DATABASE_URL is absent from all sources | Startup fails before accepting requests. |

When checking a failure, stop any earlier backend process so its responses do not make a failed startup look successful. For missing-variable tests, account for both the local `.env` and inherited environment variables.

Add focused automated checks for the validator's default, valid conversion, invalid port, and future missing required setting. These checks protect real startup behavior. Do not call external services in validator unit tests.

**Done when:** You have observed both a successful health request and a deliberate configuration failure, and can explain why a missing file differs from a missing setting. Share the relevant files and results for review; do not add database or AI integration yet.

### 4. Give the backend a database

**What and why:** PostgreSQL stores documents and their smaller passages. The pgvector extension lets it store and compare embeddings too.

**Example:** One document named “HR Policy” has several chunks. Each chunk keeps its text and source location, such as section 4.2, so we can cite it later.

**Your first task:** Define a PostgreSQL service with pgvector in `compose.yaml`. Start it and confirm you can connect. Then we will design tables and write the first migration together.

**Done when:** You can save and read a sample document and chunk, preserve their relationship and source location, and store an embedding of the chosen model's required size.

### 5. Find a passage by meaning

**What and why:** Prove that a question can find a relevant passage even when the words differ. Start with a few manually prepared chunks, before building uploads.

**Example:** “How much time off do I get?” should rank the annual-leave passage above an unrelated parking policy.

**Your first task:** Prepare three short chunks on different topics. After choosing an embedding model, generate and store their embeddings. Next, personally write the SQL that compares a question embedding with them.

**Done when:** Direct and differently worded questions rank the expected passage highly. Record the model and vector size. For now, run this locally with synthetic data.

### 6. Control who can see each document

**What and why:** Authentication identifies the employee; authorization decides which documents that employee can use. Restrict evidence before sending it to an LLM.

**Example:** An HR employee may retrieve an HR-only policy. A general employee must never receive its text, answer details, or citations.

**Your first task:** Write a small access table in `docs/architecture.md` with two example users and two documents. Decide who may upload and read each document. Then choose and implement authentication and these access rules in small steps.

**Done when:** Tests show allowed access succeeds and forbidden access fails, including retrieval and source viewing. Complete this before exposing document or search endpoints to other users.

### 7. Turn an uploaded file into searchable passages

**What and why:** Ingestion means preparing a document for search: validate it, extract text, split it, generate embeddings, and save the results.

**Example:** Uploading a supported policy file creates a document record and searchable chunks. A failed embedding request leaves a visible failure state instead of pretending the document is ready.

**Your first task:** Choose one file format and define allowed file size and type. Implement upload validation first; add extraction, chunking, and embeddings one step at a time.

**Done when:** A valid file becomes searchable, an invalid upload is rejected, and retrying a failed upload does not create duplicate searchable chunks.

### 8. Expose search through the backend

**What and why:** Turn your local retrieval experiment into an authenticated API. Return evidence first, before asking an LLM to write answers.

**Example:** A proposed `POST /search` accepts a question and returns relevant permitted chunks with document names and section references. “Top-K” means returning at most K of the highest-ranked matches.

**Your first task:** Define the request and response fields. Then connect the endpoint to your existing embedding and SQL work, including permission filtering.

**Done when:** Your sample questions find the expected sources. Empty results, invalid input, and users without access have clear behavior. A highly ranked match is not automatically sufficient evidence to answer.

### 9. Write an answer using the retrieved evidence

**What and why:** Give the LLM the question and permitted passages, then return an answer with traceable citations.

**Example:** “Employees receive 20 working days of annual leave per year. [HR Policy, section 4.2]” For carryover, the response should explain that the provided policy does not specify it.

**Your first task:** Draft the instructions and context format for the model in `docs/answer-generation.md`. Describe how each source gets an identifier that can be checked against the retrieved passages.

**Done when:** Supported answers cite the right passage, missing evidence is acknowledged, and tests cover malicious document text such as “ignore previous instructions.” Treat document text as untrusted content; prompt wording alone is not a complete security defense.

### 10. Add the chat screen

**What and why:** Build the Next.js interface employees use to ask questions and inspect evidence.

**Example:** Send a question, see a loading state, receive an answer, and open an authorized source reference.

**Your first task:** Create the frontend under `frontend/` and build a minimal question form. Connect it to the backend after the form works.

**Done when:** An authenticated user can ask a question, read citations, and understand failures. Model credentials remain on the backend.

### 11. Handle document changes and failures

**What and why:** Policies change, uploads fail, and documents get deleted. Search results must reflect those changes consistently.

**Example:** Replacing a 20-day leave policy with a 25-day version should not leave the old version active accidentally. Deleting a document must remove its chunks from future search results.

**Your first task:** Define what should happen when the same file is uploaded twice. Implement and verify that behavior before moving on to versions, reprocessing, and deletion.

**Done when:** Duplicate uploads, replacements, failed processing, retries, and deletion each have predictable behavior and focused checks.

### 12. Check quality with a repeatable question set

**What and why:** Expand the first three cases to at least 20. Run the same cases after changes so improvements and regressions are visible.

**Example:** Check whether search found the expected policy, whether the answer agrees with it, whether the citation supports the claim, and whether a restricted user was denied access.

**Your first task:** Add cases for multiple policies, missing evidence, forbidden documents, and malicious document instructions. Record expected answers, sources, and permissions.

**Done when:** You can repeat the evaluation and report retrieval results, answer and citation quality, response time, token use, and cost. We will introduce metrics such as Recall@K when you have real results to interpret.

### 13. Improve search when a test shows a weakness

**What and why:** Choose improvements based on failures rather than adding complexity in advance.

**Example:** If meaning-based search misses an exact policy code such as HR-042, keyword search may help. Combining keyword and vector results is called hybrid search.

**Your first task:** Pick one failed evaluation case and investigate why it failed. Compare one focused change against the previous results.

**Done when:** The change measurably helps without breaking access rules or previously correct answers. Skip this milestone if basic retrieval already meets the requirements; advanced methods are optional.

### 14. Deploy and practise recovery

**What and why:** Running on a server also requires keeping data, protecting secrets, detecting failures, and recovering from mistakes.

**Example:** Restarting a container must preserve documents. Restoring a database backup should recover usable data. A failed release should have a known rollback path.

**Your first task:** Make the frontend, backend, and database run together locally with Docker Compose. Then handle server deployment, HTTPS, migrations, secrets, health checks, rate limits, safe logs, monitoring, backups, and rollback one task at a time.

**Done when:** The deployed system passes functional and access checks, persistent data survives restarts, a backup has actually been restored, and deployment and recovery instructions have been exercised.

## Next task — start here

Create `evaluation/first-cases.md` and complete the three cases from milestone 1. You can use the fictional policy above. Share the file for review; you do not need to install anything or scaffold the application for this task.

Use synthetic documents during development. Before using real internal material, establish the organization's data-handling requirements and review the chosen provider's terms.

## Updating this file

After a milestone is verified, record what passed, relevant decisions, unresolved issues, and the next concrete task. Do not mark planned work complete or claim checks that were not run.
