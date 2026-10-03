# Quiz-platfrom
Status: Active development
Mode: Online-only
Architecture: Cloudflare Workers + Durable Objects + WebSockets + PostgreSQL + Hyperdrive
Primary use case: Campus events, society competitions, workshops, live assessments, and quiz contests

EventFlow is an online real-time quiz platform designed for live events where many participants join the same quiz session simultaneously. The platform focuses on reliable real-time coordination, controlled admission, server-authoritative scoring, persistent results, and a deployment model that can stay close to a near-zero-cost development target.

The system is intentionally built around Cloudflare's edge runtime instead of a traditional always-running backend server. A Cloudflare Worker handles HTTP/API traffic, while a Durable Object represents the live coordination boundary for an active quiz room. PostgreSQL stores persistent application data, with Hyperdrive providing the Worker-to-PostgreSQL connection layer.

Important: Participant capacity per quiz room is an engineering result, not a number assumed from a provider quota. The project must determine its safe room size through load testing.

Table of Contents

Project Overview

Problem Statement

Project Goals

Core Features

Architecture Principles

Technology Stack

System Architecture

Client Architecture

Cloudflare Edge and Worker

Durable Object and Real-Time System

Quiz Room Lifecycle

Admission and Queue System

Timer and Scoring Model

Data Architecture

Database Design

Authentication and Authorization

Admin System

API Design

WebSocket Protocol

Failure Handling and Recovery

Security

Deployment

CI/CD

Repository Structure

Environment Configuration

Local Development

Testing Strategy

Load Testing and Capacity

Monitoring and Logging

Cost Strategy

Resolved Architecture Flaws

Event-Day Checklist

Development Roadmap

MVP vs Final Version

Engineering Rules

Final Architecture

1. Project Overview

EventFlow is a browser-based online quiz platform for live events.

The platform separates responsibilities into three main runtime layers:

Cloudflare Worker: routing, APIs, authentication validation, authorization, validation, and orchestration.

Durable Object (QuizRoom): authoritative live state for one active quiz session, including admission, queue state, question state, deadlines, live answers, scoring, and WebSocket coordination.

PostgreSQL: persistent users, quizzes, questions, answers, participation records, final scores, results, and audit information.

flowchart LR
    USERS[Participants and Admins] --> EDGE[Cloudflare Edge]
    EDGE --> WORKER[Cloudflare Worker]
    WORKER --> DO[QuizRoom Durable Object]
    WORKER --> HD[Hyperdrive]
    DO --> WS[WebSockets]
    DO --> HD
    HD --> DB[(PostgreSQL)]
    WS --> USERS

Core runtime rule

HTTP/API traffic       -> Worker
Live quiz coordination -> Durable Object
Real-time transport   -> WebSocket
Persistent data       -> PostgreSQL
DB connectivity       -> Hyperdrive
Authentication        -> Supabase Auth
Object/media storage  -> R2 only when required

2. Problem Statement

Typical event quiz tools can create problems when an event requires:

many users joining close to the same time,

reliable live timers,

real-time question delivery,

controlled participant capacity,

waiting queues,

immediate score updates,

predictable behavior during reconnects,

persistent results,

low infrastructure cost,

and control over the platform architecture.

EventFlow addresses these requirements with a per-quiz coordination model rather than sending every live operation through a traditional central application server.

3. Project Goals

Primary goals

Provide online participant authentication.

Allow admins to create and manage quizzes.

Allow participants to join a specific quiz room.

Enforce a configurable room capacity.

Place excess participants into a waiting queue.

Deliver questions in real time.

Maintain an authoritative server-side deadline.

Validate answers server-side.

Calculate scores server-side.

Prevent duplicate answer submissions.

Support reconnects without losing the participant's session state.

Persist important records and final results.

Provide an admin dashboard for event control.

Run the application primarily on Cloudflare's edge infrastructure.

Keep development and small-event costs close to zero where free tiers are sufficient.

Secondary goals

Later versions can add:

question images and media through R2,

richer analytics,

exportable results,

certificates,

anti-abuse controls,

multiple simultaneous quizzes,

advanced event reports,

stronger administrative controls,

and a dedicated production observability stack.

4. Core Features

Feature

Description

Participant login

Users authenticate before joining the quiz

Admin login

Administrators access management functions

Quiz creation

Admin creates title, questions, timing, capacity, and rules

Quiz joining

Participant joins using the event/quiz identifier

Capacity control

Room has a configurable maximum active participant count

Waiting queue

Additional participants wait until capacity becomes available

Real-time questions

Questions are delivered through WebSockets

Server-authoritative timer

Browser renders countdown, but server owns the deadline

Answer validation

Worker / Durable Object validates the answer against quiz state

Server-side scoring

Client cannot directly modify score

Leaderboard

Live or final ranking can be published by admin configuration

Reconnect

