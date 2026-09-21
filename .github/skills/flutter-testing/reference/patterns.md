# Test patterns that work

## 1. A fake repository (preferred over mocks)

```dart
// test/fakes/fake_call_history_repository.dart
class FakeCallHistoryRepository implements CallHistoryRepository {
  FakeCallHistoryRepository({this.result = const Result.ok(<Call>[])});

  Result<List<Call>> result;
  int callCount = 0;

  @override
  Future<Result<List<Call>>> fetchCalls({int page = 1}) async {
    callCount++;
    return result;
  }
}
```

## 2. View-model unit test — happy path plus the failure

```dart
void main() {
  group('CallListViewModel', () {
    test('starts in loading and exposes data when the repository succeeds', () async {
      final repository = FakeCallHistoryRepository(
        result: Result.ok([const Call(peer: 'som', duration: Duration(minutes: 2))]),
      );
      final viewModel = CallListViewModel(repository: repository);

      expect(viewModel.state, isA<CallListLoading>());

      await viewModel.load();

      expect(viewModel.state, isA<CallListData>());
      expect(repository.callCount, 1);
    });

    test('maps a timeout to a typed failure instead of throwing', () async {
      final repository = FakeCallHistoryRepository(
        result: const Result.failure(Failure.timeout()),
      );
      final viewModel = CallListViewModel(repository: repository);

      await viewModel.load();

      expect(viewModel.state, isA<CallListFailed>());
      expect((viewModel.state as CallListFailed).failure.isRetryable, isTrue);
    });

    test('shows empty rather than error when the server returns no rows', () async {
      final viewModel = CallListViewModel(
        repository: FakeCallHistoryRepository(result: const Result.ok(<Call>[])),
      );

      await viewModel.load();

      expect(viewModel.state, isA<CallListEmpty>());
    });
  });
}
```

Always `addTearDown(viewModel.dispose)` when the view model owns controllers, timers or streams.

## 3. Four-state widget test

```dart
Widget _harness(CallListViewModel viewModel) => MaterialApp(
      localizationsDelegates: AppLocalizations.localizationsDelegates,
      supportedLocales: AppLocalizations.supportedLocales,
      home: CallListView(viewModel: viewModel),
    );

testWidgets('renders loading, then data with two tiles', (tester) async {
  final repository = FakeCallHistoryRepository(
    result: Result.ok([_call('som'), _call('tum')]),
  );
  final viewModel = CallListViewModel(repository: repository)..load();

  await tester.pumpWidget(_harness(viewModel));
  expect(find.byType(CallListSkeleton), findsOneWidget);   // loading state
  await tester.pumpAndSettle();

  expect(find.byKey(const ValueKey('call-tile')), findsNWidgets(2));
});

testWidgets('shows the empty state copy when there are no calls', (tester) async {
  final viewModel = CallListViewModel(
    repository: FakeCallHistoryRepository(result: const Result.ok(<Call>[])),
  )..load();

  await tester.pumpWidget(_harness(viewModel));
  await tester.pumpAndSettle();

  expect(find.text(tester.element(find.byType(CallListView)).l10n.noCalls), findsOneWidget);
});

testWidgets('error state offers retry and retrying recovers', (tester) async {
  final repository = FakeCallHistoryRepository(
    result: const Result.failure(Failure.timeout()),
  );
  final viewModel = CallListViewModel(repository: repository)..load();
  await tester.pumpWidget(_harness(viewModel));
  await tester.pumpAndSettle();

  expect(find.byType(ErrorView), findsOneWidget);

  repository.result = Result.ok([_call('som')]);
  await tester.tap(find.byKey(const ValueKey('retry-button')));
  await tester.pumpAndSettle();

  expect(find.byType(CallListBody), findsOneWidget);
});
```

Notes that save hours:

- Find by `Key`, text, or semantic label. `find.byType(Container)` chains break on every refactor.
- `pumpAndSettle` only when the animation really settles; otherwise `pump(const Duration(…))`.
  A `pumpAndSettle` that times out is usually a real bug (infinite animation, pending timer).
- Localised text in assertions: assert against `context.l10n.<key>`, not the literal — otherwise
  the test breaks on a copy change and quietly encodes the wrong language.

## 4. Layout / accessibility test (turns rule 13 into something CI can hold you to)

```dart
testWidgets('no overflow at 320 dp with textScaler 2.0', (tester) async {
  tester.view.physicalSize = const Size(320, 640);
  tester.view.devicePixelRatio = 1;
  tester.platformDispatcher.textScaleFactorTestValue = 2.0;
  addTearDown(tester.view.reset);

  await tester.pumpWidget(_harness(viewModelWithLongContent));
  await tester.pumpAndSettle();

  expect(tester.takeException(), isNull);   // RenderFlex overflow throws
});
```

Also assert touch targets where you wrote them:

```dart
expect(tester.getSize(find.byKey(const ValueKey('icon-only-call-button'))), 
    hasSize(const Size(48, 48)));
```

## 5. Golden test for shared/design-system widgets

```dart
testWidgets('CallTile matches golden in light and dark', (tester) async {
  await tester.pumpWidget(_themed(CallTile(peer: 'som', duration: const Duration(minutes: 2))));
  await expectLater(
    find.byType(CallTile),
    matchesGoldenFile('goldens/call_tile.light.png'),
  );
});
```

Goldens are for shared widgets and the app shell — not for every screen (they fail on every
intentional design change and get rubber-stamped).

## 6. Integration test skeleton

```dart
// integration_test/login_flow_test.dart
void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('signs in and lands on the call list', (tester) async {
    await tester.pumpWidget(const App(config: AppConfig(apiBaseUrl: kStagingUrl)));
    await tester.pumpAndSettle();

    await tester.enterText(find.byKey(const ValueKey('username')), 'demo');
    await tester.enterText(find.byKey(const ValueKey('password')), 'demo-pass');
    await tester.tap(find.byKey(const ValueKey('sign-in')));
    await tester.pumpAndSettle();

    expect(find.byType(CallListView), findsOneWidget);
  });
}
```

Run order (also what the review will ask for):

```bash
dart format .
flutter analyze --fatal-infos
flutter test --coverage            # unit + widget + goldens
flutter test integration_test      # needs a device/emulator
```

## 7. Anti-patterns that make a suite worthless

- A test with no assertion, or asserting `isNotNull` on everything.
- Logic inside a test (`for` loops over expectations, `if` branches) — it can pass while the code
  is broken.
- Testing private members; testing generated code; testing the framework.
- Mocking a class you could construct (a fake is clearer and less brittle).
- `await Future.delayed(const Duration(seconds: 1))` to "wait for" async work.
- One giant integration test instead of the pyramid — slow, flaky, and it hides which layer broke.
- Leaving a test skipped forever. Fix it in the change or delete it and say why.
