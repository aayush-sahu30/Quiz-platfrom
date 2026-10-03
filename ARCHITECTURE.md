Online Quiz Platform — System Architecture

Status: Active development
Architecture: Online-only, real-time quiz platform
Primary stack: Cloudflare Workers, Durable Objects, WebSockets, PostgreSQL, Hyperdrive, TypeScript, PWA

Related documents: README · Development Specification

Table of Contents

Project Overview

Project Goals

Development Principles

MVP Definition

Technology Stack

High-Level Architecture

Level 1 — Client Architecture

Level 2 — Network and Cloudflare Edge

Level 3 — Worker and Backend Architecture

Level 4 — Real-Time Quiz Architecture

Level 5 — Data Architecture

Level 6 — Complete Quiz Flow

Level 7 — Admin Architecture

Level 8 — Security Architecture

Level 9 — Deployment Architecture

Level 10 — Full System Architecture

Repository Structure

Quiz Lifecycle State Machine

Participant Admission and Queue

Real-Time Message Protocol

Timer and Scoring Model

Database Design

Persistence Strategy

API Design

Authentication and Authorization

Failure Handling and Recovery

Duplicate Submission and Idempotency

Capacity, Rate Limits, and Abuse Controls

Observability

Testing Strategy

CI/CD and Deployment

Environment Variables and Secrets

Local Development

Development Roadmap

Flaws Resolved by the Final Design

MVP vs Final Version

Minimum Demo

Engineering Decisions

Risks and Mitigations

Final Development Rule

1. Project Overview

The Online Quiz Platform is a real-time web application for conducting live online quizzes for campus events, workshops, competitions, and other timed assessments.

The platform separates the system into two major responsibilities:

Control/API layer: Cloudflare Worker handles routing, authentication, validation, and API operations.

Live session layer: a Durable Object instance coordinates each active quiz, including connected participants, admission, queueing, timer state, answer processing, scoring, and live updates.

Persistent data layer: PostgreSQL stores users, quiz definitions, questions, answers, scores, results, and audit data.

flowchart LR
    A[Participant / Admin Browser] --> B[Cloudflare Edge]
    B --> C[Cloudflare Worker]
    C --> D[QuizRoom Durable Object]
    C --> E[Hyperdrive]
    D --> E
    E --> F[(PostgreSQL)]
    D <-->|WebSocket| A

1.1 Core system idea

flowchart LR
    U[Online User] --> W[Cloudflare Worker]
    W --> Q[QuizRoom Durable Object]
    Q --> RT[Real-Time WebSocket Session]
    RT --> U
    Q --> P[(PostgreSQL via Hyperdrive)]

The platform is online-only. Questions, timers, answers, scoring, queueing, and results are coordinated through the online service.

2. Project Goals

2.1 Primary goals

The platform should allow a quiz organizer to:

Create and configure a quiz.

Add questions and answer options.

Set quiz timing and participant capacity.

Publish or open the quiz for participants.

Allow participants to authenticate and join online.

Admit participants until capacity is reached.

Place additional participants into a waiting queue.

Start and control a real-time quiz session.

Broadcast questions and timing to connected participants.

Receive and validate answers.

Calculate scores consistently on the server.

Broadcast score / leaderboard updates where enabled.

Persist answers and final results.

Handle reconnects and duplicate submissions safely.

Allow admins to monitor the live session.

End the quiz and publish final results.

2.2 Secondary goals

Later versions can support:

Advanced anti-cheating controls

Question randomization

Per-question or section scoring rules

Multiple quiz rooms at large scale

Analytics dashboards

Exportable results

Custom branding

Notifications

Advanced moderation

Audit/event replay

3. Development Principles

Build the online real-time core first.

Use Durable Objects for live coordination, not as the primary permanent database.

Use PostgreSQL for durable application records.

Never trust client-side timer, score, or capacity values.

Make answer submission idempotent.

Assume browsers disconnect and reconnect.

Persist all state that must survive Durable Object eviction or restart.

Use WebSocket Hibernation for the live WebSocket server path where appropriate.

Keep participant and admin permissions separate.

Every phase must produce a working system.

Do not add Redis unless load testing proves that the architecture needs it.

Do not add a separate socket server when Durable Objects already provide the required session coordination.

4. MVP Definition

The first usable version proves the fundamental concept:

Login → Join Quiz → Live Question → Answer → Score → Result

flowchart LR
    A[Participant Login] --> B[Join Quiz]
    B --> C{Capacity Available?}
    C -- Yes --> D[Admit]
    C -- No --> E[Queue]
    D --> F[Quiz Session]
    F --> G[Question + Timer]
    G --> H[Answer]
    H --> I[Score]
    I --> J[Next Question]
    J --> G
    J --> K[Final Result]
    E --> D

4.1 MVP acceptance criteria

Admin can create a quiz.

Admin can add questions and options.

Participant can authenticate.

Participant can join an active quiz.

Capacity is enforced server-side.

Overflow participants enter a queue.

Quiz questions are delivered in real time.

The server controls the timer.

Answers can be submitted once per question.

Scores are calculated server-side.

Final results are stored in PostgreSQL.

Reconnecting users can restore their active session.

Admin can observe participant count and quiz state.

5. Technology Stack

Layer

Technology

Purpose

Introduced in

Frontend

TypeScript, React or Vite, PWA APIs

