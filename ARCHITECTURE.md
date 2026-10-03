# Online Quiz Platform — System Architecture

> **Status:** Updated architecture
> **Mode:** Online-only, real-time quiz platform
> **Core stack:** Cloudflare Workers + Durable Objects + WebSockets + Hyperdrive + PostgreSQL

---

## 1. System Overview

The platform is an **online real-time quiz system** where participants and admins interact through a web application. Cloudflare Workers handle application/API requests, Durable Objects manage live quiz sessions and real-time coordination, and PostgreSQL stores persistent application data.

### High-Level Architecture

```text
Users
  |
  | HTTPS / WSS
  v
Cloudflare Edge
  |
  v
Cloudflare Worker
  |
  +----------------------+----------------------+
  |                      |                      |
  v                      v                      v
Authentication     Quiz APIs              Admin APIs
                         |
                         v
                  Durable Object
                     QuizRoom
                         |
             +-----------+-----------+
             |                       |
             v                       v
         WebSockets              Live State
             |                       |
             v                       |
       Participants                |
                                     |
                                     v
                                Hyperdrive
                                     |
                                     v
                                PostgreSQL
```

---

# 2. Level 1 — Client Architecture

This level defines the two main client types: **Participant** and **Admin**.

```text
                    ONLINE QUIZ PLATFORM
                           |
              +------------+------------+
              |                         |
        PARTICIPANT                  ADMIN
              |                         |
        +-----+-----+             +-----+-----+
        |           |             |           |
      Login       Quiz UI       Login      Dashboard
        |           |             |           |
        |      +----+----+        |      +----+----+
        |      |         |        |      |         |
        |   Question   Timer      |   Quiz Mgmt  Results
        |      |         |        |      |         |
        |    Answer    Score      |    Users    Analytics
        +------+---------+        +------+---------+
```

### Participant UI

- Registration / login
- Join quiz
- Waiting/queue screen
- Live question screen
- Timer
- Answer submission
- Score/progress
- Leaderboard
- Final result

### Admin UI

- Admin login
- Quiz creation/editing
- Question management
- Quiz start/stop
- Capacity control
- Live monitoring
- Participant management
- Result viewing

---

# 3. Level 2 — Network & Cloudflare Edge Architecture

```text
                 INTERNET
                    |
                    | HTTPS
                    v
          +-------------------+
          |       DNS         |
          |   quiz.example    |
          +---------+---------+
                    |
                    v
          +-------------------+
          |  CLOUDFLARE EDGE  |
          |                   |
          | DNS               |
          | SSL / HTTPS       |
          | DDoS Protection   |
          | WAF / Routing     |
          +---------+---------+
                    |
                    v
          +-------------------+
          | CLOUDFLARE WORKER |
          +---------+---------+
                    |
             Application Layer
```

### Request Flow

```text
Browser
  |
  | HTTPS request
  v
DNS
  |
  v
Cloudflare Edge
  |
  +--> Security / TLS / routing
  |
  v
Cloudflare Worker
```

### Responsibilities

| Component | Responsibility |
|---|---|
| DNS | Resolves application domain |
| TLS/HTTPS | Encrypts client-server communication |
| Cloudflare Edge | Receives and routes traffic at the edge |
| WAF / DDoS controls | Protects the public application |
| Worker | Sends the request into application logic |

---

# 4. Level 3 — Worker / Backend Architecture

The Worker is the main backend entry point.

```text
                    CLOUDFLARE WORKER
                           |
                           v
                  +-----------------+
                  | Request Router  |
                  +--------+--------+
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
   +------------+   +------------+   +------------+
   |    Auth    |   | Quiz APIs  |   | Admin APIs |
   |            |   |            |   |            |
   | Login      |   | Join Quiz  |   | Create Quiz|
   | Register   |   | Questions  |   | Start/Stop |
   | Session    |   | Submit     |   | Manage     |
   +------------+   +------+-----+   +------------+
                           |
                           v
                  +-----------------+
                  |   Validation    |
                  |                 |
                  | Auth Check     |
                  | Quiz Check     |
                  | Input Check    |
                  +--------+--------+
                           |
                  +--------+--------+
                  |                 |
                  v                 v
          Durable Object        PostgreSQL
```

