# StudyNova AI — Architecture

## 1. Layering (Clean Architecture, feature-first)

Every feature in `lib/features/<feature>/` has three layers:

```
feature/
  data/
    models/          # Firestore-serializable DTOs (fromJson/toJson, fromDoc)
    repositories/     # Concrete repo — talks to Firestore/Gemini/Storage
  domain/
    entities/          # Pure Dart business objects, no Firebase imports
    providers/          # Abstract repo interfaces (Riverpod providers of contracts)
  presentation/
    screens/
    widgets/
    providers/          # Riverpod StateNotifiers / AsyncNotifiers driving the screen
```

Rule: `presentation` never imports Firestore directly — it calls a `domain` provider,
which is bound to a `data` repository implementation. This keeps every screen
testable and keeps Firebase swappable.

`core/` holds cross-cutting code with no feature knowledge: theme, router, DI
wiring, error types, generic services (GeminiService, NotificationService).
`shared/` holds models/widgets used by 2+ features (e.g. `UserModel`, `GlassCard`).

## 2. Folder Structure

```
lib/
  core/
    config/            # env keys, firebase_options, app_config
    theme/              # ThemeData, colors, typography, spacing tokens
    router/              # go_router config + route guards
    constants/          # firestore collection names, asset paths, PCTB subject/chapter maps
    errors/              # Failure classes, exception -> failure mapping
    services/            # GeminiService, NotificationService, ConnectivityService
    widgets/              # GlassCard, GradientScaffold, PrimaryButton, ShimmerLoader
  shared/
    models/              # UserModel, SubjectModel, ChapterModel
    widgets/
  features/
    auth/                 # splash + google sign-in
    onboarding/           # name/class/group capture (first login only)
    home/                  # dashboard
    planner/                # study planner (daily/weekly/revision)
    ai_tutor/                # chat interface + Gemini
    scan_study/              # camera capture -> Gemini vision -> notes
    formula_book/             # subject -> formula list, search, bookmark
    mock_tests/                # AI MCQ generation, timer, scoring
    analytics/                  # charts, weakness detection
    gamification/                # streak/XP/badges (mostly logic + shared widgets)
    profile/                      # profile, settings, logout
  main.dart
  app.dart                        # MaterialApp.router + ProviderScope wiring
```

## 3. State Management — Riverpod

- **AsyncNotifierProvider** for anything that loads from Firestore/Gemini
  (chat sessions, planner, mock test generation, analytics aggregates).
- **NotifierProvider** for local UI state (selected subject/class filter, timer state).
- **StreamProvider** for real-time Firestore listeners (`chat_history`, `study_plans`
  active plan, `achievements`).
- **Provider** (plain) for repository singletons and derived/computed values
  (e.g. `weakSubjectsProvider` derived from `analyticsProvider`).

`authStateProvider` (StreamProvider<User?> wrapping `FirebaseAuth.authStateChanges()`)
is the single source of truth GoRouter's `redirect` reads from. A second
`userProfileProvider` (StreamProvider on `users/{uid}`) tells the router whether
onboarding is complete (`onboardingComplete: bool` field).

## 4. Navigation — GoRouter

```
/splash
/login
/onboarding                 (name -> class -> group, 3-step PageView)
  ShellRoute (StatefulShellRoute.indexedStack) -> bottom nav, 5 branches:
    /home
    /planner
    /ai-tutor
    /analytics
    /profile
  Full-screen (pushed, no bottom nav):
    /scan-study
    /formula-book
    /formula-book/:subjectId
    /mock-tests
    /mock-tests/session/:testId
    /mock-tests/result/:resultId
    /ai-tutor/chat/:sessionId
    /profile/settings
    /profile/achievements
```

Redirect logic (in `core/router/app_router.dart`):
1. `authState == null` and route isn't `/login` or `/splash` → redirect `/login`.
2. `authState != null` and `userProfile.onboardingComplete == false` → redirect
   `/onboarding` (blocks everything else until finished).
3. `authState != null`, onboarding complete, on `/login` or `/onboarding` →
   redirect `/home`.

`StatefulShellRoute.indexedStack` is used (not plain nested routes) so each
bottom-nav tab keeps its own navigation stack and scroll position.

## 5. Firestore Schema

