# SwimSafe — Architecture Document

**Version:** 1.0  
**Date:** February 2026

---

## 1. Architecture Overview

SwimSafe is a standard three-tier web application: a React single-page application, a Node/Express REST API, and a PostgreSQL database. A background job scheduler handles the time-sensitive alert logic.

```
┌─────────────────────────────────────────────────────┐
│                    Client (Browser)                 │
│                React SPA (Vite + Tailwind)          │
└──────────────────────┬──────────────────────────────┘
                       │ HTTPS / REST
┌──────────────────────▼──────────────────────────────┐
│               Express API Server                    │
│   ┌─────────────┐   ┌────────────┐  ┌───────────┐  │
│   │   Routes    │   │  Services  │  │  Jobs     │  │
│   │  /auth      │   │  SwimSvc   │  │  Overdue  │  │
│   │  /swims     │   │  AlertSvc  │  │  Checker  │  │
│   │  /contacts  │   │            │  │  (cron)   │  │
│   └─────────────┘   └────────────┘  └───────────┘  │
└────────┬─────────────────────────────────┬──────────┘
         │                                 │
┌────────▼──────────┐          ┌───────────▼──────────┐
│   PostgreSQL DB   │          │  External Services   │
│   (via Prisma)    │          │  - Twilio (SMS)      │
│                   │          │  - SendGrid (Email)  │
└───────────────────┘          │  - Clerk (Auth)      │
                               └──────────────────────┘
```

---

## 2. Frontend Architecture

**Framework:** React 18 with Vite  
**Styling:** TailwindCSS  
**State management:** React Query (server state) + Zustand (minimal local state)  
**Routing:** React Router v6

### Key Pages

| Route | Component | Description |
|---|---|---|
| `/` | `HomePage` | Marketing / login prompt |
| `/dashboard` | `DashboardPage` | Active swim or "Start a Swim" CTA |
| `/swim/new` | `NewSwimPage` | Create swim form |
| `/swim/:id` | `ActiveSwimPage` | Countdown, complete, cancel |
| `/history` | `HistoryPage` | Past swim log |
| `/settings` | `SettingsPage` | ICE contacts, account |
| `/onboarding` | `OnboardingPage` | First-time setup wizard |

### API Client

All API calls go through a centralised `src/lib/api.ts` module that:
- Attaches the Clerk JWT to every request
- Standardises error handling
- Uses React Query for caching and background refetching

---

## 3. Backend Architecture

**Runtime:** Node.js 20 LTS  
**Framework:** Express  
**Language:** TypeScript (strict)  
**ORM:** Prisma  
**Job Scheduler:** node-cron (runs within the same process for v1)

### Route Structure

```
POST   /api/swims              — Create a new swim
GET    /api/swims              — List user's swims (paginated)
GET    /api/swims/active       — Get the current active swim
PATCH  /api/swims/:id/complete — Mark swim as completed
PATCH  /api/swims/:id/cancel   — Cancel a swim
GET    /api/contacts           — List ICE contacts
POST   /api/contacts           — Add ICE contact
PUT    /api/contacts/:id       — Update ICE contact
DELETE /api/contacts/:id       — Remove ICE contact
```

### Service Layer

**SwimService** — core swim lifecycle logic
- `createSwim(userId, data)` — validates, persists, returns swim
- `completeSwim(swimId, userId)` — transitions to COMPLETED
- `cancelSwim(swimId, userId)` — transitions to CANCELLED
- `getOverdueSwims()` — queries DB for ACTIVE swims past their alert threshold

**AlertService** — notification dispatch
- `sendAlert(swim)` — calls Twilio and SendGrid in parallel
- `formatSmsMessage(swim, contact)` — builds SMS string
- `formatEmailMessage(swim, contact)` — builds email HTML

### Background Job

A cron job runs every **60 seconds** and:
1. Calls `SwimService.getOverdueSwims()`
2. For each overdue swim, calls `AlertService.sendAlert(swim)`
3. Updates swim status to `ALERT_SENT`
4. Logs success or failure

The 60-second poll interval means alerts may fire up to 1 minute late, which is acceptable given the 15-minute grace period already built into the threshold.

---

## 4. Database Schema

```prisma
model User {
  id           String    @id @default(cuid())
  clerkId      String    @unique
  email        String    @unique
  firstName    String
  lastName     String
  createdAt    DateTime  @default(now())
  swims        Swim[]
  iceContacts  IceContact[]
}

model IceContact {
  id        String   @id @default(cuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  name      String
  phone     String
  email     String
  priority  Int      @default(1)   // 1 = primary
  createdAt DateTime @default(now())
}

model Swim {
  id                String     @id @default(cuid())
  userId            String
  user              User       @relation(fields: [userId], references: [id])
  location          String
  locationLat       Float?
  locationLng       Float?
  notes             String?
  estimatedDuration Int        // minutes
  startTime         DateTime
  alertThreshold    DateTime   // startTime + estimatedDuration + 15 min
  completedAt       DateTime?
  cancelledAt       DateTime?
  alertSentAt       DateTime?
  status            SwimStatus @default(ACTIVE)
  createdAt         DateTime   @default(now())
}

enum SwimStatus {
  ACTIVE
  COMPLETED
  CANCELLED
  OVERDUE
  ALERT_SENT
}
```

---

## 5. Authentication

Authentication is handled entirely by **Clerk**. The backend verifies JWTs on every request using the Clerk SDK. There is no custom auth code.

User records in the database are keyed on `clerkId`. A user record is created on first login via a Clerk webhook.

---

## 6. Notification Architecture

Both notification channels are called in parallel using `Promise.allSettled()` so that a failure in one does not prevent the other from being attempted. Failures are logged and recorded on the swim record but do not cause the job to crash or retry that swim.

```
AlertService.sendAlert(swim)
  └── Promise.allSettled([
        TwilioService.sendSms(contacts, message),
        SendGridService.sendEmail(contacts, subject, body)
      ])
  └── Log results, update swim.alertSentAt
```

---

## 7. Deployment

| Component | Platform | Notes |
|---|---|---|
| Frontend | Vercel | Auto-deploys from `main` branch |
| Backend + Jobs | Railway | Single service, always-on |
| Database | Railway PostgreSQL | Managed, daily backups |
| Auth | Clerk (hosted) | No self-hosting needed |
| SMS | Twilio | Pay per message |
| Email | SendGrid | Free tier sufficient for v1 |

### Environment Promotion

- `main` branch deploys to **production**
- `develop` branch deploys to **staging** (separate Railway environment)
- Feature branches are previewed via Vercel preview URLs (frontend only)

---

## 8. Key Technical Decisions

**Why not a message queue for alerts?**  
A proper queue (e.g. BullMQ + Redis) would be more robust but adds infrastructure complexity. For v1, a 60-second cron poll is simple, predictable, and easy to debug. The only failure mode — a process crash between poll intervals — is acceptable given the 15-minute grace period. This decision should be revisited if the user base grows significantly.

**Why Clerk for auth?**  
Building secure auth from scratch is high risk and low value for v1. Clerk handles OAuth, password reset, session management, and JWT validation. The cost is acceptable at early scale.

**Why Railway over AWS/GCP?**  
Simplicity. Railway provides a Postgres database and Node.js hosting with minimal configuration. This is appropriate for an MVP and can be migrated later if needed.

**Why Prisma over raw SQL?**  
Type-safe queries and auto-generated migrations reduce bugs and speed up development. The query patterns in this app are simple enough that Prisma's limitations (e.g. complex joins) are unlikely to be hit.