Participant and admin interfaces

01

Edge

Cloudflare DNS, TLS, WAF / edge controls

Public traffic entry and security

01

Backend

Cloudflare Workers

Routing, APIs, auth, validation

01

Real-time

Durable Objects

Per-quiz coordination and state

02

Transport

WebSockets

Bidirectional live communication

02

DO storage

SQLite-backed Durable Object Storage

Durable per-session state

02

Database

PostgreSQL

Persistent application records

03

DB connection

Cloudflare Hyperdrive

Worker-to-PostgreSQL connectivity

03

Authentication

Session / token-based auth

User identity

01

Deployment

Wrangler + GitHub Actions

Build, test, deploy

04

Testing

Vitest / integration tests / load testing

Reliability and correctness

04

Observability

Workers logs + metrics + application audit events

Monitoring

05

Cloudflare currently documents Durable Objects as a stateful coordination mechanism and recommends SQLite-backed namespaces for new Durable Object classes. citeturn687645search6turn687645search9

For real-time WebSocket servers, Cloudflare documents WebSocket Hibernation as the preferred Durable Object approach when appropriate because the object can hibernate while connected clients remain connected. citeturn687645search0turn687645search1

Hyperdrive is the database connectivity layer for an existing PostgreSQL database and manages connection pooling between Workers and the database. citeturn687645search3turn687645search7

6. High-Level Architecture

flowchart TD
    P[Participant Browser] --> EDGE[Cloudflare Edge]
    A[Admin Browser] --> EDGE
    EDGE --> W[Cloudflare Worker]

    W --> AUTH[Authentication / Authorization]
    W --> API[API Router]
    W --> DO[QuizRoom Durable Object]
    W --> HD[Hyperdrive]

    DO <--> WS[WebSocket Sessions]
    WS <--> P
    DO --> HD
    HD --> DB[(PostgreSQL)]

    A --> W

Control plane

The Worker and APIs manage application-level requests:

authentication

quiz creation

quiz configuration

admin actions

participant admission requests

result retrieval

persistent reads / writes

Live session plane

The Durable Object manages one quiz session's coordination:

participant connections

admission and queue

current question

timing

answer processing

score updates

broadcasts

Data plane

PostgreSQL is the persistent source of truth for application records.

7. Level 1 — Client Architecture

flowchart TD
    APP[Quiz Platform]
    APP --> P[Participant]
    APP --> AD[Admin]

    P --> PL[Participant UI]
    PL --> LOGIN[Login]
    PL --> JOIN[Join Quiz]
    PL --> QUEUE[Queue Screen]
    PL --> QUIZ[Live Quiz]
    PL --> RESULT[Result]

    AD --> AL[Admin UI]
    AL --> CREATE[Create Quiz]
    AL --> QUESTION[Question Management]
    AL --> LIVE[Live Monitoring]
    AL --> RES[Results]

Participant responsibilities

Authenticate

Enter quiz code / link

Display admission state

Maintain WebSocket connection

Render current question

Show server-derived timer

Submit answer

Display acknowledgement

Recover session after reconnect

Display final result

Admin responsibilities

Create and edit quizzes

Configure capacity

Configure timing

Start / pause / finish quiz

Monitor participant count

Monitor queue

View results

8. Level 2 — Network and Cloudflare Edge

flowchart LR
    U[Browser] --> DNS[DNS]
    DNS --> EDGE[Cloudflare Edge]
    EDGE --> TLS[HTTPS / TLS]
    TLS --> SEC[Edge Security / Rate Controls]
    SEC --> W[Cloudflare Worker]

Request flow

Browser
  |
  | HTTPS
  v
DNS
  |
  v
Cloudflare Edge
  |
  +-- TLS termination
  +-- Edge security controls
  +-- Request routing
  |
  v
Cloudflare Worker

WebSocket flow

Browser
  |
  | WebSocket Upgrade
  v
Cloudflare Edge
  |
  v
Worker
  |
  v
QuizRoom Durable Object

The Durable Object acts as the real-time server for a quiz session and can coordinate multiple connected clients. Cloudflare's current documentation specifically describes Durable Objects as suitable for real-time WebSocket applications. citeturn687645search1

9. Level 3 — Worker and Backend Architecture

flowchart TD
    W[Cloudflare Worker]
    W --> R[Request Router]
    R --> AUTH[Authentication]
    R --> QUIZ[Quiz APIs]
    R --> ADMIN[Admin APIs]
    R --> WS[WebSocket Upgrade Route]

    AUTH --> VALID[Authorization / Validation]
    QUIZ --> VALID
    ADMIN --> VALID
    WS --> DO[QuizRoom Durable Object]

    VALID --> DB[Hyperdrive / PostgreSQL]
    VALID --> DO

Worker responsibilities

Receive Request
      |
      v
Parse Route
      |
      v
Authenticate
      |
      v
Authorize
      |
      v
Validate Input
      |
      +----> PostgreSQL for persistent data
      |
      +----> Durable Object for live session operations
      |
      v
Return Response

Example routes

POST /api/auth/login
POST /api/quizzes
GET  /api/quizzes/:quizId
POST /api/quizzes/:quizId/join
POST /api/quizzes/:quizId/start
POST /api/quizzes/:quizId/finish
GET  /api/quizzes/:quizId/results
GET  /api/quizzes/:quizId/participants
GET  /api/quizzes/:quizId/ws