Disconnected users can reconnect to the active room

Persistent results

Final result is stored in PostgreSQL

Admin monitoring

Admin can see room state, participants, and quiz status

Rate limiting

Sensitive operations are protected against abuse

5. Architecture Principles

5.1 Online-only

EventFlow is designed as an online real-time system. The participant client does not pre-download an offline quiz package.

5.2 One live coordinator per quiz room

Each active quiz session is mapped to one QuizRoom Durable Object instance.

5.3 Client is not authoritative

The browser may display state, but it does not control:

score,

answer correctness,

answer deadline,

participant admission,

queue position,

or quiz state transitions.

5.4 Separate live state from permanent data

Live session state       -> Durable Object
Permanent application data -> PostgreSQL

5.5 Minimize database writes

Do not send timer ticks, heartbeat updates, or every cosmetic UI update to PostgreSQL.

5.6 Recover from restarts

Important live-session state must be recoverable from Durable Object storage rather than depending only on process memory.

5.7 Test capacity instead of guessing it

Provider documentation gives platform capabilities and quotas, but the safe EventFlow participant count is determined by realistic load testing.

6. Technology Stack

Layer

Technology

Purpose

Frontend

React + TypeScript

Participant and admin UI

Edge

Cloudflare DNS / TLS

Domain routing and secure entry point

Backend

Cloudflare Workers

API and application logic

Real-time

Durable Objects

Quiz-room coordination

Transport

WebSockets

Real-time room communication

Durable live storage

SQLite-backed Durable Object storage

Recoverable room state

Authentication

Supabase Auth

Google OAuth and user sessions

Database

PostgreSQL

Persistent data

DB connectivity

Cloudflare Hyperdrive

Worker-to-PostgreSQL connection layer

Object storage

Cloudflare R2

Optional media/files; not required for MVP

Source control

GitHub

Repository and collaboration

CI/CD

GitHub Actions

Automated validation and deployment

Deployment tooling

Wrangler

Worker/DO deployment

7. System Architecture

7.1 High-level architecture

flowchart TD
    PARTICIPANT[Participant Browser] --> EDGE[Cloudflare Edge]
    ADMIN[Admin Browser] --> EDGE
    EDGE --> WORKER[Cloudflare Worker]

    WORKER --> AUTH[Supabase Auth Validation]
    WORKER --> ROUTER[API Router]
    ROUTER --> DO[QuizRoom Durable Object]
    ROUTER --> HD[Hyperdrive]

    DO --> WS[WebSocket Layer]
    WS --> PARTICIPANT
    DO --> LIVE[Durable Live State]
    DO --> HD

    HD --> DB[(Supabase PostgreSQL)]

    WORKER --> LOGS[Cloudflare Logs]

7.2 Control plane vs live session

flowchart LR
    CONTROL[Worker Control Layer] --> QUIZROOM[QuizRoom]
    CONTROL --> DATABASE[(PostgreSQL)]
    QUIZROOM --> USERS[Connected Participants]

The Worker acts as the control/API layer. The Durable Object is the live session coordinator.

8. Client Architecture

8.1 Participant client

flowchart TD
    USER[Participant] --> LOGIN[Google Login]
    LOGIN --> HOME[Participant Home]
    HOME --> JOIN[Join Quiz]
    JOIN --> WAIT[Waiting / Queue Screen]
    WAIT --> LIVE[Live Quiz Screen]
    LIVE --> QUESTION[Question]
    QUESTION --> TIMER[Local Countdown]
    QUESTION --> ANSWER[Submit Answer]
    ANSWER --> FEEDBACK[State / Score Update]
    FEEDBACK --> NEXT[Next Question]
    NEXT --> QUESTION
    FEEDBACK --> FINAL[Final Result]

8.2 Admin client

flowchart TD
    ADMIN[Admin] --> LOGIN2[Admin Login]
    LOGIN2 --> DASH[Dashboard]
    DASH --> CREATE[Create Quiz]
    DASH --> EDIT[Edit Questions]
    DASH --> START[Start / Resume]
    DASH --> MONITOR[Live Monitor]
    DASH --> STOP[Stop / Close]
    DASH --> RESULTS[Results]

9. Cloudflare Edge and Worker

9.1 Request path

flowchart LR
    BROWSER[Browser] --> DNS[Cloudflare DNS]
    DNS --> EDGE2[Cloudflare Edge]
    EDGE2 --> TLS[TLS / HTTPS]
    TLS --> SECURITY[Security / Rate Limits]
    SECURITY --> WORKER2[Cloudflare Worker]

9.2 Worker responsibilities

The Worker is responsible for:

route selection,

authentication-token validation,

authorization checks,

request validation,

quiz CRUD operations,

participant API operations,

admin API operations,

creating/resolving Durable Object IDs,

database access through Hyperdrive,

structured error responses.

