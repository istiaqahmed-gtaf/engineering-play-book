# Sadiq Engineering Playbook

A guide for getting from "I can make the feature work" to "my MRs pass review the
first time" on this codebase specifically. It is written against what this repo
actually does, not against generic Flutter advice.

Read `.claude/skills/mr-review/references/review_checklist.md` and
`flutter_specific.md` alongside this — those are the filters a senior applies.
This document is the *why* and the *where* for this project.

---

## 1. The shape of the app

Know this before you touch anything. Most review comments are not "you wrote bad
Dart", they are "you did not know this app already has a rule for that".

### 1.1 Layering

Every feature folder under `lib/<feature>/` follows the same four layers:

```
lib/<feature>/
  data/      repositories, local services, remote services, models w/ json
  domain/    riverpod providers + notifiers  (the only place state lives)
  ui/        full pages/screens
  widget/    composable pieces used by ui/
  util/      pure functions, mappers, constants
  service/   non-riverpod workers (image generators, etc.)
```

Rule: **data never imports domain, domain never imports ui.** If you find
yourself wanting a `BuildContext` in `data/` or `domain/`, the design is wrong —
pass the value down instead.

Reference implementations worth reading end-to-end before your next feature:

- `lib/sync/` — smallest complete vertical slice (model → local → remote → repo → provider).
- `lib/content/` — hardest one: downloads, progress, db swap, failure paths.
- `lib/quran/` — biggest one; shows how a feature gets subdivided when it grows.

### 1.2 The spine

These files are touched by nearly every feature. Read them once, properly, and a
large fraction of "how do I…" questions answer themselves.

| File | What it owns |
|---|---|
| `lib/main.dart` | Boot order. ~1200 lines. Everything init'd before `runApp` lives here. |
| `lib/common/domain/app_config_provider.dart` | `keepAlive` god-provider: locale, theme, location, calc method, auth. |
| `lib/common/data/app_config_state.dart` | Immutable app state + sentinel `copyWith`. |
| `lib/common/data/base_repository.dart` | sqflite asset-db copy / version / corruption recovery. |
| `lib/common/data/result.dart` | `Success` / `Failure` / `FieldFailure` / `CancelledFailure`. |
| `lib/common/data/basic_state.dart` | `UiState` enum: initial/loading/successful/error/empty/cancelled. |
| `lib/common/data/extensions.dart` | ~29 extensions. Check here *before* writing a helper. |
| `lib/common/data/provider_retry.dart` | Why Riverpod 3 auto-retry is disabled app-wide. |
| `lib/core/network/network_client.dart` | Dio setup, token refresh hook. |
| `lib/navigation/ui/routes.dart` | Route table; deep links resolve through here. |

**`extensions.dart` is the single highest-value file to read.** Most duplicated-
logic review comments are "we already have this extension."

### 1.3 Two locales, not one

`AppConfigState` has **`locale`** (UI chrome language) *and* **`contentLocal`**
(the language of Quran/dua/hadith content). They are set independently and they
diverge for real users.

Git history shows at least five separate bugs from conflating them:
`Fixed bug due to locale country code`, `Fixed device locale issue`,
`Fixed issue with mixup of content language`, `Using ui locale for content db,
and fixed mismatch on db checking for content update`.

Before every locale read, ask: *am I labelling a button, or am I fetching
scripture?* Button → `locale`. Scripture → `contentLocal`.

Also: locales here carry **country codes** (`useOnlyLangCode: false` in
`main.dart`). `Locale('bn')` and `Locale('bn', 'BD')` are not equal. Never
compare locales with `==` unless you are certain both sides are normalised —
use the helpers in `extension LocaleX` (`extensions.dart:69`).

---

## 2. The bug classes this project actually produces

Derived from the repo's own fix commits. These are what your seniors are
catching. Each one has a rule you can apply mechanically.

### 2.1 Stuck loading states