```
users/{uid}
  name, email, photoUrl
  studentClass: "9" | "10"
  group: "science" | "arts"
  board: "PCTB"
  onboardingComplete: bool
  xp: number, level: number, streakCount: number, lastStudyDate: Timestamp
  createdAt, updatedAt

study_plans/{uid}/plans/{planId}
  examDate: Timestamp
  dailyHours: number
  weakSubjects: [subjectId]
  dailyTasks: [{ date, subjectId, chapterId, taskType, done, autoRescheduled }]
  status: "active" | "completed"

chat_history/{uid}/sessions/{sessionId}
  title, createdAt, updatedAt
  messages: subcollection messages/{messageId}
    { role: "user"|"model", text, imageUrl?, createdAt }

scan_history/{uid}/scans/{scanId}
  imageUrl, subjectId, chapterId?, extractedText
  summary, importantPoints[], mcqs[], shortQuestions[], revisionNotes
  createdAt

mock_tests/{uid}/tests/{testId}
  subjectId, chapterId, difficulty, questions: [{q, options[4], correctIndex, explanation}]
  timeLimitMin, createdAt

mock_tests/{uid}/results/{resultId}
  testId, subjectId, chapterId, score, total, timeTakenSec
  answers: [{questionIndex, selectedIndex, correct}]
  createdAt

analytics/{uid}/daily/{yyyy-mm-dd}
  studyMinutes, subjectsTouched[], tasksCompleted, tasksPlanned

achievements/{uid}/unlocked/{badgeId}
  badgeId, unlockedAt

bookmarks/{uid}/formulas/{formulaId}
  subjectId, formulaId, savedAt

content/subjects/{class}_{subjectId}/chapters/{chapterId}
  (shared, read-only reference content — PCTB syllabus, seeded server-side,
   not per-user; formula_book and scan_study reference this for chapter names)
```

Class 9 vs 10 separation is enforced by keying content docs as `9_math`,
`10_math`, etc., and by always filtering user-generated collections
(`study_plans`, `mock_tests`, ...) by the `studentClass` on the user's own
profile — a student never queries across classes.

Security rules (summary): every top-level collection under a `{uid}` path is
readable/writable only by that `uid`; `content/*` is read-only for all
authenticated users and write-only via the Firebase console / admin SDK.

## 6. Gemini AI Integration

`core/services/gemini_service.dart` wraps `google_generative_ai`:
- `explainConcept(prompt)`, `solveQuestion(prompt, {imageBytes})`,
  `generateMcqs(chapter, count, difficulty)`, `summarizeChapter(text)`,
  `analyzeScannedPage(imageBytes)` → all return typed results parsed from a
  strict JSON response schema (model is instructed to reply JSON-only).
- Called only from `data/repositories/*_repository.dart` in each feature
  (ai_tutor, scan_study, mock_tests) — never directly from a widget.
- API key loaded from `--dart-define=GEMINI_API_KEY` (not hardcoded).

## 7. Design System (`core/theme/`)

- `app_colors.dart`: `background: #0A0A0F`, `surface: #14141C`, glass surface
  `rgba(255,255,255,0.06)` with 1px `rgba(255,255,255,0.12)` border,
  `primaryPurple: #7C5CFF`, `accentCyan: #37E6E6`, gradient
  `[#7C5CFF → #37E6E6]` for primary CTAs/progress rings.
- `app_typography.dart`: Satoshi via `google_fonts` fallback, display/headline/
  body/label scale per Material 3 type roles.
- `app_radii.dart` / `app_spacing.dart`: 24px card radius, 4/8/12/16/24/32 spacing scale.
- `glass_card.dart`: reusable `BackdropFilter(blur: 20)` + gradient border card
  used across dashboard, planner, formula book, chat bubbles.
- Dark `ThemeData` only (no light theme) — matches the "premium AI product" brief.

## 8. Build Order (this response and follow-ups)

1. ✅ Folder structure, `pubspec.yaml`, this architecture doc.
2. Core: theme, constants, error types, GoRouter skeleton, `main.dart`/`app.dart`.
3. Shared models (`UserModel`) + `core/services` (Gemini, Firebase bootstrap).
4. Auth + Onboarding (splash → Google sign-in → 3-step onboarding → Firestore write).
5. Home dashboard + bottom nav shell.
6. Study Planner.
7. AI Tutor (chat UI + Gemini).
8. Scan & Study.
9. Formula Book.
10. Mock Tests + scoring.
11. Analytics + weakness detection.
12. Gamification (streak/XP/badges) + Profile/Settings.
13. Firestore security rules + `firebase_options.dart` placeholder + README (setup steps).

Each step lands as real, compilable Dart files (not pseudocode), wired into
the router and providers already built in earlier steps.
