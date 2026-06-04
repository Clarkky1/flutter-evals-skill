---
name: flutter-evals
description: Flutter test quality evaluation and evaluation-driven development. Covers unit test quality scoring, widget test patterns, golden (screenshot) tests, integration test architecture, performance benchmarking, accessibility testing, test coverage enforcement, CI/CD test pipelines, and a rubric for evaluating whether a Flutter test suite is actually trustworthy. Use when assessing test quality, setting up an eval harness, or deciding whether a Flutter codebase is ship-ready.
origin: custom
version: 1.0.0
---

# Flutter Evals — Test Quality and Evaluation-Driven Development

## When to Use

- Assessing whether a Flutter test suite is trustworthy before shipping
- Setting up golden tests, integration tests, or performance benchmarks from scratch
- Evaluating a PR's test coverage quality — not just quantity
- Establishing a pass/fail rubric for automated quality gates in CI

---

## 1. The Flutter Test Pyramid

```
         /  Integration Tests  \       <- few, slow, high confidence
        /  (patrol, flutter_driver)\
       /----------------------------\
      /      Widget Tests            \  <- moderate, fast, visual + behavioral
     / (testWidgets, ProviderScope)   \
    /----------------------------------\
   /          Unit Tests               \ <- many, instant, logic only
  / (dart test, mocktail fakes)         \
 /________________________________________\
```

**Rule of thumb:**
- Unit: 70% of tests — every repository method, every notifier, every utility
- Widget: 20% — every screen's happy path, loading, empty, and error states
- Integration: 10% — critical user journeys only (sign in, checkout, onboarding)

---

## 2. Unit Tests — Quality Rubric

A unit test is trustworthy if it:

```
[+1] Tests behavior, not implementation (calls a method, checks output)
[+1] Uses fakes/stubs, not mocks with verify() — prefer mocktail over mockito
[+1] Covers the happy path AND at least one error path
[+1] Name describes behavior: "returns empty list when repository throws"
[+1] No setup that is longer than the test itself
[-1] Calls real I/O (network, file system, database)
[-1] Depends on test execution order
[-1] Asserts on private state or internal implementation details
[-1] Uses sleep() or delay to handle timing
```

Score 4+/5 = trustworthy. Below 3 = rewrite.

### Pattern: Fake over Mock

```dart
// PREFER: fake — real behavior, no verify() brittleness
class FakeAuthRepository implements AuthRepository {
  var shouldThrow = false;
  User? userToReturn;

  @override
  Future<User> signIn(String email, String password) async {
    if (shouldThrow) throw AuthError.invalidCredentials;
    return userToReturn ?? User(id: 'fake-id', email: email);
  }

  @override
  Stream<User?> get authStateChanges => Stream.value(userToReturn);
}

// Test
test('signIn sets state to success', () async {
  final repo = FakeAuthRepository()
    ..userToReturn = User(id: '1', email: 'a@b.com');
  final notifier = LoginNotifier(repo);

  await notifier.signIn('a@b.com', 'pw');

  expect(notifier.state, isA<AsyncData<void>>());
});

test('signIn sets state to error on failure', () async {
  final repo = FakeAuthRepository()..shouldThrow = true;
  final notifier = LoginNotifier(repo);

  await notifier.signIn('a@b.com', 'wrong');

  expect(notifier.state, isA<AsyncError<void>>());
});
```

### Riverpod Unit Tests

```dart
test('cartTotal returns 0 when cart is empty', () {
  final container = ProviderContainer(
    overrides: [
      cartNotifierProvider.overrideWith(() => FakeCartNotifier([])),
      productsProvider.overrideWith((ref) => AsyncData(fakeProducts)),
    ],
  );
  addTearDown(container.dispose);

  expect(container.read(cartTotalProvider), 0.0);
});
```

---

## 3. Widget Tests — Quality Rubric

A widget test is trustworthy if it:

```
[+1] Pumps the widget in a real ProviderScope / BlocProvider
[+1] Tests user-visible behavior (text, button, loading indicator)
[+1] Covers loading, error, and empty states — not just success
[+1] Uses find.byKey() sparingly — prefer find.text(), find.byType()
[+1] Calls pumpAndSettle() only when needed (not as a default wait)
[-1] Uses GlobalKey or accesses widget state directly
[-1] Only tests the happy path
[-1] Has > 5 lines of setup before the first action
[-1] Asserts on widget implementation details (e.g. widget.color == red)
```

### Pattern: Screen Test with Riverpod

```dart
testWidgets('ProductListScreen shows loading then list', (tester) async {
  final fakeRepo = FakeProductRepository()
    ..products = [Product(id: '1', name: 'Widget', price: 9.99)];

  await tester.pumpWidget(
    ProviderScope(
      overrides: [
        productRepositoryProvider.overrideWithValue(fakeRepo),
      ],
      child: const MaterialApp(home: ProductListScreen()),
    ),
  );

  // Loading state
  expect(find.byType(CircularProgressIndicator), findsOneWidget);

  await tester.pump(); // let Future resolve

  // Success state
  expect(find.text('Widget'), findsOneWidget);
  expect(find.byType(CircularProgressIndicator), findsNothing);
});

testWidgets('ProductListScreen shows error state on failure', (tester) async {
  final fakeRepo = FakeProductRepository()..shouldThrow = true;

  await tester.pumpWidget(
    ProviderScope(
      overrides: [productRepositoryProvider.overrideWithValue(fakeRepo)],
      child: const MaterialApp(home: ProductListScreen()),
    ),
  );

  await tester.pump();

  expect(find.text('Something went wrong'), findsOneWidget);
  expect(find.text('Retry'), findsOneWidget);
});
```

