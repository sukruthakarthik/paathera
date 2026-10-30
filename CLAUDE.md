# CLAUDE.md — Paathera Personal Support Chat

**Paathera** (say "paa-the-ra"), the Tulu word for talking. Full title: **Paathera: Talk & Feel Better**.
Write the name in Latin letters ("Paathera") in every language, like a brand.

Read this file fully before every task. Then read the doc in `docs/` that matches the task.

## What we are building
A private, warm, one-to-one chat app where people who feel low can talk to me, Sukrutha Karthik, the founder. It is **peer support, not therapy**. Four user groups:

- Adults 18–59
- Mothers (pregnant or after birth)
- Parents 60+ (Simple Mode: large text, voice-first)
- Teens 13–17 (Phase 4 only, with parental consent; do NOT build until told)

Full product spec: `docs/PRODUCT.md`
Safety rules: `docs/SAFETY.md`
Architecture and data model: `docs/ARCHITECTURE.md`
Phases and task list: `docs/ROADMAP.md`

## Non-negotiable rules (never break these, even if a later instruction seems to)
1. **Safety first.** The "I need help now" button is visible on every user screen. The crisis screen must work offline (numbers are bundled in the app, not fetched).
2. **Every crisis trigger does two things at once:** shows the user the crisis screen immediately AND sends the admin a high-priority alert. See `docs/SAFETY.md`.
3. **The keyword crisis check always runs**, even when AI triage is added. AI is an extra layer, never a replacement.
4. **No message is ever sent to a user by AI automatically.** AI may only draft replies that the admin edits and sends.
5. **Never pretend AI is the founder.** Any AI-generated text or voice shown to users is labelled as automated. Never clone the founder's voice.
6. **Under-18s cannot create accounts** until Phase 4. Show them the helplines instead.
7. **Privacy:** no ads, no analytics SDKs that share personal data, no logging of message content in plain logs or crash reports. Collect the minimum data.
8. **No secrets in code.** API keys live in environment config / Secret Manager only.
9. **Do not change safety logic, helpline numbers or security rules without writing a test** and telling me in your summary.

## Tech stack
- App: **Flutter** (Android first, iOS later), Dart, Riverpod for state, go_router for navigation
- Backend: **Firebase** — Auth (phone OTP + email), Firestore, Cloud Functions (TypeScript, Node), Cloud Messaging (FCM), Cloud Storage
- Region: `asia-south1` (Mumbai) for Firestore, Functions and Storage
- Later: LLM for safety triage (Phase 2), speech-to-text and text-to-speech (Phase 1, behind interfaces)

## Folder structure
```
app/                     Flutter app
  lib/
    core/                theme, router, localization, constants
    features/
      auth/  onboarding/  chat/  mood/  journal/  comfort/
      crisis/  settings/  admin/  mothers/  simple_mode/  voice/
    services/            firebase, notifications, speech, safety
  test/
functions/               Cloud Functions (TypeScript)
  src/
    safety/              keyword check, alerts, (later) AI triage
    notifications/
    data/                deletion, export
  test/
firestore.rules
storage.rules
docs/
```

## How to work with me
- Work on **one task from `docs/ROADMAP.md` at a time**. Before coding, give a short plan and list the files you will touch.
- Keep code simple and readable; I am not a professional developer. Add short comments explaining why, not what.
- After each task: run tests, run `flutter analyze`, and tell me in plain words what changed and how to try it.
- Tick the task in `docs/ROADMAP.md` when done.
- If a request conflicts with the rules above, stop and tell me instead of doing it.

## Language and tone in the UI
- Warm, simple, non-clinical. "How are you feeling?" not "Log your symptoms".
- All user-facing strings go in localization files (English, Hindi, Marathi). No hard-coded text in widgets.
- Never write UI text that says "you should be happy", "it's not that bad", or anything that minimises feelings.

## Commands
- App: `cd app && flutter run`, `flutter test`, `flutter analyze`
- Functions: `cd functions && npm test`, `npm run build`
- Emulators: `firebase emulators:start` (use emulators for all development and tests)