9.3 Worker should not own the full live room

The Worker should not maintain the entire active room in normal Worker memory.

Instead:

flowchart LR
    REQUEST[Request] --> WORKER3[Worker]
    WORKER3 --> ROOM[QuizRoom]
    ROOM --> STATE[Live State]
    ROOM --> SOCKETS[WebSockets]

10. Durable Object and Real-Time System

10.1 QuizRoom model

A QuizRoom represents one active quiz session. The object is responsible for the authoritative coordination of that session.

flowchart TD
    WORKER4[Worker] --> ROOM2[QuizRoom Durable Object]
    ROOM2 --> PARTICIPANTS[Participant Registry]
    ROOM2 --> QUEUE[Waiting Queue]
    ROOM2 --> CURRENT[Current Question]
    ROOM2 --> DEADLINE[Question Deadline]
    ROOM2 --> SCORES[Live Scores]
    ROOM2 --> WEBSOCKETS[WebSocket Connections]
    ROOM2 --> STORAGE[DO Durable Storage]

10.2 WebSocket flow

sequenceDiagram
    participant P as Participant
    participant W as Worker
    participant D as QuizRoom

    P->>W: Request quiz session
    W->>D: Resolve QuizRoom
    D-->>W: Session information
    W-->>P: WebSocket endpoint/session token
    P->>D: WebSocket connect
    D-->>P: Room state
    D-->>P: Question + deadline
    P->>D: Answer submission
    D->>D: Validate and score
    D-->>P: Result / next state

10.3 WebSocket hibernation

WebSocket Hibernation should be preferred when the implementation does not need an always-running handler between messages. The room should avoid needless compute during idle connection time.

This is especially useful for waiting rooms where participants may remain connected without generating high message traffic.

10.4 Capacity

Cloudflare documents Durable Objects as being capable of coordinating large numbers of WebSocket clients, including thousands in documented examples, but EventFlow must not treat that as a guaranteed room capacity.

The project must load-test the actual quiz logic before publishing an event capacity.

Recommended engineering test stages:

100 participants
      ↓
250 participants
      ↓
500 participants
      ↓
750 participants
      ↓
1000 participants

These are test levels, not provider guarantees.

11. Quiz Room Lifecycle

11.1 State machine

stateDiagram-v2
    [*] --> DRAFT
    DRAFT --> OPEN
    OPEN --> RUNNING
    RUNNING --> PAUSED
    PAUSED --> RUNNING
    RUNNING --> COMPLETED
    OPEN --> CLOSED
    RUNNING --> CLOSED
    COMPLETED --> [*]
    CLOSED --> [*]

11.2 State meaning

State

Meaning

DRAFT

Quiz is being created/configured

OPEN

Participants may attempt to join

RUNNING

Quiz questions are being served

PAUSED

Admin temporarily pauses the room

COMPLETED

Quiz ends normally

CLOSED

Quiz cancelled or force-closed

12. Admission and Queue System

12.1 Capacity flow

flowchart TD
    JOINREQUEST[Participant Join Request] --> CHECKCAP{Capacity Available?}
    CHECKCAP -->|Yes| ADMIT[Admit to QuizRoom]
    CHECKCAP -->|No| QUEUE2[Add to Waiting Queue]
    QUEUE2 --> SLOT[Participant Slot Opens]
    SLOT --> ADMIT
    ADMIT --> ACTIVE[Active Participant]

12.2 Queue ownership

Queue state belongs to the QuizRoom, not to the browser.

Participant browser
      ↓
Join request
      ↓
QuizRoom
      ↓
Capacity decision
      ↓
ADMITTED / QUEUED

This prevents a participant from manipulating their queue position locally.

13. Timer and Scoring Model

13.1 Timer model

Do not broadcast a server message every second just to display a countdown.

Instead:

flowchart LR
    ROOM3[QuizRoom] --> STARTTIME[Question Start Time]
    ROOM3 --> DURATION[Question Duration]
    STARTTIME --> CLIENTTIMER[Browser Countdown]
    DURATION --> CLIENTTIMER
    CLIENTTIMER --> ANSWER2[Answer Submission]
    ANSWER2 --> ROOM3

The server calculates whether an answer arrived before the authoritative deadline.

13.2 Why this design

It reduces unnecessary WebSocket traffic while preventing client-side timer manipulation.

13.3 Scoring

flowchart TD
    SUBMIT[Answer Submission] --> ROOM4[QuizRoom]
    ROOM4 --> QUESTIONSTATE[Current Question State]
    QUESTIONSTATE --> DEADLINE2{Before Deadline?}
    DEADLINE2 -->|No| REJECT[Reject / Timeout]
    DEADLINE2 -->|Yes| CORRECT{Correct Answer?}
    CORRECT -->|Yes| ADD[Add Points]
    CORRECT -->|No| ZERO[Zero / Rule-Based Points]
    ADD --> UPDATE[Update Live Score]
    ZERO --> UPDATE
    UPDATE --> PERSIST[Persist Required Record]
    PERSIST --> BROADCAST[Broadcast State]