### Testing Navigation

```dart
testWidgets('tapping product navigates to detail', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [...],
      child: MaterialApp.router(routerConfig: testRouter),
    ),
  );

  await tester.pump();
  await tester.tap(find.text('Widget'));
  await tester.pumpAndSettle();

  expect(find.byType(ProductDetailScreen), findsOneWidget);
});
```

---

## 4. Golden Tests — Screenshot Regression

Golden tests capture a pixel-perfect screenshot and fail if any pixel changes. Use for design-critical components.

### Setup

```yaml
# pubspec.yaml
dev_dependencies:
  golden_toolkit: ^0.15.0  # or alchemist for multi-device
  # OR
  alchemist: ^0.7.0
```

### Pattern: Component Golden Test

```dart
import 'package:golden_toolkit/golden_toolkit.dart';

void main() {
  group('ProductCard golden', () {
    testGoldens('renders correctly in light mode', (tester) async {
      await loadAppFonts(); // load real fonts for accurate rendering

      await tester.pumpWidgetBuilder(
        ProductCard(
          product: Product(id: '1', name: 'Widget', price: 9.99),
        ),
        wrapper: materialAppWrapper(theme: AppTheme.light()),
        surfaceSize: const Size(375, 120),
      );

      await screenMatchesGolden(tester, 'product_card_light');
    });

    testGoldens('renders correctly in dark mode', (tester) async {
      await loadAppFonts();

      await tester.pumpWidgetBuilder(
        ProductCard(
          product: Product(id: '1', name: 'Widget', price: 9.99),
        ),
        wrapper: materialAppWrapper(theme: AppTheme.dark()),
        surfaceSize: const Size(375, 120),
      );

      await screenMatchesGolden(tester, 'product_card_dark');
    });
  });
}
```

```bash
# Generate / update golden files
flutter test --update-goldens

# Run comparison
flutter test
```

**Gotchas:**
- Goldens are OS and font-renderer dependent — run them in a consistent CI environment (Docker)
- Never update goldens without reviewing the diff visually
- Use `tolerance` parameter for anti-aliasing differences on different machines

### When to Use Goldens

| Use | Don't use |
|---|---|
| Design-system components (buttons, cards, chips) | Screens with dynamic data |
| Typography-heavy layouts | Anything with animations |
| Custom painters and canvas output | Platform-specific UI (cupertino widgets) |
| Icon and illustration placement | Lists with variable content |

---

## 5. Integration Tests

Integration tests run the full app on a real device or emulator. Use for critical user journeys only.

### Setup with patrol (recommended over flutter_driver)

```yaml
dev_dependencies:
  patrol: ^3.0.0
```

```dart
// integration_test/sign_in_test.dart
import 'package:patrol/patrol.dart';

void main() {
  patrolTest(
    'user can sign in and see home screen',
    ($) async {
      await $.pumpWidgetAndSettle(const MyApp());

      await $(#emailField).enterText('user@example.com');
      await $(#passwordField).enterText('password123');
      await $('Sign In').tap();

      await $.pumpAndSettle();

      expect($(HomeScreen), findsOneWidget);
    },
  );
}
```

```bash
# Run on connected device
flutter test integration_test/sign_in_test.dart
```

### Integration Test Scenarios to Cover

Cover these — nothing else warrants the overhead:

1. **Sign in / sign out** — auth state changes are the highest-risk flow
2. **Onboarding completion** — first-run state initialization
3. **Core purchase or conversion flow** — checkout, subscription, or primary CTA
4. **Deep link navigation** — link opens correct screen with correct state
5. **Offline behavior** — key screens work without network (if offline-first)

### patrol vs flutter_driver vs integration_test

| Package | Use when |
|---|---|
| `patrol` | Recommended for new projects. Native interactions, system dialogs, notifications |
| `integration_test` (Flutter SDK) | Simple flows, no system UI interaction needed |
| `flutter_driver` | Legacy projects only — deprecated |

---

## 6. Performance Benchmarking

Measure before you optimize. Use `flutter_benchmark` or the built-in benchmarking tools.

### Frame Timing Benchmark

```dart
// benchmark/scroll_benchmark.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';

void main() {
  final binding = IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('product list scroll performance', (tester) async {
    await tester.pumpWidget(const MyApp());
    await tester.pumpAndSettle();

    final gesture = await tester.startGesture(const Offset(0, 300));

    await binding.watchPerformance(() async {
      for (int i = 0; i < 5; i++) {
        await gesture.moveBy(const Offset(0, -300));
        await tester.pumpAndSettle();
      }
    }, reportKey: 'product_list_scroll');
  });
}
```