### Worker Responsibilities

```text
Receive Request
      |
      v
Identify Route
      |
      v
Authenticate User
      |
      v
Validate Request
      |
      +----> Durable Object for live quiz operations
      |
      +----> PostgreSQL for persistent data operations
      |
      v
Return Response
```

### Example Route

```text
POST /quiz/123/join
        |
        v
      Worker
        |
        v
   Authentication
        |
        v
   Quiz Validation
        |
        v
     QuizRoom DO
        |
        v
  Capacity / Queue
        |
        v
     Response
```

---

# 5. Level 4 — Real-Time Quiz Architecture

Each active quiz/session is coordinated through a Durable Object instance such as `QuizRoom`.

```text
                  CLOUDFLARE WORKER
                         |
                         v
              +----------------------+
              |   DURABLE OBJECT     |
              |      QuizRoom        |
              |                      |
              | • Quiz State         |
              | • Participants       |
              | • Queue              |
              | • Timer              |
              | • Question State     |
              | • Answer Validation  |
              | • Scoring            |
              | • Leaderboard        |
              +----------+-----------+
                         |
                    WebSocket
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
   Participant 1    Participant 2    Participant N
```

### Real-Time Flow

```text
Participant
    |
    | WebSocket connection
    v
Cloudflare Worker
    |
    v
QuizRoom Durable Object
    |
    +--> Admit / Queue
    +--> Maintain Session
    +--> Send Question
    +--> Manage Timer
    +--> Receive Answer
    +--> Calculate Score
    +--> Broadcast Updates
             |
             v
       All Participants
```

### Example Question Cycle

```text
QuizRoom
   |
   v
Send Question #5
   |
   +-------> Participant 1
   +-------> Participant 2
   +-------> Participant 3
              |
              v
          Submit Answer
              |
              v
          QuizRoom
              |
         Validate Answer
              |
         Calculate Score
              |
              v
          Broadcast
              |
              v
        All Participants
```

### Live Session State

```text
QuizRoom
 |
 +-- currentQuestion
 +-- questionStartTime
 +-- participants
 +-- answers
 +-- scores
 +-- queue
 +-- quizStatus
 +-- leaderboard
```

---

# 6. Level 5 — Data Architecture

The design separates **live session state** from **persistent application data**.

```text
                    CLOUDFLARE WORKER
                           |
                +----------+----------+
                |                     |
                v                     v
       +-----------------+    +------------------+
       |   QuizRoom DO   |    |    HYPERDRIVE    |
       |                 |    |                  |
       | Live session    |    | DB connection    |
       | Participants    |    | layer            |
       | Queue           |    +--------+---------+
       | Current Q       |             |
       | Timer state     |             v
       | Live scores     |    +------------------+
       +-----------------+    |    PostgreSQL    |
                              |                  |
                              | Users            |
                              | Quizzes          |
                              | Questions        |
                              | Answers          |
                              | Results          |
                              | Events / Logs    |
                              +------------------+
```

### Data Ownership

#### Durable Object — Live / Session State

```text
- Connected participants
- Queue
- Current question
- Timer state
- Live answers
- Live scores
- WebSocket/session state
- Active quiz status
```

#### PostgreSQL — Persistent Data

```text
- Users
- Roles
- Quiz metadata
- Questions
- Options
- Submitted answers
- Scores
- Final results
- Admin data
- Audit / event data
```

### Storage Rule

```text
LIVE QUIZ STATE
      |
      v
Durable Object

PERMANENT APPLICATION DATA
      |
      v
PostgreSQL

WORKER <----> Hyperdrive <----> PostgreSQL
```

---

# 7. Level 6 — Complete Online Quiz Flow

