# StudyNova AI

Your AI Study Partner for PCTB Matric Students — a premium, glassmorphic,
AI-powered study app for Punjab Curriculum and Textbook Board Class 9 & 10
students, built with Flutter + Firebase + Gemini.

See `ARCHITECTURE.md` for the full design (layering, folder structure,
Riverpod strategy, navigation map, Firestore schema, Gemini contract, design
system, build order).

## 1. Prerequisites

- Flutter (latest stable) — `flutter --version`
- A Firebase project (console.firebase.google.com)
- A Google AI Studio / Gemini API key (aistudio.google.com/app/apikey)
- `flutterfire_cli`: `dart pub global activate flutterfire_cli`

## 2. Install dependencies

```bash
flutter pub get
```

## 3. Firebase setup

1. Create a Firebase project.
2. Enable **Authentication → Google** sign-in provider.
3. Enable **Cloud Firestore** (start in production mode — the rules below
   lock it down).
4. Enable **Cloud Storage**.
5. Enable **Cloud Messaging** (for study reminders / push notifications).
6. From the project root, run:

   ```bash
   flutterfire configure
   ```

   This regenerates `lib/firebase_options.dart` with your real project
   credentials and wires up `android/` and `ios/` — it **replaces** the
   placeholder file shipped in this repo.

7. Deploy the security rules:

   ```bash
   firebase deploy --only firestore:rules,storage:rules
   ```

   (`firestore.rules` and `storage.rules` are already in the project root.)

## 4. Gemini API key

Never hardcode the key. Run with:

```bash
flutter run --dart-define=GEMINI_API_KEY=your_key_here
```

Or add it to a `--dart-define-from-file=env.json` (gitignored):

```json
{ "GEMINI_API_KEY": "your_key_here" }
```

```bash
flutter run --dart-define-from-file=env.json
```

## 5. Seeding reference content

`content/subjects_{class}_{subjectId}/chapters/*` (Formula Book chapter
names) and `content/formulas_{subjectId}/items/*` (the formulas themselves)
are shared, read-only collections — they're not created by the app. Seed
them once via the Firebase console or a small Admin SDK script; see the
`FormulaModel` shape in
`lib/features/formula_book/data/models/formula_model.dart` for the exact
fields each formula document needs (`subjectId`, `chapterName`, `title`,
`formula`, `explanation`, `example`).

## 6. Run

```bash
flutter run --dart-define=GEMINI_API_KEY=your_key_here
```

## 7. Fonts

`pubspec.yaml` references a bundled `Satoshi` font family under
`assets/fonts/`. Add the four weights listed there (or delete that `fonts:`
block) — the app falls back to Google Fonts' `Manrope` either way, so it
runs fine without them.

## Project status

Built module-by-module per `ARCHITECTURE.md`'s build order. All 11 features
(Auth, Onboarding, Home, Planner, AI Tutor, Scan & Study, Formula Book, Mock
Tests, Analytics, Gamification, Profile) have real repositories, Riverpod
providers, and screens wired through GoRouter — no placeholder screens.

Not included (intentionally, as next steps for a real ship):
- Android/iOS native project folders (`flutterfire configure` + `flutter
  create .` scaffolds these)
- FCM push notification payload handling / local notification scheduling
  logic (the dependency is included; wiring the actual triggers is a
  product decision — e.g. what time to remind, which missed-task threshold)
- Automated tests
- CI/CD