10. Level 4 — Real-Time Quiz Architecture

Each active quiz is coordinated by a deterministic Durable Object identity such as quiz:{quizId}.

flowchart TD
    W[Worker] --> DO[QuizRoom Durable Object]

    DO --> STATE[Live Quiz State]
    DO --> ADMIT[Admission / Queue]
    DO --> TIMER[Timer State]
    DO --> ANSWER[Answer Processing]
    DO --> SCORE[Scoring]
    DO --> BOARD[Leaderboard]

    DO <-->|WebSocket| P1[Participant 1]
    DO <-->|WebSocket| P2[Participant 2]
    DO <-->|WebSocket| PN[Participant N]

QuizRoom state

QuizRoom
 |
 +-- quizId
 +-- status
 +-- currentQuestionId
 +-- currentQuestionIndex
 +-- questionStartAt
 +-- questionDeadlineAt
 +-- capacity
 +-- admittedCount
 +-- queue
 +-- participant sessions
 +-- per-question answer status
 +-- live score state

Durable state rule

In-memory state is an optimization, not the only copy of critical state. State that must survive object eviction, restart, or deployment is stored through the Durable Object Storage API or PostgreSQL, depending on its ownership. Cloudflare documents that in-memory Durable Object state can be lost on eviction/restart and recommends durable storage for state that must survive those events. citeturn751067search2

11. Level 5 — Data Architecture

flowchart TD
    W[Worker] --> DO[QuizRoom DO]
    W --> HD[Hyperdrive]
    DO --> HD
    HD --> PG[(PostgreSQL)]

    DO --> LIVE[Live Session State]
    PG --> PERM[Persistent Application Data]

Durable Object owns

Current live quiz state

Queue order

Connected session state

Current question state

Question timing metadata

Temporary live answer state

Live score state

PostgreSQL owns

Users

Roles

Quiz metadata

Questions

Options

Participant records

Submitted answers

Final scores

Final results

Admin actions

Audit events

Storage rule

LIVE / COORDINATION DATA
        |
        v
Durable Object Storage

PERMANENT APPLICATION DATA
        |
        v
PostgreSQL

Worker / DO
        |
        v
Hyperdrive
        |
        v
PostgreSQL

Hyperdrive can cache eligible read-only queries. For quiz operations that require read-after-write freshness, use a cache-disabled Hyperdrive configuration or another explicitly fresh read path. citeturn751067search4turn751067search10

12. Level 6 — Complete Quiz Flow

flowchart TD
    A[Participant Login] --> B[Join Quiz]
    B --> C[Worker Authentication]
    C --> D[QuizRoom]
    D --> E{Capacity Available?}

    E -- Yes --> F[Admit]
    E -- No --> G[Queue]
    G --> H[Slot Opens]
    H --> F

    F --> I[WebSocket Connected]
    I --> J[Quiz Starts]
    J --> K[Question Broadcast]
    K --> L[Server Timer]
    L --> M[Answer Submission]
    M --> N[Answer Validation]
    N --> O[Score Calculation]
    O --> P[Persist Required Data]
    P --> Q[Broadcast Update]
    Q --> R{More Questions?}
    R -- Yes --> K
    R -- No --> S[Final Result]
    S --> T[Persist Final Result]
    T --> U[Leaderboard / Result UI]

Join flow

Participant
   |
   v
POST /join
   |
   v
Worker
   |
   v
Authenticate + validate quiz
   |
   v
QuizRoom
   |
   +-- capacity available --> ADMITTED
   |
   +-- capacity full -------> QUEUED

Question flow

QuizRoom
   |
   v
Question becomes active
   |
   v
Store deadline
   |
   v
Broadcast question + deadline
   |
   v
Receive answers
   |
   v
Validate against deadline
   |
   v
Score
   |
   v
Persist answer / result data
   |
   v
Broadcast state update

13. Level 7 — Admin Architecture

flowchart TD
    A[Admin] --> UI[Admin Dashboard]
    UI --> W[Worker]

    W --> CQ[Create / Edit Quiz]
    W --> QQ[Question Management]
    W --> START[Start / Pause / Finish]
    W --> CAP[Capacity Control]
    W --> MON[Live Monitoring]
    W --> RESULT[Results]

    CQ --> PG[(PostgreSQL)]
    QQ --> PG
    START --> DO[QuizRoom]
    CAP --> DO
    MON --> DO
    RESULT --> PG

Admin operations

CREATE QUIZ
   -> PostgreSQL

ADD QUESTIONS
   -> PostgreSQL

PUBLISH QUIZ
   -> PostgreSQL

START QUIZ
   -> QuizRoom

PAUSE / RESUME
   -> QuizRoom

FINISH QUIZ
   -> QuizRoom + PostgreSQL

VIEW RESULTS
   -> PostgreSQL

14. Level 8 — Security Architecture

flowchart TD
    U[User] --> HTTPS[HTTPS / WSS]
    HTTPS --> EDGE[Cloudflare Edge]
    EDGE --> RL[Rate Controls]
    RL --> W[Worker]
    W --> AUTH[Authentication]
    AUTH --> RBAC[Role Authorization]
    RBAC --> VALID[Input Validation]
    VALID --> DO[Durable Object]
    VALID --> DB[PostgreSQL]