```text
                    PARTICIPANT
                         |
                         v
                    LOGIN / AUTH
                         |
                         v
                    JOIN QUIZ
                         |
                         v
              +---------------------+
              |   CAPACITY CHECK    |
              +----------+----------+
                         |
                 +-------+-------+
                 |               |
              SPACE            FULL
                 |               |
                 v               v
             ADMIT USER      WAITING QUEUE
                 |               |
                 |        Slot becomes free
                 |               |
                 +-------+-------+
                         |
                         v
                 QUIZROOM DURABLE OBJECT
                         |
                         v
                  QUIZ STARTED?
                    |          |
                   NO         YES
                    |          |
                    |          v
                    |       QUESTION
                    |          |
                    |          v
                    |         TIMER
                    |          |
                    |          v
                    |       ANSWER
                    |          |
                    |          v
                    |    VALIDATE ANSWER
                    |          |
                    |          v
                    |      CALCULATE SCORE
                    |          |
                    |          v
                    |    SAVE / UPDATE DATA
                    |          |
                    |          v
                    |      NEXT QUESTION
                    |          |
                    +----------+
                               |
                         ALL QUESTIONS
                               |
                               v
                         FINAL SCORE
                               |
                               v
                       STORE RESULT
                               |
                               v
                         LEADERBOARD
                               |
                               v
                           RESULT UI
```

### Backend Interaction

```text
Participant
    |
    v
Worker
    |
    v
QuizRoom DO
    |
    +-- Queue / Admission
    +-- Quiz State
    +-- Timer
    +-- WebSocket
    +-- Answer Handling
    +-- Live Score
    |
    +-----------------------> PostgreSQL
    |                           |
    |                           +--> Permanent results/data
    |
    +-----------------------> Participants
                                |
                                +--> Question
                                +--> Timer
                                +--> Score
                                +--> Leaderboard
```

---

# 8. Level 7 — Admin & Management Architecture

```text
                         ADMIN
                           |
                           v
                  +-----------------+
                  |  ADMIN DASHBOARD |
                  +--------+--------+
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
        Quiz Manager   User Manager   Live Monitor
             |             |             |
             v             v             v
        Create/Edit     View Users    Active Quiz
        Questions       Roles         Participants
        Start/Stop      Status        Queue
             |             |             |
             +-------------+-------------+
                           |
                           v
                  CLOUDFLARE WORKER
                           |
             +-------------+-------------+
             |                           |
             v                           v
      DURABLE OBJECT                POSTGRESQL
             |                           |
      Live quiz control          Permanent data
      Start / Stop               Quizzes
      Capacity                   Questions
      Queue                      Users
      Monitoring                 Results
```

### Admin Operations

```text
CREATE QUIZ
    |
    v
POSTGRESQL

ADD QUESTIONS
    |
    v
POSTGRESQL

START QUIZ
    |
    v
DURABLE OBJECT

CONTROL CAPACITY
    |
    v
DURABLE OBJECT

MONITOR LIVE USERS
    |
    v
DURABLE OBJECT

END QUIZ
    |
    v
FINAL RESULTS
    |
    v
POSTGRESQL
```

---

# 9. Level 8 — Security Architecture

```text
                    USER / ADMIN
                         |
                         v
                    HTTPS / TLS
                         |
                         v
                 CLOUDFLARE EDGE
                         |
              +----------+----------+
              |                     |
             DDoS                WAF /
          Protection           Rate Limit
              |                     |
              +----------+----------+
                         |
                         v
                 CLOUDFLARE WORKER
                         |
                  Authentication
                         |
                  Authorization
                         |
                  Input Validation
                         |
                  Request Checks
                         |
              +----------+----------+
              |                     |
              v                     v
        Durable Object          PostgreSQL
              |                     |
        Session control       Persistent data
```

### Security Flow

