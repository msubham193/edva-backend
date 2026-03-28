# APEXIQ Backend — NestJS Monolith

JEE/NEET Battle Learning Platform — Complete backend starter.

## Stack

| Layer | Tech |
|-------|------|
| Framework | NestJS 10 + TypeScript |
| Database | PostgreSQL 16 + TypeORM |
| Cache / Sessions | Redis 7 |
| Real-time | Socket.io (Battle Arena) |
| Auth | JWT (access + refresh) + OTP (Twilio) |
| Background Jobs | BullMQ + @nestjs/schedule |
| Media | Cloudflare R2 / AWS S3 |
| Push | Firebase FCM |
| SMS / WhatsApp | Twilio |
| API Docs | Swagger (auto-generated) |

---

## Quick Start

### 1. Clone and install

```bash
npm install
cp .env.example .env
# Edit .env with your DB and Redis credentials
```

### 2. Start services (Docker)

```bash
docker run --name pg -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=apexiq -p 5432:5432 -d postgres:16
docker run --name redis -p 6379:6379 -d redis:7
```

### 3. Run migrations

```bash
npm run migration:run
```

### 4. Start dev server

```bash
npm run start:dev
```

- API: `http://localhost:3000/api/v1`
- Swagger Docs: `http://localhost:3000/docs`
- Battle WS: `ws://localhost:3000/battle`

---

## Module Structure

```
src/
├── common/
│   ├── decorators/      # @CurrentUser, @Public, @TenantId, @Roles
│   ├── filters/         # Global exception filter
│   ├── guards/          # JwtAuthGuard, RolesGuard
│   ├── interceptors/    # Response wrapper interceptor
│   └── middleware/      # TenantMiddleware (multi-tenancy)
│
├── config/              # App, JWT, Redis, AI, OTP, Storage config
│
├── database/
│   ├── entities/        # All 20+ TypeORM entities
│   └── migrations/      # SQL migrations
│
└── modules/
    ├── auth/            # OTP login, JWT, onboarding       ✅ COMPLETE
    ├── student/         # Dashboard, weak topics, streak   ✅ SCAFFOLD
    ├── battle/          # ELO battles + Socket.io gateway  ✅ COMPLETE
    ├── ai-bridge/       # All 12 AI service adapters       ✅ COMPLETE
    ├── content/         # Lectures, questions, notes        🔲 TODO
    ├── assessment/      # Mock tests, gate lock, grading    🔲 TODO
    ├── analytics/       # Leaderboard, rank prediction      🔲 TODO
    └── notification/    # Push, WhatsApp, SMS               🔲 TODO
```

---

## Authentication Flow

```
POST /api/v1/auth/otp/send    { phoneNumber }
→ OTP sent via Twilio SMS (or console in dev mode — OTP is always 123456)

POST /api/v1/auth/otp/verify  { phoneNumber, otp }
→ { accessToken, refreshToken, user, isNewUser, onboardingRequired }

POST /api/v1/auth/onboard     (requires Bearer token)
→ { examTarget, class, examYear, dailyStudyHours, language, city }
→ Creates Student record + PerformanceProfile + ELO

POST /api/v1/auth/refresh     { refreshToken }
→ { accessToken, refreshToken }
```

---

## Multi-Tenancy

Every request goes through `TenantMiddleware` which resolves `tenant_id` from:
1. `X-Tenant-ID` header (admin/internal calls)
2. Subdomain (e.g. `allen-kota.apexiq.in` → subdomain = `allen-kota`)
3. Falls back to platform tenant (B2C students)

All database queries automatically scoped by `tenant_id`.
PostgreSQL Row-Level Security policies are enabled in migration.

---

## Battle Arena WebSocket

Connect: `ws://localhost:3000/battle`

```javascript
// Events Client → Server
socket.emit('battle:join',   { roomCode, studentId })
socket.emit('battle:answer', { roomCode, battleId, questionId, optionId, roundNumber, responseTimeMs, studentId })

// Events Server → Client
socket.on('battle:player_joined', ({ participants }) => ...)
socket.on('battle:start',        ({ battle, firstQuestion, totalRounds, timePerRound }) => ...)
socket.on('battle:round_result', ({ roundNumber, winnerId, correctOptionId, scores }) => ...)
socket.on('battle:question',     ({ question, roundNumber, timeLimit }) => ...)
socket.on('battle:end',          ({ winnerId, finalScores, eloChanges }) => ...)
socket.on('battle:opponent_left',({ message }) => ...)
socket.on('battle:error',        ({ message }) => ...)
```

---

## AI Services (AI Bridge)

All 12 AI services are called via `AiBridgeService`. Inject it into any module:

```typescript
constructor(private readonly ai: AiBridgeService) {}

// Resolve a doubt
const result = await this.ai.resolveDoubt({
  questionText: 'Why is entropy always positive?',
  topicId: 'uuid',
  mode: 'detailed',
});

// Generate study plan
const plan = await this.ai.generateStudyPlan({
  studentId: 'uuid',
  examTarget: 'jee',
  examYear: '2026',
  dailyHours: 4,
  weakTopics: ['topic-uuid-1', 'topic-uuid-2'],
});
```

---

## Environment Variables

See `.env.example` for all variables. Key ones:

| Variable | Description |
|----------|-------------|
| `DB_*` | PostgreSQL connection |
| `REDIS_*` | Redis connection |
| `JWT_SECRET` | Access token signing key |
| `JWT_REFRESH_SECRET` | Refresh token signing key |
| `OTP_DEV_MODE=true` | Use fixed OTP `123456` in development |
| `AI_BASE_URL` | Your AI services base URL |
| `AI_API_KEY` | API key for AI services |
| `TWILIO_*` | SMS/WhatsApp credentials |

---

## Next Modules to Build

1. **Content Module** — lecture CRUD, video upload to R2, STT trigger
2. **Assessment Module** — quiz gate lock engine, adaptive questions, mock test session
3. **Analytics Module** — leaderboard computation (CRON), rank prediction
4. **Notification Module** — BullMQ queue, FCM push, Twilio WhatsApp

Each module follows identical structure to `auth/` and `student/`.