Security requirements

HTTPS for normal traffic.

Secure WebSocket connections.

Authentication before protected actions.

Role-based authorization for admin endpoints.

Server-side validation for every input.

Rate limiting for login, join, answer and admin actions.

Quiz identifiers must not be treated as authorization credentials.

Never trust client-provided score, timer, capacity, or role.

Do not log sensitive tokens or session secrets.

Use parameterized database queries.

Participant permission model

Participant
  |
  +-- Join assigned/public quiz
  +-- Read permitted quiz data
  +-- Submit own answers
  +-- Read own result

Participant CANNOT
  |
  +-- Create quiz
  +-- Modify questions
  +-- Change capacity
  +-- Change score
  +-- Read another participant's private data

Admin permission model

Admin
  |
  +-- Create / edit quiz
  +-- Manage questions
  +-- Start / pause / finish
  +-- Monitor participants
  +-- View results

15. Level 9 — Deployment Architecture

flowchart TD
    DEV[Developer] --> GIT[GitHub]
    GIT --> CI[GitHub Actions]
    CI --> TEST[Lint + Unit + Integration Tests]
    TEST --> BUILD[Build Frontend + Worker]
    BUILD --> DEPLOY[Wrangler Deploy]

    DEPLOY --> EDGE[Cloudflare Edge]
    EDGE --> WORKER[Worker]
    WORKER --> DO[Durable Objects]
    WORKER --> HD[Hyperdrive]
    DO --> HD
    HD --> PG[(PostgreSQL)]

Production deployment

GitHub
  |
  v
GitHub Actions
  |
  +-- install
  +-- lint
  +-- test
  +-- build
  +-- deploy
  |
  v
Cloudflare
  |
  +-- Static Assets / Frontend
  +-- Worker
  +-- Durable Object bindings
  +-- Hyperdrive binding
  |
  v
PostgreSQL

16. Level 10 — Full System Architecture

flowchart TD
    P[Participant Browser]
    A[Admin Browser]

    P --> EDGE[Cloudflare Edge]
    A --> EDGE

    EDGE --> W[Cloudflare Worker]

    W --> AUTH[Auth / Authorization]
    W --> API[API Router]
    W --> DO[QuizRoom Durable Object]
    W --> HD[Hyperdrive]

    DO <-->|WebSocket| P
    DO --> STATE[Live Quiz State]
    DO --> QUEUE[Queue / Capacity]
    DO --> TIMER[Authoritative Timer]
    DO --> SCORE[Server Scoring]

    DO --> HD
    HD --> PG[(PostgreSQL)]

    PG --> USERS[Users / Roles]
    PG --> QUIZZES[Quizzes / Questions]
    PG --> ANSWERS[Answers]
    PG --> RESULTS[Results]
    PG --> AUDIT[Audit Events]

Full request path

User
 |
 v
Cloudflare Edge
 |
 v
Worker
 |
 +--> Authentication
 |
 +--> Authorization
 |
 +--> API operation
 |
 +--> QuizRoom
       |
       +--> WebSocket
       +--> Admission
       +--> Timer
       +--> Answer validation
       +--> Scoring
       |
       v
   Hyperdrive
       |
       v
   PostgreSQL

17. Repository Structure

Recommended structure:

quiz-platform/
│
├── frontend/
│   ├── src/
│   │   ├── participant/
│   │   ├── admin/
│   │   ├── components/
│   │   ├── services/
│   │   └── main.tsx
│   ├── public/
│   ├── package.json
│   └── vite.config.ts
│
├── worker/
│   ├── src/
│   │   ├── index.ts
│   │   ├── routes/
│   │   │   ├── auth.ts
│   │   │   ├── quizzes.ts
│   │   │   ├── participants.ts
│   │   │   └── admin.ts
│   │   ├── durable-objects/
│   │   │   └── QuizRoom.ts
│   │   ├── services/
│   │   │   ├── auth.ts
│   │   │   ├── quiz.ts
│   │   │   ├── scoring.ts
│   │   │   └── persistence.ts
│   │   ├── lib/
│   │   │   ├── db.ts
│   │   │   ├── validation.ts
│   │   │   └── errors.ts
│   │   └── types/
│   ├── migrations/
│   └── package.json
│
├── db/
│   ├── schema.sql
│   └── migrations/
│
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── websocket/
│   └── load/
│
├── .github/
│   └── workflows/
│       ├── test.yml
│       └── deploy.yml
│
├── docs/
│   ├── ARCHITECTURE.md
│   ├── API.md
│   ├── DATABASE.md
│   └── SECURITY.md
│
├── wrangler.jsonc
├── package.json
├── README.md
└── .env.example

18. Quiz Lifecycle State Machine

A quiz has explicit lifecycle states.

stateDiagram-v2
    [*] --> DRAFT
    DRAFT --> PUBLISHED
    PUBLISHED --> OPEN
    OPEN --> LIVE
    OPEN --> CANCELLED
    LIVE --> PAUSED
    PAUSED --> LIVE
    LIVE --> FINISHING
    FINISHING --> COMPLETED
    FINISHING --> FAILED
    COMPLETED --> [*]
    CANCELLED --> [*]

State

Meaning

DRAFT

Quiz is being configured

PUBLISHED

Quiz is ready for participants

OPEN

