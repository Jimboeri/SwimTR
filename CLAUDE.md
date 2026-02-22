# CLAUDE.md — SwimSafe Project

## Project Overview

SwimSafe is a web application that acts as a safety assistant for open water swimmers. Users register planned swims, share their location, and check in when they finish. If a swimmer fails to check in within their estimated window, the system automatically notifies their emergency contact.

## Tech Stack

- **Frontend:** React (Vite), TailwindCSS
- **Backend:** Node.js with Express
- **Database:** PostgreSQL (via Prisma ORM)
- **Notifications:** Twilio (SMS) and SendGrid (email)
- **Hosting:** Railway (backend + DB), Vercel (frontend)
- **Auth:** Clerk (managed auth, handles JWT)

## Project Structure

```
swimsafe/
├── client/               # React frontend (Vite)
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── hooks/
│   │   └── lib/          # API client, utils
├── server/               # Express backend
│   ├── routes/
│   ├── services/         # Business logic (notifications, swim state)
│   ├── jobs/             # Scheduled jobs (overdue swim checker)
│   ├── prisma/
│   │   └── schema.prisma
│   └── index.ts
└── CLAUDE.md
```

## Key Domain Concepts

- **Swim:** A planned open water swim session created by a user. Has a start time, estimated duration, and location.
- **ICE Contact (In Case of Emergency):** A named person with phone/email that a user registers in their profile. Notified if a swim is not completed in time.
- **Check-in:** The action a user takes when they have safely finished a swim.
- **Overdue Swim:** A swim where the expected end time has passed and the user has not checked in.

## Common Commands

```bash
# Install deps
npm install

# Run dev servers (from root)
npm run dev

# Run backend only
cd server && npm run dev

# Run frontend only
cd client && npm run dev

# Database migrations
cd server && npx prisma migrate dev

# Run tests
npm test

# Lint
npm run lint
```

## Coding Conventions

- Use TypeScript everywhere (strict mode)
- API routes follow REST conventions: `GET /swims`, `POST /swims`, `PATCH /swims/:id/complete`
- All times stored and transmitted as UTC ISO 8601 strings
- Database column names: snake_case. TypeScript variables: camelCase (Prisma handles mapping)
- Error responses always return `{ error: string, code: string }`
- Environment variables loaded via `.env` — never hardcode secrets

## Critical Business Logic

1. When a swim is created, schedule a job to fire at `startTime + estimatedDuration + gracePeriod (15 min)`
2. If the swimmer has not checked in by then, trigger the ICE notification
3. Once notified, do not send duplicate notifications — mark swim as `ALERT_SENT`
4. Users must be able to cancel a swim before it starts (prevents false alerts)
5. A completed or cancelled swim must NOT trigger notifications

## Swim State Machine

```
PLANNED → ACTIVE → COMPLETED
                 → OVERDUE → ALERT_SENT
PLANNED → CANCELLED
```

## Environment Variables Required

```
DATABASE_URL
CLERK_SECRET_KEY
TWILIO_ACCOUNT_SID
TWILIO_AUTH_TOKEN
TWILIO_FROM_NUMBER
SENDGRID_API_KEY
SENDGRID_FROM_EMAIL
```

## Testing Notes

- Use Jest + Supertest for backend API tests
- Use Vitest + React Testing Library for frontend
- Mock Twilio and SendGrid in tests — never send real notifications in test mode
- Seed script available: `npm run db:seed`