> `Fix Quran reading views stuck loading when word-by-word is unavailable`
> `Fixed issue of goal list not loading`
> `Handled quran data load failure`

The pattern: optional data is missing (translation not downloaded, WBW db
absent, network down), the provider never resolves, spinner spins forever.

**Rule:** every async surface needs four branches, not two — `loading`,
`data`, `error`, and **`data-but-empty`**. That is exactly why `UiState` has both
`error` and `empty`. If your `.when()` has no error branch, that is a P1 in
review.

**Rule:** Riverpod 3's retry is disabled on purpose
(`provider_retry.dart`). Errors surface *immediately*. So an unhandled error
branch is visibly broken, not eventually-fine.

### 2.2 `BuildContext` across an async gap

> `Fixed async gap issue`, `Fixed issue of showing snackbar indefinitely`

Note `use_build_context_synchronously: false` in `analysis_options.yaml` — the
analyzer will **not** catch this for you here. It is on you.

```dart
// wrong
await repo.save();
ScaffoldMessenger.of(context).showSnackBar(...);

// right
await repo.save();
if (!context.mounted) return;
ScaffoldMessenger.of(context).showSnackBar(...);
```

Same for `Navigator.pop`, `showDialog`, `Theme.of`, `ref.read` after an await in
a widget callback.

### 2.3 Lifecycle: things that outlive their widget

> `Fixed issue for audio kept playing in the background on app config change`
> `Added explicit scroll controller in scrollbar to fix error`

Config changes (locale switch, theme switch) rebuild the tree. Anything holding
a resource — `AudioPlayer`, `AnimationController`, `ScrollController`,
`PageController`, `TextEditingController`, `FocusNode`, `Timer`,
`StreamSubscription`, `WidgetsBindingObserver` — must be disposed.

**Rule:** for every `X()` you construct in `initState`, write the `X.dispose()`
line in `dispose()` *in the same edit*, before you write any other code. For
every `.listen(` write the `.cancel()`. For every `addListener` write the
`removeListener`.

**Rule:** for `keepAlive` providers holding resources, use `ref.onDispose`.

### 2.4 Concurrency

> `Fixed random crash on multiple translation download due to concurrency`

Downloads, db writes, and sync can all be triggered twice — user double-taps,
background fetch fires while the UI is open, a retry overlaps the original.

**Rule:** any operation that mutates a file or a db table needs a guard: an
in-flight flag (see `_isHandlingTokenExpiry` in `app_config_provider.dart`), a
`Completer` that later callers await, or a queue. "It won't be called twice" is
not true on this app.

### 2.5 Date, time, and calendar edge cases

> `Fixed issue of calendar not showing days from the week, e,g 31st Aug 2026`
> `Fixed prayer time window calculation in iOS for Asr and Isha`
> `Fix Jakim angle, Add some custom prayer method`

This app is a *date and time app*. Prayer times, Hijri calendar, fasting days,
notification scheduling. Hard cases are the default case here.

**Rule:** test every date change against: month boundaries, the 29/30/31 split,
DST transitions, the Hijri/Gregorian offset (`hijri_adjustment.dart`), high
latitudes (`HighLatitudeRule`), and timezone changes while the app is open.
Never use `DateTime.now()` inside logic you want to test — take it as a
parameter.

### 2.6 Release-only failures

> `Fixed build issue due R8 pro-guard rule`
> `Fixed crash on release build due to firebase native SO being obfuscated`
> `Fixed android prod build failure`

Debug builds are not evidence. R8 strips and renames; anything reached by
reflection or JNI (Firebase native libs, some plugins) needs a keep rule in
`android/app/proguard-rules.pro`.

**Rule:** if your MR adds a plugin with native code, build a release APK before
requesting review. This is the single cheapest way to stop a class of P1s.

### 2.7 Migration / existing-user paths

> `Fixed issue of showing onboarding for updating users`
> `lib/quran/util/translation_db_migration.dart`