```text
Request
  |
  v
HTTPS
  |
  v
Cloudflare Security
  |
  v
Authentication
  |
  v
Authorization / Role Check
  |
  v
Input Validation
  |
  v
Rate Limiting
  |
  v
Worker
  |
  +--> Durable Object
  |
  +--> PostgreSQL
```

### Access Control

```text
                 AUTHENTICATED USER
                        |
              +---------+---------+
              |                   |
              v                   v
        PARTICIPANT             ADMIN
              |                   |
       Join / Answer       Create / Manage
       View Result         Monitor / Results
              |                   |
              +---------+---------+
                        |
                        v
                   ROLE CHECK
```

### Main Protection Layers

```text
HTTPS/TLS
   |
   v
DDoS Protection
   |
   v
WAF / Rate Limiting
   |
   v
Authentication
   |
   v
Role-Based Authorization
   |
   v
Input Validation
   |
   v
Secure Database Access
```

---

# 10. Level 9 — Deployment Architecture

```text
                         GitHub Repository
                                |
                                v
                       GitHub Actions / CI-CD
                                |
                       Build + Test + Deploy
                                |
                                v
                     +----------------------+
                     |   CLOUDFLARE         |
                     |                      |
                     | Static Assets        |
                     | Worker               |
                     | Durable Objects      |
                     +----------+-----------+
                                |
                                v
                           Hyperdrive
                                |
                                v
                           PostgreSQL
```

### Deployment Flow

```text
Developer
   |
   v
Git Push
   |
   v
GitHub
   |
   v
GitHub Actions
   |
   +-- Install dependencies
   +-- Lint
   +-- Test
   +-- Build frontend
   +-- Deploy
           |
           v
     Cloudflare Workers
           |
      +----+----+
      |         |
      v         v
   Frontend   Backend
               Worker
                 |
                 v
           Durable Object
                 |
                 v
              Hyperdrive
                 |
                 v
             PostgreSQL
```

### Suggested Repository Structure

```text
quiz-platform/
|
+-- frontend/
|   +-- participant/
|   +-- admin/
|
+-- worker/
|   +-- src/
|
+-- durable-objects/
|   +-- QuizRoom.ts
|
+-- tests/
|
+-- .github/
|   +-- workflows/
|       +-- deploy.yml
|
+-- wrangler.jsonc
+-- README.md
+-- ARCHITECTURE.md
```

---

# 11. Level 10 — Full System Architecture

```text
                                  +--------------------------+
                                  |          USERS           |
                                  |                          |
                                  | Participant |   Admin   |
                                  +-------------+------------+
                                                |
                                           HTTPS / WSS
                                                |
                                                v
                              +------------------------------+
                              |      CLOUDFLARE EDGE         |
                              |                              |
                              | DNS | TLS | WAF | DDoS      |
                              | Rate Limiting | Routing      |
                              +--------------+---------------+
                                             |
                                             v
                              +------------------------------+
                              |      CLOUDFLARE WORKER       |
                              |                              |
                              | Request Router               |
                              | Authentication              |
                              | Authorization               |
                              | Input Validation             |
                              | Quiz APIs                    |
                              | Admin APIs                   |
                              +--------------+---------------+
                                             |
                +----------------------------+----------------------------+
                |                            |                            |
                v                            v                            v
        +------------------+       +------------------+        +------------------+
        | AUTH / USER      |       | QUIZ SESSION     |        | ADMIN SYSTEM     |
        |                  |       |                  |        |                  |
        | Login            |       | Join Quiz        |        | Create Quiz      |
        | Register         |       | Queue            |        | Questions        |
        | Session          |       | Capacity         |        | Start / Stop     |
        | Role             |       | Live State       |        | Monitor          |
        +---------+--------+       +---------+--------+        +---------+--------+
                  |                          |                          |
                  |                          v                          |
                  |               +----------------------+              |
                  |               |   DURABLE OBJECT     |              |
                  |               |      QuizRoom        |              |
                  |               |                      |              |
                  |               | Participants         |              |
                  |               | Queue                |              |
                  |               | Quiz State           |              |
                  |               | Current Question     |              |
                  |               | Timer                |              |
                  |               | Answers              |              |
                  |               | Live Score           |              |
                  |               | Leaderboard          |              |
                  |               | WebSockets           |              |
                  |               +-----------+----------+              |
                  |                           |                         |
                  |                     Real-time                       |
                  |                      WebSocket                       |
                  |                           |                         |
                  |              +------------+------------+            |
                  |              |            |            |            |
                  |              v            v            v            |
                  |             P1           P2           P3           |
                  |          Participant  Participant  Participant     |
                  |                                                   |
                  +-----------------------------+---------------------+
                                                |
                                                v
                                     +----------------------+
                                     |      HYPERDRIVE      |
                                     | DB connection layer  |
                                     +----------+-----------+
                                                |
                                                v
                                     +----------------------+
                                     |      POSTGRESQL      |
                                     |                      |
                                     | Users                |
                                     | Roles                |
                                     | Quizzes              |
                                     | Questions            |
                                     | Options              |
                                     | Answers              |
                                     | Scores               |
                                     | Results              |
                                     | Events / Logs        |
                                     +----------------------+
```

