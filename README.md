<div align="center">

# flutter-evals

### Flutter Test Quality and Evaluation-Driven Development

<br/>

![Version](https://img.shields.io/badge/version-1.0.0-6366f1?style=for-the-badge&labelColor=1e1b4b)
![Platform](https://img.shields.io/badge/Flutter-testing-02569B?style=for-the-badge&labelColor=013A6B)
![Coverage](https://img.shields.io/badge/coverage-80%25_minimum-10b981?style=for-the-badge&labelColor=064e3b)
![Tools](https://img.shields.io/badge/works_in-any_AI_tool-f97316?style=for-the-badge&labelColor=431407)

<br/>

> **Unit tests. Widget tests. Golden tests. patrol integration tests. Performance benchmarks. Accessibility testing. A 100-point quality scorecard. Know if your Flutter app is actually ready to ship.**

<br/>

</div>

---

## What This Covers

| Topic | What's included |
|---|---|
| **Test pyramid** | 70/20/10 split — unit, widget, integration with clear rationale |
| **Unit test rubric** | +/- scoring system, fake-over-mock pattern, Riverpod container tests |
| **Widget tests** | Loading, success, error, and empty states; navigation testing; ProviderScope overrides |
| **Golden tests** | `golden_toolkit` and `alchemist` setup, when to use goldens, OS consistency pitfalls |
| **Integration tests** | `patrol` (recommended), `integration_test` (Flutter SDK), why not `flutter_driver` |
| **Performance benchmarks** | Frame timing tests, pass/fail thresholds, `watchPerformance` API |
| **Accessibility testing** | Semantics assertions, `axe_flutter` automated scanning |
| **Coverage enforcement** | `lcov` threshold check, what to exclude (generated files), meaningful vs vanity coverage |
| **CI/CD pipeline** | GitHub Actions template with analyze, format, test, coverage gate, and golden comparison |
| **Quality scorecard** | 100-point rubric — score 90+ to ship, below 70 means stop |

---

## Installation

### Claude Code

```bash
npx skills add Clarkky1/flutter-evals-skill@flutter-evals -g -y
```

Invoke with `/flutter-evals` or say "evaluate this Flutter test suite".

### Any Other AI Tool

| Tool | How to use |
|---|---|
| **Gemini / ChatGPT / Claude.ai** | Paste the contents of `SKILL.md` as your system prompt or first message |
| **Cursor** | Copy `SKILL.md` content into `.cursorrules` in your project root |
| **Windsurf** | Copy into `.windsurfrules` |
| **GitHub Copilot** | Copy into `.github/copilot-instructions.md` |
| **Any other tool** | Paste relevant sections directly into the chat |

---

## The Quality Scorecard

Use this before marking any feature done:

| Category | Check | Points |
|---|---|---|
| Unit | Repository happy + error path tested | 10 |
| Unit | All notifier/cubit state transitions covered | 10 |
| Unit | Fakes used — no `verify()` mocks | 5 |
| Widget | Loading, success, error, empty states | 10 |
| Widget | Navigation flow tested | 5 |
| Golden | Design-system components have goldens | 10 |
| Integration | Critical user journey covered | 15 |
| Accessibility | Semantics verified on key screens | 10 |
| Coverage | >= 80% on non-generated code | 15 |
| CI | All tests run on every PR | 10 |
| **Total** | | **100** |

**90+**: ship it. **70–89**: acceptable with gaps documented. **Below 70**: not ready.

---

## Test Pyramid

```
         /  Integration Tests  \       few, slow, high confidence
        /  patrol / integration_test \
       /----------------------------\
      /       Widget Tests           \  moderate, fast, visual + behavioral
     / testWidgets + ProviderScope    \
    /----------------------------------\
   /          Unit Tests               \ many, instant, logic only
  / dart test + mocktail fakes          \
 /________________________________________\

70% unit  ·  20% widget  ·  10% integration
```

---

## Unit Test Pattern — Fake over Mock

```dart
// Fake: real behavior, no brittle verify() calls
class FakeAuthRepository implements AuthRepository {
  var shouldThrow = false;
  User? userToReturn;

  @override
  Future<User> signIn(String email, String password) async {
    if (shouldThrow) throw AuthError.invalidCredentials;
    return userToReturn ?? User(id: 'fake-id', email: email);
  }
}

test('signIn sets state to error on failure', () async {
  final notifier = LoginNotifier(FakeAuthRepository()..shouldThrow = true);
  await notifier.signIn('a@b.com', 'wrong');
  expect(notifier.state, isA<AsyncError<void>>());
});
```

---

## Widget Test Pattern — All Four States

```dart
testWidgets('shows loading, then list, then handles error', (tester) async {
  // Loading
  await tester.pumpWidget(ProviderScope(
    overrides: [productRepoProvider.overrideWithValue(SlowFakeRepo())],
    child: const MaterialApp(home: ProductListScreen()),
  ));
  expect(find.byType(CircularProgressIndicator), findsOneWidget);

  // Success
  await tester.pump();
  expect(find.text('Widget'), findsOneWidget);

  // Error
  await tester.pumpWidget(ProviderScope(
    overrides: [productRepoProvider.overrideWithValue(ThrowingFakeRepo())],
    child: const MaterialApp(home: ProductListScreen()),
  ));
  await tester.pump();
  expect(find.text('Something went wrong'), findsOneWidget);
  expect(find.text('Retry'), findsOneWidget);
});
```

---

## Golden Test Setup

```bash
# Add to pubspec.yaml dev_dependencies
golden_toolkit: ^0.15.0
# OR
alchemist: ^0.7.0
```

```dart
testGoldens('ProductCard light + dark', (tester) async {
  await loadAppFonts();
  for (final theme in [AppTheme.light(), AppTheme.dark()]) {
    await tester.pumpWidgetBuilder(
      ProductCard(product: fakeProduct),
      wrapper: materialAppWrapper(theme: theme),
      surfaceSize: const Size(375, 120),
    );
    await screenMatchesGolden(tester, 'product_card_${theme.brightness.name}');
  }
});
```

```bash
flutter test --update-goldens  # generate
flutter test                   # compare
```

---

## patrol Integration Test

```dart
patrolTest('user can sign in and reach home', ($) async {
  await $.pumpWidgetAndSettle(const MyApp());
  await $(#emailField).enterText('user@example.com');
  await $(#passwordField).enterText('password123');
  await $('Sign In').tap();
  await $.pumpAndSettle();
  expect($(HomeScreen), findsOneWidget);
});
```

---

## CI Pipeline

```yaml
# .github/workflows/flutter_tests.yml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: subosito/flutter-action@v2
      - run: flutter pub get
      - run: flutter analyze --fatal-infos
      - run: dart format --output=none --set-exit-if-changed .
      - run: flutter test --coverage
      - name: Enforce 80% coverage
        run: |
          COV=$(lcov --summary coverage/lcov.info 2>&1 | grep "lines" | grep -o '[0-9.]*%' | head -1 | tr -d '%')
          python3 -c "import sys; sys.exit(0 if float('$COV') >= 80 else 1)"

  golden:
    runs-on: ubuntu-latest  # must match OS used to generate goldens
    steps:
      - uses: subosito/flutter-action@v2
      - run: flutter pub get
      - run: flutter test --tags golden
```

---

## Performance Thresholds

| Metric | Target | Fail |
|---|---|---|
| Average frame time | < 16ms (60fps) | > 16ms average |
| 90th percentile | < 16ms | > 32ms |
| Missed frames | < 5% | > 10% |
| Cold app startup | < 2s | > 4s |
| Memory at rest | < 150MB | > 300MB |

---

## Companion Skills

| Skill | What it adds |
|---|---|
| [`flutter-google`](https://github.com/Clarkky1/flutter-google-skill) | Effective Dart, Material 3, debugging, pub.dev evaluation, architecture |
| [`dart-flutter-patterns`](https://pub.dev) | BLoC/Riverpod patterns, GoRouter, Freezed |
| [`flutter-dart-code-review`](https://pub.dev) | Library-agnostic code review checklist |

---

## Key References

- [Flutter testing overview](https://docs.flutter.dev/testing/overview)
- [patrol](https://pub.dev/packages/patrol)
- [golden_toolkit](https://pub.dev/packages/golden_toolkit)
- [alchemist](https://pub.dev/packages/alchemist)
- [mocktail](https://pub.dev/packages/mocktail)
- [axe_flutter](https://pub.dev/packages/axe_flutter)
- [Very Good Ventures testing guide](https://verygood.ventures/blog/flutter-testing-best-practices)

---

Using this skill? [Open an issue](https://github.com/Clarkky1/flutter-evals-skill/issues) or reply on X [@kinnnparksung](https://x.com/kinnnparksung).