Every change to stored data has two audiences: a fresh install and a user
upgrading from 1.18 with old data on disk.

**Rule:** when you add a persisted field, answer in the MR description: what
does an upgrading user with no value for this field see? Default value? Migration?
`BaseRepository.databaseVersion` bump? (Note the `FIXME` at
`base_repository.dart:35` — db versioning is known-unreliable; do not assume it
saved you.)

### 2.8 Layout under real content

> `Fixed issue of hadith card not taking full width`
> `Fixed bottomsheet height issue on translation search`
> `Fixed dua ayah being on same line`

**Rule:** check every new screen at: smallest supported width, largest text
scale, dark mode, Arabic/Urdu (RTL + tall scripts: `NotoNastaliqUrdu` needs more
line height than Latin), and with the longest real string from the arb files,
not "Test".

---

## 3. Pre-MR self-review

Do this before you push. It is ~15 minutes and it is where the improvement will
actually come from.

**Read your own diff, alone, as if someone else wrote it.**
`git diff development...HEAD` in a pager. Not the IDE. You will find things.

Then, line by line:

1. **Every `await`** — is a `BuildContext` used after it? Is `mounted` checked?
2. **Every `!` and every `late`** — can the source be null in reality (API, JSON, nav args, db row)? Prefer `?.` / `firstWhereOrNull`.
3. **Every `.first` / `.last` / `.firstWhere`** — what happens on an empty list?
4. **Every constructed controller / listener / subscription / timer** — disposed?
5. **Every new async surface** — loading, error, *and* empty branch present?
6. **Every user-visible string** — in an arb file, not a literal?
7. **Every locale read** — `locale` or `contentLocal`? Deliberate?
8. **Every colour / text style** — from `Theme` / `dimens.dart`, not hardcoded?
9. **Every widget in a list** — `const` where possible, `ListView.builder` not `ListView(children:)`?
10. **Every new helper** — does `extensions.dart` or `util.dart` already have it?
11. **`ref.watch` vs `ref.read`** — `watch` in `build`, `read` in callbacks. Never the reverse.
12. **Debug leftovers** — `print`, commented-out code, `// ignore:` you added.
13. **Scope** — is anything in the diff not required by the ticket? Split it out.
14. `fvm flutter analyze` clean. Then run the app and use the feature.

Then write the MR description with: what changed, why, how you tested it, and
what you deliberately did not do. A reviewer who can see your reasoning stops
guessing and starts reviewing.

**When a reviewer catches something, log it.** Keep a file — one line per
finding, categorised. After twenty entries you will see that (for example) 60%
of yours are disposal or async-gap. That is a much sharper target than "get
better at Flutter."

---

## 4. Learning path

Ordered by payoff-per-hour **for this codebase**. Do them in order.

### Tier 1 — the next two weeks

**1. Dart async semantics, properly.**
Not "how to use await" — the model. Microtask vs event queue, why `Future`s are
not cancellable, what an unawaited future does with its error, `Completer`,
`Stream` vs `Future`, broadcast vs single-subscription streams.
Note `unawaited_futures: false` in `analysis_options.yaml` — 206 existing
violations, and the analyzer will not warn you on new ones.
→ Dart docs "Asynchronous programming"; then the `dart:async` source for `Completer`.

**2. Riverpod 3 — lifecycle, not just syntax.**
`ref.watch` vs `ref.read` vs `ref.listen`. When a provider is disposed and
rebuilt. `keepAlive` vs autoDispose. `ref.invalidate`. Family providers and why
a new family argument is a new provider. `AsyncValue` and all its states
(including `isRefreshing`, `isReloading`).
Then read `lib/common/domain/app_config_provider.dart` top to bottom and trace
what happens when the user changes language.
→ riverpod.dev docs, "Concepts" section in full.

**3. Flutter widget lifecycle and rebuild mechanics.**
`initState` / `didChangeDependencies` / `didUpdateWidget` / `dispose` — what runs
when. Element vs Widget vs RenderObject. Why `const` matters. Keys.
→ Flutter docs "Inside Flutter"; the `Element` class docs.