---

# 12. End-to-End Runtime Flow

```text
Participant opens website
        |
        v
Cloudflare DNS / Edge
        |
        v
Worker
        |
        v
Authentication
        |
        v
Join Quiz API
        |
        v
QuizRoom Durable Object
        |
        +---- Capacity available ----> Admit
        |
        +---- Capacity full ---------> Queue
        |
        v
WebSocket connection
        |
        v
Quiz starts
        |
        v
Question + timer broadcast
        |
        v
Participant submits answer
        |
        v
QuizRoom validates + scores
        |
        +----> Live update via WebSocket
        |
        +----> Persist required data to PostgreSQL
        |
        v
Next question
        |
        v
Final score
        |
        v
Leaderboard / Result
```

---

# 13. Component Responsibilities

| Component | Main Responsibility |
|---|---|
| Participant Frontend | Join quiz, answer questions, view timer/score/results |
| Admin Frontend | Manage quizzes, questions, participants and results |
| DNS | Domain resolution |
| Cloudflare Edge | Traffic entry point, TLS, security, routing |
| Cloudflare Worker | API/backend logic and request routing |
| Durable Object / QuizRoom | Per-quiz live state and real-time coordination |
| WebSocket | Real-time communication between QuizRoom and clients |
| Hyperdrive | Worker-to-PostgreSQL database connectivity layer |
| PostgreSQL | Persistent application data |
| GitHub Actions | CI/CD automation |
| GitHub | Source-code repository |

---

# 14. Final Design Rules

```text
1. The platform is ONLINE-ONLY.

2. Cloudflare Worker is the main backend/API entry point.

3. Durable Object manages live state for each active quiz/session.

4. WebSocket is used for real-time quiz communication.

5. Capacity control and waiting queue are handled in the live quiz session.

6. PostgreSQL stores permanent application data.

7. Hyperdrive connects the Worker to PostgreSQL.

8. Participant and Admin have separate application flows and permissions.

9. Security is enforced at the edge and application layers.

10. CI/CD deploys the application through the GitHub workflow.
```

---

# 15. Architecture in One Line

```text
Participants / Admin
        |
        v
Cloudflare Edge
        |
        v
Cloudflare Worker
        |
        v
Durable Object (QuizRoom)
        |
        +---- WebSocket ----> Participants
        |
        v
Hyperdrive
        |
        v
PostgreSQL
```

---

## 16. Architecture Summary

The final system uses **Cloudflare as the application edge**, **Workers as the backend/API layer**, **one Durable Object instance per active quiz/session for live coordination**, **WebSockets for real-time communication**, and **PostgreSQL for persistent data through Hyperdrive**.

The architecture is designed specifically for an **online real-time quiz platform**, including authentication, admission control, queueing, live questions, timers, answer handling, scoring, leaderboard updates, administration, security, and production deployment.