```bash
flutter drive \
  --driver=test_driver/perf_driver.dart \
  --target=benchmark/scroll_benchmark.dart \
  --profile
```

### Benchmark Thresholds (pass/fail)

| Metric | Target | Fail |
|---|---|---|
| Average frame time | < 8ms (120fps) / < 16ms (60fps) | > 16ms average |
| 90th percentile frame time | < 16ms | > 32ms |
| Missed frames | < 5% | > 10% |
| App startup (cold) | < 2s | > 4s |
| Memory at rest | < 150MB | > 300MB |

---

## 7. Accessibility Testing

```dart
testWidgets('ProductCard meets accessibility requirements', (tester) async {
  final handle = tester.ensureSemantics();

  await tester.pumpWidget(
    MaterialApp(
      home: ProductCard(
        product: Product(id: '1', name: 'Widget', price: 9.99),
      ),
    ),
  );

  // Check semantic labels exist
  expect(
    tester.getSemantics(find.byType(ProductCard)),
    matchesSemantics(
      isButton: true,
      label: 'Widget, \$9.99',
      hasTapAction: true,
    ),
  );

  handle.dispose();
});
```

### axe-flutter for automated a11y scanning

```yaml
dev_dependencies:
  axe_flutter: ^0.1.0
```

```dart
testWidgets('HomeScreen has no accessibility violations', (tester) async {
  await tester.pumpWidget(const MaterialApp(home: HomeScreen()));
  await tester.pumpAndSettle();

  final violations = await axeFlutter.analyze(tester);
  expect(violations, isEmpty);
});
```

---

## 8. Coverage Enforcement

80% minimum. Measure with intent — 100% coverage on trivial getters while missing repository error paths is worse than 70% on critical paths.

```bash
# Run with coverage
flutter test --coverage

# Generate HTML report
genhtml coverage/lcov.info -o coverage/html
open coverage/html/index.html

# Check coverage threshold (CI)
lcov --summary coverage/lcov.info | grep "lines" # must be >= 80%
```

### What 80% Actually Means

| Must be tested (count toward coverage) | Acceptable to skip |
|---|---|
| All repository methods — happy + error path | Generated code (`*.g.dart`, `*.freezed.dart`) |
| All notifier/cubit state transitions | Platform channel implementations |
| All input validation logic | `main.dart` entry point |
| All utility functions | Firebase initialization boilerplate |
| Error mapping and user-facing messages | |

### Exclude Generated Files from Coverage

```yaml
# analysis_options.yaml or lcov configuration
# Exclude generated files
```

```bash
# Remove generated files from coverage report
lcov --remove coverage/lcov.info \
  '*.g.dart' \
  '*.freezed.dart' \
  'lib/generated/*' \
  -o coverage/filtered.lcov.info
```

---

## 9. CI/CD Test Pipeline

```yaml
# .github/workflows/flutter_tests.yml
name: Flutter Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.22.x'
          cache: true

      - name: Install dependencies
        run: flutter pub get

      - name: Analyze
        run: flutter analyze --fatal-infos

      - name: Format check
        run: dart format --output=none --set-exit-if-changed .

      - name: Unit + Widget tests with coverage
        run: flutter test --coverage

      - name: Check coverage threshold
        run: |
          COVERAGE=$(lcov --summary coverage/lcov.info 2>&1 | grep "lines" | grep -o '[0-9.]*%' | head -1 | tr -d '%')
          echo "Coverage: $COVERAGE%"
          python3 -c "import sys; sys.exit(0 if float('$COVERAGE') >= 80 else 1)"

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: coverage/lcov.info

  golden:
    runs-on: ubuntu-latest  # must match OS used to generate goldens
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
      - run: flutter pub get
      - run: flutter test --tags golden
```

---

## 10. Test Quality Scorecard

Use this rubric before marking a feature "done":

| Category | Check | Points |
|---|---|---|
| **Unit** | Repository happy + error path tested | 10 |
| **Unit** | All notifier/cubit states covered | 10 |
| **Unit** | Fakes used (no verify() mocks) | 5 |
| **Widget** | Loading, success, error, empty states | 10 |
| **Widget** | Navigation flow tested | 5 |
| **Golden** | Design-system components have goldens | 10 |
| **Integration** | Critical user journey covered | 15 |
| **A11y** | Semantics verified on key screens | 10 |
| **Coverage** | >= 80% on non-generated code | 15 |
| **CI** | All tests run on PR | 10 |
| **Total** | | 100 |

**Score 90+**: ship it.
**Score 70–89**: acceptable with known gaps documented.
**Score below 70**: not ready — identify and fill the gaps.

---

## Key References

- Flutter testing docs: https://docs.flutter.dev/testing/overview
- patrol: https://pub.dev/packages/patrol
- golden_toolkit: https://pub.dev/packages/golden_toolkit
- alchemist: https://pub.dev/packages/alchemist
- mocktail: https://pub.dev/packages/mocktail
- flutter_lints: https://pub.dev/packages/flutter_lints
- Very Good Ventures testing guide: https://verygood.ventures/blog/flutter-testing-best-practices
