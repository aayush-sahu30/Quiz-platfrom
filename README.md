EventFlow

Real-time online quiz platform for college events.

Features

Online real-time quizzes

WebSocket-based live communication

Durable Object per quiz room

Waiting queue and capacity control

Server-authoritative timer and scoring

PostgreSQL for persistent data

Google OAuth authentication

Admin dashboard

Live leaderboard

Reconnection and failure recovery

Architecture

flowchart LR
    U[Participants / Admin] --> CF[Cloudflare]
    CF --> W[Cloudflare Worker]
    W --> DO[Durable Object]
    DO <--> WS[WebSocket Clients]
    W --> HD[Hyperdrive]
    HD --> PG[(PostgreSQL)]
    W --> AUTH[Supabase Auth]

Detailed architecture: ARCHITECTURE.md

Main Quiz Flow

flowchart LR
    A[Login] --> B[Join Quiz]
    B --> C{Capacity Available?}
    C -->|Yes| D[Admit Participant]
    C -->|No| E[Waiting Queue]
    E --> F[Slot Available]
    F --> D
    D --> G[Quiz Room]
    G --> H[Question]
    H --> I[Timer]
    I --> J[Answer]
    J --> K[Validate Answer]
    K --> L[Calculate Score]
    L --> M[Leaderboard]
    M --> N[Next Question]
    N --> H
    M --> O[Final Result]

Real-Time Flow

sequenceDiagram
    participant P as Participant
    participant W as Worker
    participant D as QuizRoom
    participant DB as PostgreSQL

    P->>W: Join Quiz
    W->>D: Join Request
    D-->>P: WebSocket Connected
    D->>P: Question + Start Time
    P->>D: Submit Answer
    D->>D: Validate + Score
    D-->>P: Score Update
    D->>DB: Persist Required Data

Tech Stack

Layer

Technology

Frontend

React / TypeScript

Backend

Cloudflare Workers

Real-time

Durable Objects + WebSockets

Database

PostgreSQL

DB Connection

Cloudflare Hyperdrive

Authentication

Supabase Auth

Storage

R2 (optional)

CI/CD

GitHub Actions

Data Responsibilities

Component

Responsibility

Cloudflare Worker

API routing, validation, admin APIs

Durable Object

Live quiz state, queue, timer, scoring

WebSocket

Real-time communication

PostgreSQL

Users, quizzes, questions, answers, results

Hyperdrive

Worker-to-PostgreSQL connectivity

Supabase Auth

Google OAuth and sessions

R2

Optional media/object storage

Security

flowchart TD
    A[User Request] --> B[HTTPS]
    B --> C[Cloudflare Security]
    C --> D[Worker]
    D --> E[Authentication]
    E --> F[Authorization]
    F --> G[Input Validation]
    G --> H[Rate Limiting]
    H --> I[Durable Object / PostgreSQL]

Failure Recovery

flowchart TD
    A[Participant Connected] --> B[WebSocket]
    B --> C{Connection Stable?}
    C -->|Yes| D[Continue Quiz]
    C -->|No| E[Reconnect]
    E --> F[Restore Session]
    F --> G[Recover Quiz State]
    G --> H[Send Current Question]
    H --> D

Deployment

flowchart LR
    DEV[Developer] --> GH[GitHub]
    GH --> CI[GitHub Actions]
    CI --> CF[Cloudflare Workers]
    CF --> DO[Durable Objects]
    CF --> HD[Hyperdrive]
    HD --> DB[(PostgreSQL)]
    CF --> AUTH[Supabase Auth]

Project Structure

eventflow/
├── frontend/
│   ├── participant/
│   └── admin/
├── worker/
│   └── src/
│       ├── routes/
│       ├── auth/
│       ├── durable-objects/
│       │   └── QuizRoom.ts
│       └── lib/
├── tests/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── wrangler.jsonc
├── package.json
├── README.md
└── ARCHITECTURE.md

Development

git clone <repository-url>
cd eventflow
npm install
npm run dev

Capacity Testing

The final room capacity must be determined by load testing.

100 users
   ↓
250 users
   ↓
500 users
   ↓
750 users
   ↓
1000 users

These are engineering test targets, not guaranteed platform limits.

Cost Strategy

Development

Use Cloudflare and Supabase free tiers where practical.

Keep R2 disabled unless media storage is required.

Avoid unnecessary database writes and log volume.

Event

Move to paid resources when required for reliability, quota, or production guarantees.

Main possible billing sources:

Cloudflare Workers

Durable Objects

Hyperdrive

PostgreSQL / Supabase

R2

Excessive logging

MVP

The first release should support:

Login

Create and join quiz

Real-time questions

Timer

Answer submission

Automatic scoring

Queue

Capacity control

Leaderboard

Final results

Admin controls

Documentation

Architecture
