# SwimSafe — Functional Specification

**Version:** 1.0  
**Date:** February 2026  
**Status:** Draft

---

## 1. Purpose

SwimSafe is a web application that provides a lightweight safety net for people doing open water swims (sea, lakes, rivers). The core premise: a swimmer tells the app they're going in, and if they don't tell it they're out, it alerts someone who can raise the alarm.

---

## 2. Users

**Primary user — the Swimmer**
A person who performs open water swims, either solo or in a group. They want peace of mind that someone will be alerted if something goes wrong, without needing a companion physically present.

**Secondary user — the ICE Contact (In Case of Emergency)**
A trusted person (friend, family member, coach) who receives alerts on behalf of the swimmer. They do not have a SwimSafe account — they are simply a named contact stored in the swimmer's profile.

---

## 3. Core User Flows

### 3.1 Account Setup

1. User visits SwimSafe and signs up with email and password (or Google OAuth)
2. During onboarding, user is prompted to add at least one ICE contact:
   - Name
   - Mobile phone number (for SMS)
   - Email address
3. User can add up to 3 ICE contacts. The first is primary; all are notified simultaneously.
4. User must complete onboarding before creating a swim.

### 3.2 Creating a Swim

1. User navigates to "Start a Swim" (prominent CTA on dashboard)
2. User fills in:
   - **Location** — free text or map pin (e.g. "Manly Beach, Sydney" or drop a pin)
   - **Estimated duration** — selected in 15-minute increments (min 15 min, max 8 hours)
   - **Notes (optional)** — e.g. "Doing a 2km loop around the buoys"
3. User confirms. The swim is created in `PLANNED` state and transitions to `ACTIVE` immediately.
4. A countdown is shown: "Alert will be sent in X hours Y minutes if you don't check in."

### 3.3 Completing a Swim (Check-In)

1. User returns to the app after their swim.
2. They tap "I'm safely back" on the active swim screen.
3. Swim transitions to `COMPLETED`. No alert is sent.
4. User sees a summary: swim duration, location, confirmation message.

### 3.4 Cancelling a Swim

1. User can cancel an active swim at any time before the alert fires.
2. Cancellation immediately stops any pending notification.
3. Swim transitions to `CANCELLED`.

### 3.5 Overdue Swim & Alert

1. At `startTime + estimatedDuration + 15 minute grace period`, the system checks if the swim is still `ACTIVE`.
2. If so, the system:
   - Sends an SMS to all ICE contacts: "⚠️ [Name] started a swim at [Location] at [time] and has not checked in. Please check on them. This is an automated alert from SwimSafe."
   - Sends the same message by email.
   - Transitions the swim to `OVERDUE`, then `ALERT_SENT`.
3. No further notifications are sent for this swim (no repeated alerts).
4. The swimmer's dashboard shows a banner: "An alert was sent to your contacts."

### 3.6 Viewing Swim History

1. User can view a log of past swims with date, location, duration, and status.
2. No editing of historical swims.

---

## 4. Features In Scope (v1.0)

| Feature | Description |
|---|---|
| Authentication | Email/password signup and login |
| ICE Contact Management | Add, edit, remove up to 3 contacts |
| Create Swim | Location, duration, optional notes |
| Active Swim Dashboard | Countdown, cancel, complete actions |
| Overdue Detection | Background job checks for overdue swims |
| SMS Alert | Twilio SMS to ICE contacts |
| Email Alert | SendGrid email to ICE contacts |
| Swim History | Read-only log of past swims |

---

## 5. Features Out of Scope (v1.0)

- Group swims (multiple swimmers on one session)
- Live GPS tracking
- Integration with wearables (e.g. Garmin, Apple Watch)
- ICE contact app / portal (contacts are passive recipients only)
- Recurring swim schedules
- Push notifications (mobile app)
- Swim route recording

---

## 6. Notifications — Message Templates

**SMS (sent to each ICE contact):**
```
⚠️ SWIMSAFE ALERT
[FirstName] [LastName] started an open water swim at [Location] at [StartTime] 
and has not checked in. Their estimated duration was [Duration].

Please try to contact them or raise the alarm if needed.

This is an automated safety alert from SwimSafe.
```

**Email subject:** `⚠️ SwimSafe Safety Alert — [FirstName] has not checked in`

**Email body:** Same content as SMS, plus a link to SwimSafe and a note that the swimmer can still check in to cancel further concern.

---

## 7. Non-Functional Requirements

- **Reliability:** Alert jobs must be fault-tolerant. If a job fails, it should retry up to 3 times before logging a failure.
- **Latency:** Alert should fire within 2 minutes of the overdue threshold.
- **Mobile-first UI:** The app is primarily used on mobile (swimmer pulls phone out of dry bag). All core actions must be one-thumb accessible.
- **Accessibility:** WCAG 2.1 AA compliance.
- **Data retention:** Swim history retained for 2 years, then deleted. ICE contact data retained until account deletion.

---

## 8. Edge Cases to Handle

| Scenario | Behaviour |
|---|---|
| User creates swim but loses internet | Swim remains ACTIVE; alert fires based on server-side timer |
| User has no ICE contacts set | Cannot create a swim — prompted to add a contact first |
| ICE contact number is invalid | Twilio error is logged; email is still attempted; failure recorded on swim |
| User checks in after alert already sent | Swim moves to COMPLETED; no further action; alert already sent cannot be recalled |
| User creates two swims simultaneously | UI prevents this — only one active swim permitted at a time |

---

## 9. Success Metrics (v1.0)

- Zero missed alerts for genuinely overdue swims (alert reliability)
- Less than 5% false alert rate (user checks in but alert still fires — should not happen)
- Core user flow completable in under 60 seconds on mobile