13.4 Duplicate answer protection

Every answer submission should include a logical idempotency key such as:

quiz_id + participant_id + question_id

Repeated submissions for the same question should not create multiple logical answers.

14. Data Architecture

14.1 Live vs persistent data

flowchart LR
    SESSION[Active Quiz Session] --> LIVE[Durable Object Storage]
    SESSION --> PERMANENT[Persistent Records]
    LIVE --> LIVEITEMS[Queue / Current Question / Deadline / Live Score]
    PERMANENT --> DB[(PostgreSQL)]

14.2 Durable Object live state

Keep in or recover through Durable Object storage:

participant membership,

queue information,

current quiz phase,

current question,

question deadline,

accepted answer state,

live scores,

room configuration required for recovery.

14.3 PostgreSQL persistent state

Store:

users,

roles,

quizzes,

questions,

options,

participation records,

submitted answers,

final scores,

results,

administrative events.

14.4 Write strategy

Do not write every:

timer tick,

WebSocket heartbeat,

UI repaint,

temporary cursor movement,

to PostgreSQL.

Only write data that must become durable application history or must be recovered outside the room.

15. Database Design

15.1 ER diagram

erDiagram
    USER ||--o{ QUIZ : creates
    QUIZ ||--o{ QUESTION : contains
    QUESTION ||--o{ OPTION : has
    USER ||--o{ PARTICIPATION : joins
    QUIZ ||--o{ PARTICIPATION : has
    USER ||--o{ ANSWER : submits
    QUESTION ||--o{ ANSWER : receives
    QUIZ ||--o{ RESULT : produces
    USER ||--o{ RESULT : receives

    USER {
        uuid id PK
        string name
        string email
        string role
        datetime created_at
    }

    QUIZ {
        uuid id PK
        uuid creator_id FK
        string title
        int capacity
        int duration_seconds
        string status
        datetime created_at
    }

    QUESTION {
        uuid id PK
        uuid quiz_id FK
        string text
        int position
        int points
    }

    OPTION {
        uuid id PK
        uuid question_id FK
        string text
        boolean is_correct
    }

    PARTICIPATION {
        uuid id PK
        uuid user_id FK
        uuid quiz_id FK
        string status
        int final_score
        datetime joined_at
    }

    ANSWER {
        uuid id PK
        uuid user_id FK
        uuid question_id FK
        uuid option_id FK
        boolean is_correct
        int points_awarded
        datetime submitted_at
    }

    RESULT {
        uuid id PK
        uuid user_id FK
        uuid quiz_id FK
        int score
        int rank
        datetime created_at
    }

15.2 Data rules

Use server-generated IDs.

Add uniqueness constraints for logical duplicate operations.

Do not expose internal admin-only fields to participants.

Keep timestamps server-generated.

Keep final result history immutable after publication unless an explicit admin correction workflow exists.

16. Authentication and Authorization

EventFlow uses Supabase Auth for identity and Google OAuth.

16.1 Login flow

sequenceDiagram
    participant P as Participant
    participant UI as Frontend
    participant SA as Supabase Auth
    participant W as Worker

    P->>UI: Click Google Login
    UI->>SA: Start OAuth
    SA-->>UI: Auth session
    UI->>W: API request + JWT
    W->>W: Validate JWT claims
    W-->>UI: Authorized response

16.2 Authorization

Roles should include at least:

participant

admin

An administrator can create/start/manage quizzes. A participant cannot call administrative operations merely by changing a client-side field.

16.3 Token behavior

The application should use Supabase's supported session-refresh behavior. The Worker should validate the access token/JWT claims for protected operations.

16.4 Event-day login strategy

Encourage participants to authenticate before the quiz starts instead of making hundreds of participants perform OAuth at the exact start second.

17. Admin System

17.1 Admin operations

flowchart TD
    ADMIN2[Admin Dashboard] --> CREATE2[Create Quiz]
    ADMIN2 --> QUESTIONS2[Manage Questions]
    ADMIN2 --> CONFIG[Configure Capacity / Timing]
    ADMIN2 --> START2[Start Quiz]
    ADMIN2 --> MONITOR2[Monitor Live Room]
    ADMIN2 --> STOP2[Stop Quiz]
    ADMIN2 --> RESULTS2[View Results]

    CREATE2 --> DB8[(PostgreSQL)]
    QUESTIONS2 --> DB8
    CONFIG --> DB8
    START2 --> ROOM5[QuizRoom]
    MONITOR2 --> ROOM5
    STOP2 --> ROOM5
    RESULTS2 --> DB8

17.2 Admin dashboard should display

quiz status,

active participants,

queue length,

current question,

elapsed time,

connection/reconnect information,

answer activity,

final results.

18. API Design

The API should stay small and purpose-specific.

Authentication

Method

Route

Purpose

GET

/api/auth/me

Return current authenticated user

POST

/api/auth/session

Optional session/bootstrap endpoint

Quiz management

Method

Route

Purpose

POST

/api/quizzes

Create quiz

GET

/api/quizzes/:id

Get quiz metadata

PATCH

/api/quizzes/:id

Update quiz

POST

/api/quizzes/:id/start

Start quiz

POST

/api/quizzes/:id/pause

Pause quiz

POST

/api/quizzes/:id/stop

Stop/close quiz

GET

/api/quizzes/:id/results

Results

Participant operations

Method

Route

Purpose

POST

/api/quizzes/:id/join

Join or queue participant

GET

/api/quizzes/:id/session

Get session state

POST

/api/quizzes/:id/reconnect

Recover session after disconnect

WebSocket

GET /api/quizzes/:quizId/ws

The exact routing can change with implementation, but the authoritative live room remains the Durable Object.

Example join response

{
  "quizId": "quiz_123",
  "status": "QUEUED",
  "position": 12
}

Or:

{
  "quizId": "quiz_123",
  "status": "ADMITTED",
  "roomId": "room_123"
}

19. WebSocket Protocol

Use explicit event names rather than relying on loosely structured strings.

Example client → server events

{
  "type": "answer.submit",
  "quizId": "quiz_123",
  "questionId": "q_07",
  "optionId": "opt_03",
  "submissionId": "sub_9a1"
}

Example server → client events

{
  "type": "quiz.question",
  "questionId": "q_07",
  "position": 7,
  "durationMs": 10000,
  "serverStartTime": 1790000000000
}

Useful event types

quiz.state
quiz.question
quiz.timer
quiz.answer.accepted
quiz.answer.rejected
quiz.score.update
quiz.leaderboard.update
quiz.paused
quiz.resumed
quiz.completed
participant.queued
participant.admitted
session.resume
error

The event protocol should be versioned when breaking changes are introduced.

20. Failure Handling and Recovery

20.1 Participant disconnect

flowchart TD
    CONNECTED[Connected] --> DROP[Network Drop]
    DROP --> RECONNECT[Automatic Reconnect]
    RECONNECT --> LOOKUP[Lookup Existing Participant Session]
    LOOKUP --> ACTIVE{Quiz Still Active?}
    ACTIVE -->|Yes| RESTORE[Restore Session State]
    ACTIVE -->|No| ENDED[Show Quiz Ended]
    RESTORE --> CONTINUE[Continue]

20.2 Reconnect behavior

The client should:

reconnect automatically,

use a stable session/participant identifier,

request the current room state,

receive the active question and authoritative deadline,

restore UI state without trusting locally stored score.

20.3 Durable Object restart/eviction

flowchart LR
    MEMORY[In-memory State] --> PERSISTED[Durable Object Storage]
    EVENT[Restart / Eviction] --> RESTORE[Read Persisted State]
    PERSISTED --> RESTORE
    RESTORE --> ROOM6[Reconstructed QuizRoom]

Memory is treated as a performance layer, not the only source of truth for state that must survive recovery.

20.4 PostgreSQL temporary failure

flowchart TD
    EVENTWRITE[Important Event] --> CHECKDB{Database Available?}
    CHECKDB -->|Yes| SAVE[Persist]
    CHECKDB -->|No| BUFFER[Durable Retry / Outbox]
    BUFFER --> RETRY[Retry]
    RETRY --> CHECKDB

The active quiz room should not fail merely because one database request temporarily fails. Persistent writes must have a defined retry/recovery strategy.

20.5 Duplicate operations

For operations that must be logically single-use:

submissionId
participantId + quizId + questionId

should be used to enforce idempotency.

21. Security

21.1 Security boundaries

flowchart LR
    INTERNET[Internet] --> EDGESEC[Cloudflare Edge]
    EDGESEC --> AUTHSEC[Authentication]
    AUTHSEC --> AUTHZ[Authorization]
    AUTHZ --> VALIDSEC[Validation]
    VALIDSEC --> APPSEC[Application Logic]
    APPSEC --> DATASEC[Data Layer]

21.2 Core security rules

Use HTTPS/WSS only.

Validate authentication on protected operations.

Enforce role-based authorization.

Never trust client score values.

Never trust client timer values.

Validate quiz/question identifiers server-side.

Prevent answer replay and duplicate submissions.

Rate-limit sensitive endpoints.

Do not expose database credentials to clients.

Do not log access tokens or refresh tokens.

Treat administrative operations as privileged.

21.3 Rate limiting targets

At minimum protect:

login/session endpoints,

join endpoints,

reconnect endpoints,

answer submission endpoints,

admin start/stop operations.

Exact limits should be tuned with load testing rather than fixed blindly.

22. Deployment

22.1 Deployment map

flowchart TD
    DEV[Developer] --> GIT[GitHub]
    GIT --> ACTIONS[GitHub Actions]
    ACTIONS --> WRANGLER[Wrangler Deploy]
    WRANGLER --> CF[Cloudflare]

    CF --> STATIC[Frontend Static Assets]
    CF --> WORKER7[Worker]
    CF --> DO7[Durable Objects]
    WORKER7 --> HYPER7[Hyperdrive]
    DO7 --> HYPER7
    HYPER7 --> SUPADB[(Supabase PostgreSQL)]

    WORKER7 --> SUPAAUTH[Supabase Auth]

22.2 Domains

Recommended production structure:

quiz.example.com      -> participant application
admin.example.com     -> admin UI (optional split)
api.example.com       -> API routing if separated

A single Worker deployment can also serve frontend assets and API routes when that is simpler for the implementation.

22.3 SSL/TLS

Traffic should be HTTPS/WSS. Cloudflare provides the edge TLS termination. Internal service credentials remain server-side.

22.4 Secrets

Secrets should live in platform secret/configuration mechanisms, not in Git.

Examples:

SUPABASE_URL
SUPABASE_ANON_KEY
SUPABASE_JWT_SECRET or supported verification configuration
DATABASE_URL / Hyperdrive binding configuration
ADMIN configuration
WEBHOOK secrets if introduced later

Use .dev.vars or equivalent local secret files for development and keep them out of Git.

23. CI/CD

23.1 CI flow

flowchart LR
    PUSH[Git Push / Pull Request] --> CI[GitHub Actions]
    CI --> TYPECHECK[Type Check]
    TYPECHECK --> LINT[Lint]
    LINT --> TEST[Unit Tests]
    TEST --> BUILD3[Build]
    BUILD3 --> DEPLOY3[Deploy on Protected Branch]

23.2 Recommended workflow

.github/
└── workflows/
    ├── test.yml
    └── deploy.yml

23.3 Branch rule

pull request -> test only,

protected main branch -> production deployment after successful checks,

event-day production changes -> controlled release only.

24. Repository Structure

Recommended repository layout:

quiz-platform/
├── frontend/
│   ├── participant/
│   ├── admin/
│   └── shared/
│
├── worker/
│   └── src/
│       ├── index.ts
│       ├── routes/
│       ├── auth/
│       ├── services/
│       └── lib/
│
├── durable-objects/
│   └── QuizRoom.ts
│
├── database/
│   ├── schema.sql
│   └── migrations/
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── load/
│
├── .github/
│   └── workflows/
│       ├── test.yml
│       └── deploy.yml
│
├── docs/
│   ├── architecture.md
│   ├── deployment.md
│   └── api.md
│
├── .gitignore
├── package.json
├── wrangler.jsonc
├── README.md
└── LICENSE

25. Environment Configuration

Example local configuration:

# Public application configuration
VITE_SUPABASE_URL=
VITE_SUPABASE_ANON_KEY=

# Worker-side bindings / secrets
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=

# Do not expose private database credentials to frontend code.

Prefer platform-native bindings for Cloudflare resources where appropriate instead of copying secrets into source code.

26. Local Development

26.1 Requirements

Recommended development tools:

Node.js LTS

npm / pnpm

Wrangler

Git

Cloudflare account for remote development/deployment

Supabase project for remote Auth/PostgreSQL integration

26.2 Install

npm install

26.3 Run frontend

npm run dev

26.4 Run Worker locally

npx wrangler dev

26.5 Typical local flow

flowchart LR
    DEVUI[Local Frontend] --> LOCALWORKER[Wrangler Dev Worker]
    LOCALWORKER --> LOCALDO[Durable Object Emulator]
    LOCALWORKER --> REMOTEDB[Development PostgreSQL]
    LOCALWORKER --> AUTHSUPA[Supabase Auth]

Use a dedicated development database/project. Do not use the production database for local experimentation.

27. Testing Strategy

27.1 Unit testing

Test:

quiz state transitions,

capacity decisions,

queue operations,

answer validation,

scoring,

deadline checks,

role authorization,

idempotency.

27.2 Integration testing

Test:

Frontend / client
      ↓
Worker
      ↓
Durable Object
      ↓
Hyperdrive
      ↓
PostgreSQL

27.3 Failure testing

Intentionally test:

participant disconnect,

reconnect during a question,

duplicate answer,

answer after deadline,

room capacity full,

queue promotion,

database temporary failure,

Durable Object recovery,

worker errors,

malformed messages,

invalid JWT,

unauthorized admin request.

28. Load Testing and Capacity

28.1 Why capacity is not guessed

Cloudflare and Supabase publish platform quotas and limits, but there is no single documented number that represents EventFlow's safe participant count per room.

Actual capacity depends on:

message rate,

number of simultaneous connections,

question broadcast size,

answer submission burst,

validation logic,

Durable Object compute time,

Durable Object storage operations,

database write rate,

reconnect spikes,

and frontend behavior.

28.2 Test levels

100 users
    ↓
250 users
    ↓
500 users
    ↓
750 users
    ↓
1000 users

28.3 Test scenarios

All users join at once.

All users connect WebSockets at once.

One question is broadcast to all users.

All users submit answers within a small burst.

Everyone reconnects after a simulated network drop.

PostgreSQL becomes slow/unavailable temporarily.

Quiz completion writes final results.

28.4 Capacity result

The event capacity should be configured only after the load test identifies a safe operating point with margin.

29. Monitoring and Logging

29.1 Important metrics

Active quiz rooms
Active WebSocket connections
Participants per room
Queue length
Join latency
Answer latency
Answers per second
Reconnect count
Worker errors
Durable Object errors
Database latency
Database failures
Quiz completion rate

29.2 Log events

Recommended structured event names:

quiz.created
quiz.opened
participant.joined
participant.queued
participant.admitted
participant.connected
participant.reconnected
answer.submitted
answer.rejected
quiz.paused
quiz.resumed
quiz.completed
result.persisted
system.error

29.3 Do not log

Passwords
Refresh tokens
JWT secrets
Database passwords
Private API keys
Raw authentication cookies

30. Cost Strategy

30.1 Development

The MVP can target a near-zero infrastructure cost when the workload stays within free quotas.

Potential development setup:

Cloudflare Worker          -> Free tier where suitable
Durable Objects            -> Free tier where suitable
Supabase Auth/PostgreSQL   -> Free project for development
R2                          -> Not required
Custom domain               -> Optional

30.2 Event production

For a serious live event, budget for paid service tiers if the event cannot tolerate free-tier pauses, quota limits, or resource constraints.

30.3 Main cost drivers

Worker request / compute usage
Durable Object request / duration usage
Durable Object storage
Hyperdrive usage
Database plan / egress
R2 storage and operations when media is added
Logging volume

30.4 Cost-control design rules

Do not store timer ticks in PostgreSQL.

Do not broadcast unnecessary messages.

Use WebSocket Hibernation where applicable.

Keep database writes meaningful and bounded.

Avoid R2 until media is required.

Perform load tests before event day.

Monitor quotas before and during the event.

Exact billing must be checked against the provider's current pricing before a real event. Architecture estimates are not billing guarantees.

31. Resolved Architecture Flaws

Previous flaw / concern

Final solution

Offline-first idea conflicted with event requirement

Final system is online-only

Traditional Socket.IO server increased infrastructure

Durable Object owns the live room

Redis was being considered as a core requirement

Redis is not required for the initial architecture

Worker expected to hold full live-session state

QuizRoom owns live state

Client could potentially control timer

Server-authoritative deadline

Client could potentially manipulate score

Server-authoritative scoring

Queue depended on frontend

Queue belongs to QuizRoom

In-memory state could disappear

Important state persists through Durable Object storage

Every live event could hit PostgreSQL

Live coordination is handled by Durable Object; only important data is persisted

Duplicate answers could be created

Idempotency key / logical uniqueness

Disconnect could lose session context

Reconnect and session restoration flow

Worker-to-PostgreSQL connection handling was unclear

Hyperdrive is the database connection layer

Auth traffic could spike at quiz start

Pre-event authentication / valid sessions before quiz start

R2 was being treated as necessary

R2 is optional and delayed until media/object use cases appear

Capacity was being guessed

Capacity is established through load testing

Failure behavior was not explicit

Recovery, retry, reconnect, and failure states are defined

32. Event-Day Checklist

Infrastructure

Production Cloudflare deployment verified.

Production domain configured.

HTTPS/WSS verified.

Durable Object bindings verified.

Hyperdrive configuration verified.

Production PostgreSQL accessible.

Supabase Auth Google OAuth redirect URLs verified.

Production secrets configured.

Quiz setup

Quiz created.

Questions reviewed.

Correct options reviewed.

Capacity set from load-test results.

Duration configured.

Admin accounts verified.

Test participant account verified.

Reliability

Reconnect tested.

Duplicate answer protection tested.

Queue admission tested.

Final-result persistence tested.

Database failure behavior tested.

Event logging visible.

Event rehearsal

Full end-to-end rehearsal completed.

Peak-user load test completed.

Participant instructions prepared.

Admin emergency controls tested.

Backup communication channel prepared.

33. Development Roadmap

flowchart LR
    P1[Phase 1<br/>Frontend + Worker] --> P2[Phase 2<br/>Auth + Quiz APIs]
    P2 --> P3[Phase 3<br/>QuizRoom]
    P3 --> P4[Phase 4<br/>WebSockets]
    P4 --> P5[Phase 5<br/>PostgreSQL + Hyperdrive]
    P5 --> P6[Phase 6<br/>Queue + Capacity]
    P6 --> P7[Phase 7<br/>Admin Dashboard]
    P7 --> P8[Phase 8<br/>Security + Recovery]
    P8 --> P9[Phase 9<br/>Load Testing]
    P9 --> P10[Phase 10<br/>Production Event]

Phase definitions

Phase

Deliverable

Done when

1

Client + Worker

Frontend communicates with Worker

2

Authentication + APIs

Protected user and quiz endpoints work

3

QuizRoom

One active quiz can maintain live state

4

WebSockets

Question and answer events work in real time

5

Database

Persistent quiz and result data works

6

Queue

Capacity limits and waiting users work

7

Admin

Admin can create/manage/run a quiz

8

Recovery

Reconnect and failure handling work

9

Load testing

Safe room capacity is measured

10

Production

Event rehearsal succeeds end-to-end

34. MVP vs Final Version

MVP

Google Login
    ↓
Create Quiz
    ↓
Join Quiz
    ↓
Single QuizRoom
    ↓
WebSocket
    ↓
Question + Timer
    ↓
Answer + Score
    ↓
PostgreSQL Result

Final target

flowchart LR
    AUTHFLOW[Auth] --> QUIZFLOW[Quiz Management]
    QUIZFLOW --> ROOMFLOW[QuizRoom]
    ROOMFLOW --> REALTIME[WebSockets]
    ROOMFLOW --> QUEUEFLOW[Queue]
    ROOMFLOW --> SCOREBOARD[Scoring / Leaderboard]
    ROOMFLOW --> RECOVERY[Reconnect / Recovery]
    ROOMFLOW --> PERSIST[Persistent Results]
    PERSIST --> ANALYTICS[Admin Results / Analytics]

Not required initially

Multiple cloud providers

Complex microservices

Dedicated Redis cluster

Separate WebSocket server

Kubernetes

Multi-region deployment

Advanced media pipelines

Full-scale analytics warehouse

These can be added only when actual requirements justify them.

35. Engineering Rules

Keep EventFlow online-only.

Worker handles API/control traffic.

One Durable Object coordinates each active quiz room.

WebSockets are the real-time transport.

Timer deadlines are authoritative on the server.

Scores are calculated on the server.

Queue state is controlled by the room.

PostgreSQL stores permanent application data.

Do not write every live UI update to PostgreSQL.

Important live state must survive DO restarts.

All privileged operations are authenticated and authorized.

Use idempotency for answer submission and critical commands.

Do not claim an event capacity without load testing.

Keep R2 optional until the platform needs object/media storage.

Keep provider quotas separate from EventFlow's measured performance.

36. Final Architecture

36.1 Final system

flowchart TD
    USERS2[Participants + Admins] --> EDGE4[Cloudflare Edge]
    EDGE4 --> WORKER8[Cloudflare Worker]

    WORKER8 --> AUTH4[Supabase Auth]
    WORKER8 --> ROOM7[QuizRoom Durable Object]
    WORKER8 --> HYPER8[Hyperdrive]

    ROOM7 <--> SOCKETS2[WebSockets]
    SOCKETS2 --> USERS2
    ROOM7 --> STATE2[Durable Live State]
    ROOM7 --> HYPER8

    HYPER8 --> POSTGRES[(Supabase PostgreSQL)]

    WORKER8 --> LOGGING[Cloudflare Logging]

36.2 Final runtime rule

Participant / Admin
        ↓
Cloudflare Edge
        ↓
Cloudflare Worker
        ↓
+-----------------------------+
|                             |
|  API / Auth / Validation    |
|                             |
+-----------------------------+
        ↓
QuizRoom Durable Object
        ↓
WebSocket ↔ Participants
        ↓
Live Quiz State
        ↓
Hyperdrive
        ↓
PostgreSQL

36.3 One-line architecture

Users → Cloudflare Edge → Worker → QuizRoom Durable Object → WebSocket Participants
                                      |
                                      +→ Hyperdrive → PostgreSQL

Conclusion

EventFlow is intentionally built as a small number of clearly separated responsibilities rather than a large collection of services.

The final architecture keeps the project close to the original Cloudflare-first direction while solving the main design flaws:

real-time coordination is centralized per quiz room,

client authority is removed from timer and scoring decisions,

live state is separated from permanent data,

reconnect and recovery are part of the core design,

database writes are controlled,

queue and capacity belong to the server-side room,

R2 remains optional,

and room capacity is established experimentally through load testing instead of being guessed from provider marketing or quotas.

The implementation should therefore proceed from Worker/API → QuizRoom → WebSockets → PostgreSQL/Hyperdrive → Queue → Admin → Recovery → Load Testing → Production.