**4. This project's spine.**
The table in §1.2. Read all ten files. Take notes. You are allowed to not
understand `main.dart` on the first pass.

### Tier 2 — the following month

**5. SQLite and the drift migration.**
`lib/common/data/drift/` is new scaffolding; `base_repository.dart` is the old
sqflite path. Both are live. Understand: indexes, query plans, transactions,
why a db copy from assets can fail, what a corrupted db looks like.
→ SQLite docs on `EXPLAIN QUERY PLAN`; drift.simonbinder.eu "Migrations".

**6. i18n done right.**
ICU message format, plural categories (Arabic has six), RTL layout, why
`if (n == 1)` is a bug. This project uses `easy_localization` + arb files + a
custom `ArbAssetLoader` + runtime Weblate updates
(`lib/localization/util/weblate_translation_service.dart`) — read all of
`lib/localization/`.
→ Unicode CLDR plural rules; Flutter i18n docs.

**7. Testing, at the level this repo uses it.**
`test/widget/` for widget tests, `integration_test/` with Patrol. Learn
`mocktail`, `ProviderScope(overrides:)`, `pump` vs `pumpAndSettle`, and how to
fake time and network.
Start by writing a regression test for the next bug you are asked to fix. That
is the habit that most changes how seniors see your MRs.

**8. Android/iOS release builds.**
R8/ProGuard keep rules, `--obfuscate --split-debug-info`, native symbol upload
to Crashlytics, Info.plist usage descriptions, AndroidManifest permissions.
→ Flutter "Build and release" docs; Android R8 docs.

### Tier 3 — the quarter

**9. Performance profiling.** DevTools timeline, rebuild counter, raster vs UI
thread jank, `RepaintBoundary`, image `cacheWidth`. Profile mode only.

**10. Designing for failure.** Retries with backoff, idempotency, partial
downloads, what the UI does at 2G. `lib/content/` is the case study.

**11. Reading code you did not write.** Pick a feature you have never touched —
`lib/app_widget/` or `lib/goals/` — and write a one-page explanation of how it
works. Check it against the feature's `.md` file. This skill compounds more than
any framework knowledge.

### Books worth the time

- *A Philosophy of Software Design*, Ousterhout — short; about where complexity
  comes from. The most directly applicable book for the "my design gets picked
  apart" problem.
- *Working Effectively with Legacy Code*, Feathers — about changing code safely
  when you cannot test it easily. This repo, exactly.
- *The Pragmatic Programmer* — broad, but the chapters on orthogonality and
  assertions apply daily here.

---

## 5. Habits that change review outcomes fastest

1. **Ask before you build, not after.** Two minutes of "I plan to put the state
   in X and the db call in Y, sound right?" prevents a full rewrite at review.
   Seniors read a design paragraph far faster than a 400-line diff.

2. **Find the precedent first.** Before writing anything new, search for a
   feature that solved the same shape of problem and follow it. A diff that
   matches the house style gets reviewed on its logic; one that does not gets
   reviewed on its style, and the logic bugs slip through.

3. **Make the diff small.** Formatting, renames, and refactors go in separate
   commits or separate MRs. Reviewers find real bugs in 100-line diffs and miss
   them in 800-line ones. Half of "too many bugs caught" can be "too much in one
   MR".

4. **Never suppress an analyzer complaint.** `// ignore:` is a confession. The
   list of disabled lints at the bottom of `analysis_options.yaml` is labelled
   `TODO: remove following suppressions once fixed` — do not add to it.

5. **Run the thing.** On a real device, on the feature you changed, and once on
   an adjacent feature you might have broken. Especially after touching anything
   in `common/`.

6. **Write the test for the bug you just fixed.** One test. It converts a fix
   into a permanent guarantee, and it is the strongest signal of seniority in an
   MR.