Participants may join / queue

LIVE

Questions are active

PAUSED

Temporary admin-controlled pause

FINISHING

Final submissions and result persistence

COMPLETED

Final result available

CANCELLED

Quiz terminated before completion

FAILED

Unexpected terminal failure requiring admin attention

State transition rule

Only the authorized server path can transition a quiz between states.

19. Participant Admission and Queue

Capacity is controlled inside the QuizRoom Durable Object, so admission decisions for one quiz are serialized through the same coordination point.

flowchart TD
    J[Join Request] --> V[Validate User + Quiz]
    V --> C{Capacity Available?}
    C -- Yes --> A[Admit]
    C -- No --> Q[Queue]
    Q --> S[Slot Becomes Available]
    S --> A
    A --> W[WebSocket Session]

Queue requirements

Queue position is assigned by the QuizRoom.

A queued participant cannot receive live questions until admitted.

Disconnecting participants are marked inactive after a timeout policy.

A queue promotion is processed by the QuizRoom, not by the browser.

Admission cannot exceed configured capacity.

Capacity invariant

admitted participants <= configured capacity

20. Real-Time Message Protocol

Use explicit message types instead of sending unstructured strings.

Client → Server

{
  "type": "answer.submit",
  "quizId": "quiz_123",
  "questionId": "q_05",
  "submissionId": "uuid",
  "answer": "B"
}

Other message types:

session.resume
answer.submit
ping

Server → Client

session.accepted
session.queued
session.promoted
quiz.started
question.started
answer.accepted
answer.rejected
score.updated
leaderboard.updated
quiz.paused
quiz.resumed
quiz.finished
session.resync
error

Message rule

Every message must contain enough context to identify the operation safely:

message type
quiz/session identity
participant identity from authenticated session
request/submission identity where needed
payload

21. Timer and Scoring Model

21.1 Timer ownership

The server is authoritative.

flowchart LR
    DO[QuizRoom] --> ST[Question Start Time]
    DO --> DL[Question Deadline]
    DL --> B[Broadcast Deadline]
    B --> UI[Participant UI]

The browser displays the countdown, but the server validates whether an answer arrived before the authoritative deadline.

21.2 Why the client timer is not trusted

Client timer
   |
   +-- can be modified
   +-- can pause in background tabs
   +-- can drift
   +-- can be manipulated

Therefore:

server start time + configured duration = authoritative deadline

21.3 Scoring

The Worker / QuizRoom determines the final score.

Example:

answer correct + within deadline -> points awarded
answer incorrect                 -> zero / configured penalty
answer after deadline             -> rejected / zero
duplicate submission              -> ignored / deterministic response

The client never sends its own score.

22. Database Design

