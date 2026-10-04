quiz-platform/
├── frontend/
│   ├── participant/
│   ├── admin/
│   └── shared/
│
├── worker/
│   ├── src/
│   │   ├── index.ts
│   │   ├── routes/
│   │   ├── auth/
│   │   ├── services/
│   │   ├── validation/
│   │   ├── queue/
│   │   └── lib/
│   └── wrangler.jsonc
│
├── durable-objects/
│   ├── QuizCoordinator.ts
│   └── QuizShard.ts
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
│       └── deploy.yml
│
├── README.md
└── ARCHITECTURE.md