erDiagram
    USER ||--o{ QUIZ_PARTICIPANT : joins
    USER ||--o{ QUIZ : creates
    QUIZ ||--o{ QUESTION : contains
    QUESTION ||--o{ QUESTION_OPTION : has
    QUIZ ||--o{ QUIZ_PARTICIPANT : has
    QUIZ_PARTICIPANT ||--o{ ANSWER : submits
    QUESTION ||--o{ ANSWER : receives
    QUIZ_PARTICIPANT ||--|| RESULT : produces
    QUIZ ||--o{ AUDIT_EVENT : records

    USER {
        uuid id PK
        string name
        string email
        string role
        datetime created_at
    }

    QUIZ {
        uuid id PK
        uuid created_by FK
        string title
        string status
        int capacity
        int question_time_seconds
        datetime start_at
        datetime end_at
        datetime created_at
    }

    QUESTION {
        uuid id PK
        uuid quiz_id FK
        int position
        string question_text
        int points
        int time_limit_seconds
    }

    QUESTION_OPTION {
        uuid id PK
        uuid question_id FK
        string option_text
        boolean is_correct
    }

    QUIZ_PARTICIPANT {
        uuid id PK
        uuid quiz_id FK
        uuid user_id FK
        string state
        datetime joined_at
        datetime admitted_at
    }

    ANSWER {
        uuid id PK
        uuid participant_id FK
        uuid question_id FK
        string selected_option_id
        string submission_id
        boolean is_correct
        int points_awarded
        datetime submitted_at
    }

    RESULT {
        uuid id PK
        uuid participant_id FK
        int score
        int correct_count
        int answered_count
        datetime completed_at
    }

    AUDIT_EVENT {
        uuid id PK
        uuid quiz_id FK
        uuid actor_id FK
        string event_type
        json payload
        datetime created_at
    }

Database rules

submission_id must be unique per participant/question where appropriate.

User access is always filtered by authorization.

Final results are immutable after quiz completion unless an explicit admin correction workflow exists.

Question order is persisted and should not depend on client ordering.

23. Persistence Strategy

Live session state

Store in the Durable Object:

current question
question deadline
admitted participants
queue
live connection metadata
live score state
quiz status

Persistent state

Store in PostgreSQL:

quiz configuration
questions
participant records
submitted answers
final scores
results
admin actions

Persistence sequence

flowchart LR
    A[Answer Received] --> V[Validate]
    V --> S[Update Live State]
    S --> P[Persist Answer / Required Record]
    P --> B[Broadcast Result]

The exact ordering should be implemented so the system has a deterministic recovery strategy. A result that must never be lost should not exist only in volatile in-memory state.

24. API Design

Public / participant endpoints

Method

Path

Purpose

POST

/api/auth/login

Create / establish user session

GET

/api/auth/me

Current user

GET

/api/quizzes/{id}

Read public quiz metadata

POST

/api/quizzes/{id}/join

Request admission

GET

/api/quizzes/{id}/session

Get current session state

GET

/api/quizzes/{id}/ws

WebSocket upgrade

GET

/api/quizzes/{id}/result

Participant result

Admin endpoints

Method

Path

Purpose

POST

/api/admin/quizzes

Create quiz

PATCH

/api/admin/quizzes/{id}

Update quiz

POST

/api/admin/quizzes/{id}/publish

Publish quiz

POST

/api/admin/quizzes/{id}/start

Start quiz

POST

/api/admin/quizzes/{id}/pause

Pause quiz

POST

/api/admin/quizzes/{id}/resume

Resume quiz

POST

/api/admin/quizzes/{id}/finish

Finish quiz

GET

/api/admin/quizzes/{id}/participants

Participant monitoring

GET

/api/admin/quizzes/{id}/results

Results

Response format

{
  "data": {},
  "error": null,
  "requestId": "req_123"
}

Error format:

{
  "data": null,
  "error": {
    "code": "QUIZ_NOT_ACTIVE",
    "message": "The quiz is not currently accepting participants."
  },
  "requestId": "req_123"
}

25. Authentication and Authorization

Authentication flow

sequenceDiagram
    participant U as User
    participant W as Worker
    participant DB as PostgreSQL

    U->>W: Login request
    W->>DB: Validate identity / account
    DB-->>W: User record
    W-->>U: Secure session
    U->>W: Protected request
    W->>W: Validate session
    W->>W: Check role / ownership
    W-->>U: Authorized response

Authorization rules

request
  |
  v
authenticated?
  |
  +-- no --> 401
  |
  +-- yes
        |
        v
      role allowed?
        |
        +-- no --> 403
        |
        +-- yes
              |
              v
          resource ownership / permission

A quiz code or URL is not sufficient authorization for admin operations.

26. Failure Handling and Recovery

The system assumes failures are normal:

Browser disconnect

WebSocket disconnect

Durable Object eviction / restart

Network latency

Database error

Duplicate messages

Participant closing a tab

Admin refreshing the page

26.1 WebSocket reconnect

flowchart TD
    C[Connected] --> X[Connection Lost]
    X --> R[Reconnect with Backoff]
    R --> S[Authenticate Session]
    S --> RS[Request Session Resync]
    RS --> DO[QuizRoom]
    DO --> SNAP[Current Session Snapshot]
    SNAP --> C

26.2 Session resync

The server should be able to reconstruct:

quiz status
current question
question deadline
participant state
latest accepted answer state
score / result state permitted for the participant

26.3 Durable Object restart

Critical live state must be persisted so the object can restore its session after eviction or restart. Cloudflare explicitly notes that in-memory Durable Object state may be lost during eviction/restart and that durable storage should be used for state that must survive. citeturn751067search2

27. Duplicate Submission and Idempotency

A browser can retry an answer because of network uncertainty.

Problem

Client sends answer
       |
       v
Network delay
       |
Client retries
       |
Server receives twice

Solution

Every answer submission receives a unique submissionId.

{
  "type": "answer.submit",
  "submissionId": "0f6d...",
  "questionId": "q5",
  "answer": "B"
}

The server stores / checks the submission identity.

submissionId already processed?
        |
    +---+---+
    |       |
   yes      no
    |        |
return      process
existing      |
result        v
           persist

The same request must never award points twice.

28. Capacity, Rate Limits, and Abuse Controls

Admission invariant

admittedCount <= configuredCapacity

Suggested controls

Operation

Control

Login

Rate limit per identity / IP policy

Join

Rate limit + quiz validation

WebSocket upgrade

Auth + session validation

Answer

Question/session validation

Admin actions

Strict role check + rate limit

Result export

Admin-only + rate limit

Capacity behavior

flowchart LR
    J[Join] --> C{Capacity}
    C -- Available --> A[Admit]
    C -- Full --> Q[Queue]
    Q --> N[Next Available Slot]
    N --> A

Avoid introducing Redis solely to implement the queue. The first design keeps the queue inside the QuizRoom responsible for that quiz. A separate distributed queue can be evaluated later if load testing demonstrates a need.

29. Observability

29.1 Application events

Record structured events such as:

quiz.created
quiz.published
participant.joined
participant.queued
participant.admitted
quiz.started
question.started
answer.accepted
answer.rejected
participant.reconnected
quiz.paused
quiz.resumed
quiz.finished
result.persisted

29.2 Important metrics

active_quizzes
active_websocket_connections
queued_participants
admitted_participants
answers_per_second
answer_rejection_rate
reconnect_rate
quiz_completion_rate
result_persistence_failures
worker_errors

29.3 Structured log example

{
  "level": "INFO",
  "event": "answer.accepted",
  "quizId": "quiz_123",
  "participantId": "user_456",
  "questionId": "q_05",
  "requestId": "req_987"
}

Never log passwords, session secrets, or sensitive credentials.

30. Testing Strategy

Test type

What it validates

Tools / approach

Unit

Scoring, validation, state transitions

Vitest

API integration

Worker routes and persistence

Integration tests

WebSocket

Join, question, answer, reconnect

WebSocket test client

State machine

Valid / invalid transitions

Unit tests

Queue

Capacity invariant

Concurrent tests

Idempotency

Duplicate answer handling

Repeated request tests

Failure

Disconnect / restart recovery

Fault injection

Load

Many concurrent participants

k6 / equivalent

Security

Unauthorized operations

Security test suite

Required failure tests

Full capacity does not over-admit.

Two concurrent join requests cannot bypass capacity.

Duplicate answer does not double-score.

Late answer is rejected according to the server deadline.

Participant reconnect restores correct question state.

Admin-only endpoint rejects participants.

Participant cannot access another participant's result.

Durable Object restart does not erase required persistent data.

Database outage produces a controlled failure state.

31. CI/CD and Deployment

flowchart LR
    C[Commit / Pull Request] --> L[Lint]
    L --> T[Test]
    T --> B[Build]
    B --> I[Integration Tests]
    I --> D[Deploy Preview / Staging]
    D --> P[Production Deploy]

GitHub Actions

Recommended workflows:

.github/workflows/
├── test.yml
└── deploy.yml

test.yml

Install dependencies
       |
       v
Type check / lint
       |
       v
Unit tests
       |
       v
Integration tests

deploy.yml

Build
  |
  v
Test
  |
  v
Deploy Worker + Assets
  |
  v
Apply / verify Durable Object migrations
  |
  v
Smoke test

Keep deployment credentials in GitHub Actions secrets or the appropriate Cloudflare secret mechanism; never commit them.

32. Environment Variables and Secrets

Example configuration:

DATABASE_URL=<development-only or Hyperdrive-backed configuration>
SESSION_SECRET=<secret>
AUTH_SECRET=<secret>
QUIZ_PUBLIC_URL=<url>
HYPERDRIVE_ID=<configured in Wrangler>

Rules

Never commit .env.

Commit .env.example only.

Never send database credentials to the browser.

Never send signing secrets to the browser.

Store production secrets using the platform's secret mechanism.

Rotate secrets when required.

Wrangler bindings

Conceptually:

Worker
 |
 +-- DURABLE_OBJECTS
 |
 +-- HYPERDRIVE
 |
 +-- Secrets
 |
 +-- Static Assets

Cloudflare documents bindings as capabilities exposed to the Worker runtime, including Durable Objects and Hyperdrive. citeturn751067search1

33. Local Development

Suggested local architecture

flowchart TD
    F[Frontend] --> W[Wrangler Dev]
    W --> DO[Local Durable Object]
    W --> DB[(Local PostgreSQL)]

Commands

npm install
npm run dev

For Wrangler-based development, use local bindings for Durable Objects and Hyperdrive-compatible local database configuration as required by the environment.

Example local environment

Frontend        -> local browser
Worker          -> wrangler dev
Durable Object  -> local simulator/runtime
PostgreSQL      -> local Docker container

34. Development Roadmap

Phase 01 — Project Foundation

Repository structure

TypeScript / Worker setup

Frontend shell

Wrangler configuration

Local development

Phase 02 — Authentication

Login

Session handling

Participant / admin roles

Protected routes

Phase 03 — Quiz Management

Create quiz

Edit quiz

Add questions

Configure capacity and timing

PostgreSQL schema

Phase 04 — Durable Object QuizRoom

QuizRoom binding

Per-quiz Durable Object identity

Admission control

Queue

Live quiz state

Phase 05 — WebSocket Real-Time Layer

WebSocket upgrade

Connection management

Question broadcasting

Answer messages

Reconnect / resync

Phase 06 — Scoring and Results

Server-side scoring

Idempotent answer processing

Result persistence

Leaderboard

Phase 07 — Admin Dashboard

Live monitoring

Participant count

Queue state

Start / pause / finish controls

Results

Phase 08 — Security and Reliability

Rate limiting

Authorization hardening

Failure recovery

Audit events

Load tests

Phase 09 — Production Deployment

CI/CD

Production database

Hyperdrive

Monitoring

Domain / HTTPS

Production smoke tests

35. Flaws Resolved by the Final Design

35.1 Old problem: using a separate socket server

Problem:

Worker -> Node.js / Socket.IO -> Redis

adds an additional always-running real-time service.

Final design:

Worker -> QuizRoom Durable Object -> WebSocket

The Durable Object is responsible for the per-quiz real-time session.

35.2 Old problem: relying on only in-memory state

Problem:

A restart / eviction could remove live state.

Final design:

Live coordination -> Durable Object
Critical durable state -> DO Storage / PostgreSQL

Cloudflare documents that Durable Object in-memory state does not survive eviction/restart and durable storage should be used for state that must survive. citeturn751067search2

35.3 Old problem: client-controlled timer

Problem:

A browser could alter or freeze its timer.

Final design:

QuizRoom
  |
  +-- questionStartAt
  +-- questionDeadlineAt

The client only displays the timer.

35.4 Old problem: client-controlled score

Problem:

The browser cannot be trusted to calculate the final score.

Final design:

Answer -> QuizRoom -> validate -> score -> persist -> broadcast

35.5 Old problem: duplicate answers

Problem:

Retries could score the same answer twice.

Final design:

submissionId
      |
      v
Idempotency check
      |
      v
Process once

35.6 Old problem: capacity checked by multiple servers

Problem:

Concurrent requests could exceed capacity.

Final design:

All admission decisions for one quiz go through the same QuizRoom coordination point.

Join A --+
Join B --+--> QuizRoom --> serialized capacity decisions
Join C --+

35.7 Old problem: treating PostgreSQL as the real-time state machine

Problem:

Using the database as the primary coordination loop for every WebSocket event creates unnecessary load and complexity.

Final design:

Live coordination -> Durable Object
Persistent records -> PostgreSQL

35.8 Old problem: stale database reads after writes

Problem:

Hyperdrive can cache eligible read queries.

Final design:

Use a fresh / cache-disabled read path for operations that require read-after-write consistency, such as authoritative result reads immediately after a write. Cloudflare documents this Hyperdrive behavior explicitly. citeturn751067search4turn751067search10

35.9 Old problem: assuming WebSocket memory survives forever

Problem:

Durable Objects can hibernate or restart.

Final design:

Use Durable Object storage for durable state and WebSocket connection attachments / resync mechanisms for connection-associated information. Cloudflare documents the hibernation model and serializeAttachment / deserializeAttachment for connection state. citeturn687645search0turn687645search1

36. MVP vs Final Version

flowchart LR
    MVP[Login + Join + Quiz + Answer + Result]
      --> V2[Durable Object + WebSocket + Queue]
      --> V3[Admin + Reconnect + Idempotency]
      --> V4[Security + Load Testing + Observability]
      --> V5[Production Deployment]

Version

Features

MVP

Auth, quiz creation, join, questions, answers, score, result

Version 2

Durable Objects, WebSockets, queue, capacity

Version 3

Admin dashboard, reconnect, idempotency, audit events

Version 4

Security hardening, rate limits, load tests, monitoring

Final

CI/CD, production PostgreSQL + Hyperdrive, production domain, complete reliability controls

37. Minimum Demo

A successful first demonstration should show:

Admin creates a quiz.

Admin sets capacity.

Participants open the online quiz page.

Participants authenticate.

First users are admitted.

Additional users enter the queue.

Admin starts the quiz.

All admitted participants receive the same question.

Server-authoritative timer is visible.

Participants submit answers.

Scores are calculated server-side.

Leaderboard / progress updates are broadcast.

A participant disconnects and reconnects.

The participant receives the current session state.

Quiz finishes.

Results are stored in PostgreSQL.

Admin views the final results.

Demo architecture

flowchart LR
    A[Admin] --> W[Worker]
    P1[Participant 1] --> W
    P2[Participant 2] --> W
    P3[Participant 3] --> W
    W --> DO[QuizRoom]
    DO <--> WS[WebSocket]
    WS --> P1
    WS --> P2
    WS --> P3
    DO --> DB[(PostgreSQL)]

38. Engineering Decisions

Decision

Reasoning

Online-only

The system is designed around live online participation

Worker as backend entry

Simple Cloudflare-native request routing

One QuizRoom per quiz

Keeps state and coordination scoped to a quiz

Durable Objects for live state

Stateful coordination with strong consistency and WebSocket support

WebSockets

Real-time bidirectional quiz updates

PostgreSQL for permanent data

Structured persistent records and result history

Hyperdrive for DB access

Cloudflare-managed connectivity / pooling to PostgreSQL

Client timer only for display

Prevents client clock from becoming authoritative

Server scoring

Prevents client score manipulation

Queue inside QuizRoom

Keeps capacity decision centralized for each quiz

Idempotent submissions

Protects against retries / duplicate network delivery

Reconnect and resync

Browsers and networks are unreliable

No Redis initially

Avoid extra infrastructure until load testing demonstrates a need

39. Risks and Mitigations

Risk

Impact

Mitigation

Too many concurrent participants in one quiz

High live-session load

Load test; tune message frequency; consider quiz sharding strategy if needed

WebSocket disconnects

Participant loses live view

Reconnect + session resync

Duplicate answer requests

Incorrect score

Idempotency keys

Client clock manipulation

Timing abuse

Server-authoritative deadline

Database outage

Persistence failure

Controlled error state + retry / recovery strategy

Durable Object restart

Live memory lost

Persist recoverable state

Hyperdrive stale read

Incorrect fresh state

Fresh read / cache-disabled path where required

Admin misuse

Quiz corruption

Role checks + state transition validation

Abuse / flooding

Resource exhaustion

Rate limits + admission controls

Scope expansion

Project delay

Follow phased roadmap

40. Final Development Rule

Build the smallest working real-time system first.

Start with:

flowchart LR
    A[Login] --> B[Join Quiz] --> C[QuizRoom] --> D[Question] --> E[Answer] --> F[Score] --> G[Result]

Then add, in order:

Queue
  -> WebSocket reconnect
  -> Idempotency
  -> Admin monitoring
  -> Security hardening
  -> Load testing
  -> Observability
  -> Production deployment

Do not add Redis, a separate Socket.IO server, or unnecessary infrastructure before the core Durable Object architecture has been implemented and load-tested.

The final target architecture remains:

Participants / Admin
        |
        v
Cloudflare Edge
        |
        v
Cloudflare Worker
        |
        v
QuizRoom Durable Object
        |
        +------ WebSocket ------> Participants
        |
        v
Hyperdrive
        |
        v
PostgreSQL
