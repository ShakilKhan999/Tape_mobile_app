# Flutter App Architecture Guide (Riverpod)

## Project Overview

This Flutter application follows a **feature-first architecture** with **Riverpod state management**, implementing a modular, testable structure for building responsive, maintainable UI screens.

This file is the project's architecture reference. `CLAUDE.md` points at it via `@docs/riverpod-state-management-guide.md` — this is what gets read before any feature code is written.

**Assumed to exist elsewhere, not defined in this document:** `authSecureStoreProvider` (secure local storage for the access token, read by `auth_interceptor.dart`) and `authStateProvider` (current login state, read by `app_router.dart`'s redirect guard). Both are auth/session concerns that belong in their own feature, not `core/`. If either doesn't exist yet, the interceptor and the router guard won't compile until it does.

---

## Architecture Layers

### 1. Core Layer (`/lib/core/`)
Shared utilities, common widgets, design system, network layer, routing, and configuration. Nothing feature-specific lives here.

### 2. Application Layer (`/lib/application/`)
Framework-agnostic shared code with zero Flutter or third-party imports — currently just the shared exception type.

### 3. Features Layer (`/lib/features/`)
Feature-specific modules organized by domain. Each feature is self-contained: its own models, repositories, providers, notifiers, widgets, and pages.

### 4. Routes Layer (`/lib/core/router/`)
Centralized navigation using `go_router`, wired as a Riverpod provider so route guards can read app state.

---

## Project Structure

```
lib/
├── main.dart
├── app.dart                          # MaterialApp.router configuration
├── l10n/                             # localization .arb files
│
├── core/
│   ├── common/
│   │   ├── styles/
│   │   │   ├── global_text_style.dart
│   │   │   └── app_theme.dart
│   │   └── widgets/
│   │       ├── common_background.dart
│   │       ├── unsupported_device_screen.dart
│   │       ├── [FILENAME].dart          # custom loading indicator
│   │       ├── customize_filled_button.dart
│   │       ├── customize_unfilled_button.dart
│   │       └── custom_text_field.dart
│   ├── utils/
│   │   ├── sizer.dart
│   │   ├── connectivity_service.dart
│   │   └── constants/
│   │       ├── colors.dart
│   │       ├── app_urls.dart
│   │       └── asset_paths.dart
│   ├── network/
│   │   ├── error_mapper.dart
│   │   ├── network_hofs.dart
│   │   └── auth_interceptor.dart
│   ├── providers/
│   │   ├── dio_provider.dart
│   │   └── theme_mode_provider.dart
│   └── router/
│       └── app_router.dart
│
├── application/
│   └── exceptions/
│       └── network_exceptions.dart
│
└── features/
    └── <feature_name>/
        ├── data/
        │   ├── models/
        │   ├── remote/
        │   └── repositories/
        └── presentation/
            ├── notifiers/
            ├── widgets/
            └── pages/
```

`.g.dart` and `.freezed.dart` files omitted throughout — generated, never opened by hand.

---

## Design System Implementation

### 1. Responsive Design with Sizer

**Strategy:** optimized for mobile phones, using `MediaQuery` under the hood.

**File:** `/lib/core/utils/sizer.dart`

```dart
class Sizer {
  static late double _screenWidth;
  static late double _screenHeight;
  static late double _blockSizeHorizontal;
  static late double _blockSizeVertical;
  static late TextScaler _textScaler;

  static void init(BuildContext context) {
    final Size size = MediaQuery.sizeOf(context);
    _screenWidth = size.width;
    _screenHeight = size.height;
    _blockSizeHorizontal = _screenWidth / designWidth;
    _blockSizeVertical = _screenHeight / designHeight;
    _textScaler = MediaQuery.textScalerOf(context);
  }

  static double w(double width) => width * _blockSizeHorizontal;
  static double h(double height) => height * _blockSizeVertical;
  static double sp(double fontSize) => _textScaler.scale(fontSize * _blockSizeHorizontal);
  static double r(double radius) => radius * _blockSizeHorizontal;

  static Widget verticalSpace(double height) => SizedBox(height: h(height));
  static Widget horizontalSpace(double width) => SizedBox(width: w(width));
}

extension SizerExt on num {
  double get w => Sizer.w(toDouble());
  double get h => Sizer.h(toDouble());
  double get sp => Sizer.sp(toDouble());
  double get r => Sizer.r(toDouble());
  Widget get verticalSpace => Sizer.verticalSpace(toDouble());
  Widget get horizontalSpace => Sizer.horizontalSpace(toDouble());
}
```

**`.sp()` now composes two independent scale factors, not one.** `fontSize * _blockSizeHorizontal` still handles proportional scaling across different phone screen widths — unchanged. `_textScaler.scale(...)` wraps that result and applies the user's own OS-level accessibility text size preference on top. A user who's bumped their system font size up now actually sees larger text in this app; before this fix, that setting was silently ignored. `.w`/`.h`/`.r` stay exactly as they were — text scale factor only applies to text, not spacing or corner radius.

**Setup:** `Sizer.init(context)` runs once, inside `MaterialApp.router`'s `builder` callback — see the complete `app.dart` at the end of the Routing section for exactly how this fits with everything else wired there.

**Usage — unchanged, throughout every screen:**
```dart
width: 160.w, height: 48.h, fontSize: 16.sp, borderRadius: 12.r,
16.verticalSpace, 20.horizontalSpace,
```

**What Sizer is for.** It scales proportions across different phone sizes — a button stays the right relative size on a small phone and a large one. It's not responsible for handling a tablet-sized window; that's the safety net below, not a design decision.

### 2. Orientation Lock and the Large-Screen Gate

Mobile-only target — no tablet or desktop layouts being built yet. Orientation lock still works exactly as before, because the platform change that affects it only kicks in at 600dp+ smallest width, which excludes actual phones entirely.

```dart
// lib/main.dart, before runApp
await SystemChrome.setPreferredOrientations([
  DeviceOrientation.portraitUp,
  DeviceOrientation.portraitDown,
]);
```

**The one real risk, worth guarding against on purpose:** Play Store doesn't block a phone-only app from being opened on a tablet or foldable by default. If that happens, the lock above is ignored on that device. Rather than quietly stretching the phone layout into the extra space, show it plainly — this is a real limitation today, not one to hide:

```dart
// lib/core/common/widgets/unsupported_device_screen.dart
class UnsupportedDeviceScreen extends StatelessWidget {
  const UnsupportedDeviceScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final scheme = Theme.of(context).colorScheme;
    return Scaffold(
      body: Center(
        child: Padding(
          padding: const EdgeInsets.all(24),
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              Icon(Icons.smartphone, size: 64, color: scheme.onSurfaceVariant),
              const SizedBox(height: 16),
              Text(
                'Best experienced on mobile',
                style: getTextStyle(fontSize: 20, fontWeight: FontWeight.w700, color: scheme.onSurface),
                textAlign: TextAlign.center,
              ),
              const SizedBox(height: 8),
              Text(
                'Tablet and larger screen support is coming soon. Please open this app on a phone for now.',
                style: getTextStyle(fontSize: 14, fontWeight: FontWeight.w400, color: scheme.onSurfaceVariant),
                textAlign: TextAlign.center,
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```
Same fix as Theme Configuration, applied here: colors come from `Theme.of(context).colorScheme`, not a fixed `AppColors` hex — so this screen reads correctly in dark mode too, instead of being the one place in the app that silently didn't get the memo.

Wired inside `MaterialApp.router`'s `builder`, the same place `Sizer.init` runs — the gate check happens before `child` is ever returned:

```dart
builder: (context, child) {
  Sizer.init(context);
  // TODO: remove this gate once real tablet/large-screen layouts exist.
  final isLargeScreen = MediaQuery.sizeOf(context).width >= 600;
  return isLargeScreen ? const UnsupportedDeviceScreen() : child!;
},
```

Same 600dp threshold Android itself uses to define "large screen" — deliberately, so the gate and the platform agree on what counts. `MediaQuery` is the right tool here, not `LayoutBuilder`: this is a whole-app decision made once at the root, not a per-widget layout choice. Full file, with this `builder` in context alongside everything else: end of the Routing section.

**Not doing this at the store level for now, on purpose:** Play Console can exclude tablets and large screens from the app's availability entirely — but since there's a real plan to add proper large-screen support later, hiding the app from those devices now would just mean undoing it then. The in-app gate above is the right temporary measure — visible, honest about the limitation, and deleted in one place once real tablet layouts ship.

### 3. Theme Configuration

**Not enough as written — two things missing, one real trap.** Only `light` exists; there's no `dark`, and `MaterialApp` isn't wired to switch between them. The trap: `AppColors.textPrimary` and `.backgroundLight` are fixed hex values. Reuse them for a dark theme and text goes near-invisible — dark text on a dark background, the exact failure official guidance warns about by name.

**The fix — one seed color, `brightness` does the rest.** Material 3's `ColorScheme.fromSeed` generates a full, accessible, coordinated palette for either mode from a single seed — that's the current recommendation over hand-picking each color, light and dark both:

**File:** `/lib/core/common/styles/app_theme.dart`

```dart
class AppTheme {
  static ThemeData get light => _build(Brightness.light);
  static ThemeData get dark => _build(Brightness.dark);

  static ThemeData _build(Brightness brightness) {
    final scheme = ColorScheme.fromSeed(seedColor: AppColors.primary, brightness: brightness);
    return ThemeData(
      useMaterial3: true,
      colorScheme: scheme,
      scaffoldBackgroundColor: scheme.surface,
      textTheme: TextTheme(
        headlineLarge: getTextStyle(fontSize: 36, fontWeight: FontWeight.w700, color: scheme.onSurface),
        headlineMedium: getTextStyle(fontSize: 32, fontWeight: FontWeight.w700, color: scheme.onSurface),
        bodyLarge: getTextStyle(fontSize: 16, fontWeight: FontWeight.w400, color: scheme.onSurface),
        bodyMedium: getTextStyle(fontSize: 14, fontWeight: FontWeight.w400, color: scheme.onSurfaceVariant),
      ),
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          backgroundColor: scheme.primary,
          shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
        ),
      ),
    );
  }
}
```

**The actual fix, stated plainly:** every color that came from `AppColors` (a fixed hex) now comes from `scheme` (`onSurface`, `onSurfaceVariant`, `surface`) instead — the same semantic slot, but one that resolves to a different real color depending on `brightness`. `AppColors.primary` still seeds the palette; it's just no longer read directly for text or backgrounds, which is exactly where the trap lived.

**Switching between them — `ThemeMode`, wired through Riverpod since that's the state layer already in place:**

```dart
// lib/core/providers/theme_mode_provider.dart
@riverpod
class ThemeModeNotifier extends _$ThemeModeNotifier {
  @override
  ThemeMode build() => ThemeMode.system;

  void setThemeMode(ThemeMode mode) => state = mode;
}
```

Wired into `MaterialApp.router`: `theme: AppTheme.light, darkTheme: AppTheme.dark, themeMode: ref.watch(themeModeNotifierProvider)` — three properties, alongside everything else. Full file: end of the Routing section.

`ThemeMode.system` as the default is the current recommendation — the app follows the device's setting automatically, no toggle needed for most users. A settings screen can still override it explicitly with `ref.read(themeModeNotifierProvider.notifier).setThemeMode(ThemeMode.dark)` whenever one exists.

**Not built yet, on purpose:** persisting a manual override across app restarts needs local storage — the same gap flagged earlier and deferred until a feature actually needs it. This is that feature, the moment a settings screen exists. Until then, `ThemeMode.system` means the app is always correct on launch regardless, since it's just reading the device's current setting fresh every time.

One `ThemeData` object, wired once in `app.dart`'s `theme:` property. Individual widgets read `Theme.of(context).textTheme.bodyLarge` when they want the app-wide default, or `getTextStyle(...)` directly when a screen needs a one-off size the theme doesn't define. Both are valid — the theme covers the common case, `getTextStyle` covers the exception.

### 4. Typography

No `google_fonts` package — plain Flutter local fonts instead. Since production means bundling the files ourselves either way, the package isn't earning its dependency, and it currently has an open bug where font loading breaks due to a Flutter engine change that removed `AssetManifest.json` support — exactly the mechanism the package relies on to find bundled files.

Download the weights actually used (400, 600, 700 — not the full 100–900 range) from fonts.google.com into `assets/fonts/`, plus a Bengali-covering fallback family for any non-Latin text:

```yaml
# pubspec.yaml
flutter:
  fonts:
    - family: Inter
      fonts:
        - asset: assets/fonts/Inter-Regular.ttf
          weight: 400
        - asset: assets/fonts/Inter-SemiBold.ttf
          weight: 600
        - asset: assets/fonts/Inter-Bold.ttf
          weight: 700
    - family: NotoSansBengali
      fonts:
        - asset: assets/fonts/NotoSansBengali-Regular.ttf
          weight: 400
```

**File:** `/lib/core/common/styles/global_text_style.dart`

```dart
TextStyle getTextStyle({
  required double fontSize,
  required FontWeight fontWeight,
  required Color color,
  double? lineHeight,
  String fontFamily = 'Inter',
}) {
  return TextStyle(
    fontFamily: fontFamily,
    fontFamilyFallback: const ['NotoSansBengali'],
    fontSize: fontSize,
    fontWeight: fontWeight,
    color: color,
    height: lineHeight != null ? lineHeight / fontSize : null,
  );
}
```

**Non-Latin script fallback — relevant the moment any Bengali text appears:** Inter has no Bengali glyphs. `fontFamilyFallback` is Flutter's built-in mechanism for exactly this — when a character isn't in `Inter`, it's looked up in `NotoSansBengali` automatically, per-character, inside the same string. No per-call branching needed; it's declared once, above, and every `getTextStyle` call already has it.

**Type scale:** H1 36sp/700 — H2 32sp/700 — Body/Button 16sp/500 — B1 16sp/400 — B2 14sp/400 — B3 12sp/400.

### 5. Color System

**File:** `/lib/core/utils/constants/colors.dart`

```dart
// lib/core/utils/constants/colors.dart
class AppColors {
  static const Color primary = Color(0xFF4A7BFF);
  static const Color secondary = Color(0xFF34C759);
  static const Color textPrimary = Color(0xFF161618);
  static const Color textSecondary = Color(0xFF8E8E93);
  static const Color backgroundLight = Color(0xFFF6F6F6);
  static const Color backgroundNeutral = Color(0xFFF5F5F7);
  static const Color inputBackground = Color(0xFFF8F8F8);
  static const Color inputBorder = Color(0xFFE8E8E8);
}
```

### 6. Localization

Flutter's built-in `gen-l10n`, not a third-party package — one less dependency, and it's what `flutter_localizations` is actually built to pair with.

**`pubspec.yaml` additions:**
```yaml
flutter:
  generate: true

dependencies:
  flutter_localizations:
    sdk: flutter
  intl: ^0.19.0
```

**`l10n.yaml`, at the project root:**
```yaml
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
```

**`lib/l10n/app_en.arb`:**
```json
{
  "loginButton": "Log In",
  "welcomeMessage": "Welcome back!"
}
```
Add `app_bn.arb` (or whichever locale) alongside it with the same keys, translated values.

**Wired into `MaterialApp.router`:** `localizationsDelegates: AppLocalizations.localizationsDelegates, supportedLocales: AppLocalizations.supportedLocales` — two more properties on the same widget everything else lands on. Full file: end of the Routing section.

**Usage in a screen:**
```dart
Text(AppLocalizations.of(context)!.loginButton)
```

Run `flutter gen-l10n` after adding or changing a key — same generated-file discipline as `build_runner`, just a different command.

---

## Common Widgets Architecture

### 1. Background System

**File:** `/lib/core/common/widgets/common_background.dart` — three-layer gradient (base gray, blue radial top-left, green radial bottom).

```dart
CommonBackgroundScaffold(
  body: YourContent(),
)
```

### 2. Buttons

**Filled:** `/lib/core/common/widgets/customize_filled_button.dart`
```dart
CustomFilledButton(
  text: 'Log In',
  onPressed: () => ref.read(loginNotifierProvider.notifier).submit(),
)
```

**Unfilled:** `/lib/core/common/widgets/customize_unfilled_button.dart`
```dart
CustomUnfilledButton(
  text: 'Create Account',
  onPressed: () => context.push('/register'),
  textColor: Theme.of(context).colorScheme.primary,
  borderColor: Theme.of(context).colorScheme.primary,
)
```

Both widgets just take a callback — what goes inside it is whatever the screen needs, usually a `ref.read(...notifier)` call or a `go_router` navigation.

### 3. Text Field

**File:** `/lib/core/common/widgets/custom_text_field.dart`
```dart
CustomTextField(
  label: 'Email or Phone',
  hintText: 'Enter your email address',
  controller: emailController,
  keyboardType: TextInputType.emailAddress,
  suffixIcon: const Icon(Icons.email_outlined),
)
```

Features unchanged: no border, responsive sizing via Sizer, label/hint support, prefix/suffix icons, password visibility toggle, form validation.

Validation splits across two layers and they never mix. Field-level rules
(required, format, length) live in the widget via `Form` + `GlobalKey<FormState>`
and run *before* the Notifier is called. Server-side rejection (wrong password,
duplicate email) arrives back as an `AsyncError` from the Notifier. See Screen
Implementation Pattern → Screens with text input.
### 4. Image Loading

**Never `Image.network()`.** Always `cached_network_image`, for offline caching and a loading placeholder instead of a blank flash.

```dart
CachedNetworkImage(
  imageUrl: "https://example.com/image.jpg",
  placeholder: (context, url) => const AppLoadingIndicator(),
  errorWidget: (context, url, error) => const Icon(Icons.error),
)
```

### 5. Loading Indicator

**File:** `/lib/core/common/widgets/[FILENAME].dart` — replace with the actual file once it's in place.

A custom loading widget, used project-wide. `CircularProgressIndicator()` is never called directly anywhere else in the codebase — every loading state, in a `.when()`, a button, an image placeholder, or anywhere else, goes through this one widget instead.

```dart
AppLoadingIndicator()

// with an optional size and color override, if the implementation supports it
AppLoadingIndicator(size: 32, color: AppColors.primary)
```

**Why this matters enough to be a rule, not a preference:** a raw `CircularProgressIndicator` anywhere in the codebase is a brand inconsistency waiting to be found in review — one screen with the custom loading animation, another with Flutter's stock spinner, purely because whoever wrote that screen forgot. One widget, one import, no exceptions, closes that off entirely.

### 6. Asset Paths

**File:** `/lib/core/utils/constants/asset_paths.dart`

```dart
class AssetPaths {
  AssetPaths._();
  static const _icons = 'assets/icons';
  static const cartIcon = '$_icons/cart.svg';
}
```
`Image.asset(AssetPaths.cartIcon)` — misspell the constant name, the compiler catches it. Misspell a raw string, nothing does.

---

## Connectivity

**File:** `/lib/core/utils/connectivity_service.dart`

The provider only reports the fact; reacting to it (a snackbar, say) is a separate concern:

```dart
@riverpod
Stream<bool> isOnline(Ref ref) {
  return Connectivity().onConnectivityChanged.map(
    (results) => !results.contains(ConnectivityResult.none),
  );
}
```

The snackbar moves to wherever the app root lives, using `ref.listen` — watch for state, listen for a one-time reaction:
```dart
ref.listen(isOnlineProvider, (previous, next) {
  if (next.valueOrNull == false) {
    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(content: Text('No internet connection')),
    );
  }
});
```

---

## Data Layer — Model, Repository, Mock and Real Impl

The first four steps of the Ten-File Formula, worked end-to-end. Every feature repeats this shape; `example` stands in for whichever feature is being built. This is what `exampleRepositoryProvider` — used by `ExampleNotifier`, just below — actually resolves to.

### Freezed 3 model — the `abstract class` requirement

Freezed 3 changed the class declaration from Freezed 2's plain `class` to `abstract class ... with _$ClassName`. This is a hard requirement, not a style choice — omitting `abstract` is a compile error under Freezed 3, not a lint warning.

```dart
// lib/features/example/data/models/example_model.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'example_model.freezed.dart';
part 'example_model.g.dart';

@freezed
abstract class ExampleModel with _$ExampleModel {
  const factory ExampleModel({
    required String id,
    required String title,
    @JsonKey(name: 'created_at') required DateTime createdAt,
  }) = _ExampleModel;

  factory ExampleModel.fromJson(Map<String, dynamic> json) =>
      _$ExampleModelFromJson(json);
}
```

`@JsonKey(name: 'created_at')` appears only on `createdAt` — the one field where the backend's snake_case differs from the Dart field name, per the `@JsonKey` convention already set in `CLAUDE.md`. `id` and `title` need nothing extra.

### Repository contract

An abstract interface, no implementation. Everything downstream — the provider, the notifier — types against this, never against a concrete class.

```dart
// lib/features/example/data/repositories/example_repository.dart
abstract class ExampleRepository {
  Future<List<ExampleModel>> fetchAll();
}
```

### Mock impl

Built first, always. Nothing here touches the network — it's what lets the Notifier, Widget, and Page steps get built and look correct before a backend endpoint exists.

```dart
// lib/features/example/data/repositories/example_repository_mock_impl.dart
class ExampleRepositoryMockImpl implements ExampleRepository {
  @override
  Future<List<ExampleModel>> fetchAll() async {
    await Future.delayed(const Duration(milliseconds: 400)); // simulate latency
    return [
      ExampleModel(id: '1', title: 'Mock Item', createdAt: DateTime.now()),
    ];
  }
}
```

### Real impl

Built later, once the endpoint in `docs/api-contract.md` is live. Routes through the `executeApiCall` / `executeApiCallForList` helpers from the Network Layer section below — no feature reimplements its own error handling.

```dart
// lib/features/example/data/repositories/example_repository_impl.dart
class ExampleRepositoryImpl implements ExampleRepository {
  final Dio _dio;
  const ExampleRepositoryImpl(this._dio);

  @override
  Future<List<ExampleModel>> fetchAll() {
    return executeApiCallForList(
      apiCall: () => _dio.get('/examples'),
      fromJson: ExampleModel.fromJson,
    );
  }
}
```

### Provider — and the swap

Return type is always the abstract contract; the returned value is one specific implementation. The swap is this one line, changed in one place, with no ripple into the Notifier, Widget, or Page above it — they only ever depend on `ExampleRepository`, never on which impl backs it.

```dart
// lib/features/example/data/repositories/example_repository_provider.dart
@Riverpod(keepAlive: true)
ExampleRepository exampleRepository(Ref ref) {
  return ExampleRepositoryImpl(ref.watch(dioProvider));
  // Before the endpoint exists: return ExampleRepositoryMockImpl();
}
```

---

## State Management with Riverpod

Every feature follows the same ten-step shape, already decided, documented here as the fixed reference:

> Model → Repository contract → Mock impl → Provider → Notifier → Widget → Page
> → Router → Real impl → the swap (one line)

Mock before real, always. Return type on every repository provider is the abstract contract; the returned value is one specific implementation. Full worked example with real code: the Data Layer section directly above. The Notifier below is the next step — the piece that sits on top of `exampleRepositoryProvider`.

```dart
@riverpod
class ExampleNotifier extends _$ExampleNotifier {
  @override
  Future<List<ExampleModel>> build() {
    return ref.watch(exampleRepositoryProvider).fetchAll();
  }
}
```

`build()` returning a `Future` is what makes loading, data, and error one `AsyncValue` instead of three separately-managed fields — handled by `.when()` in the widget.
### Notifiers that write

`build()` covers loading. For a user action that changes state — submit, delete,
toggle — add a method that assigns to `state`.

```dart
@riverpod
class LoginNotifier extends _$LoginNotifier {
  @override
  FutureOr<void> build() {
    // No initial load. State starts as AsyncData(null).
  }

  Future<void> submit({
    required String email,
    required String password,
  }) async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(
      () => ref.read(authRepositoryProvider).login(email, password),
    );
  }
}
```

`AsyncValue.guard` replaces try/catch/finally — it catches the throw and puts it
in `state` as `AsyncError`. Assigning to `state` *is* the notification.

**Never write `ChangeNotifier`, `ValueNotifier`, `ListenableBuilder`, a
`ViewModel` class, or a manual `bool _isLoading` in this project.** Loading,
data and error are three cases of one `AsyncValue`. This holds even when an
installed agent skill demonstrates otherwise.

Reading the result in the widget — `ref.listen` for one-off effects, `ref.watch`
for rebuilds. Never navigate from inside `build()`.

```dart
ref.listen(loginNotifierProvider, (previous, next) {
  next.whenOrNull(
    data: (_) => context.go('/home'),
    error: (e, _) => ScaffoldMessenger.of(context)
        .showSnackBar(SnackBar(content: Text(e.toString()))),
  );
});
```

---

## Screen Implementation Pattern

### 1. Screen Structure Template

Same instinct as before — extract sections into named methods, keep `build()` short. One line disappears (`Get.put()`), nothing replaces it — Riverpod has no registration step.

```dart
class ScreenName extends ConsumerWidget {
  const ScreenName({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return CommonBackgroundScaffold(
      body: SingleChildScrollView(
        child: Padding(
          padding: EdgeInsets.symmetric(horizontal: 16.w),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              _buildSection1(ref),
              _buildSection2(ref),
            ],
          ),
        ),
      ),
    );
  }

  Widget _buildSection1(WidgetRef ref) {
    return Container(
      margin: EdgeInsets.only(bottom: 24.h),
      child: Column(children: [ /* content */ ]),
    );
  }
  ### 2. UI Spacing Guidelines — unchanged
  ### 2. Screens with text input

A `ConsumerWidget` is stateless and cannot own a `TextEditingController` —
there is nowhere to create it and nowhere to `dispose()` it. Any screen with a
text field uses `ConsumerStatefulWidget` instead. This is the only approved
reason to deviate from the template above.

```dart
class LoginPage extends ConsumerStatefulWidget {
  const LoginPage({super.key});

  @override
  ConsumerState<LoginPage> createState() => _LoginPageState();
}

class _LoginPageState extends ConsumerState<LoginPage> {
  final _formKey = GlobalKey<FormState>();
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();

  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    super.dispose();
  }

  void _submit() {
    if (!_formKey.currentState!.validate()) return;
    ref.read(loginNotifierProvider.notifier).submit(
          email: _emailController.text.trim(),
          password: _passwordController.text,
        );
  }

  @override
  Widget build(BuildContext context) {
    // `ref` is available directly on ConsumerState — no WidgetRef parameter.
    ...
  }
}
```

**Controllers never go in a Notifier.** A controller has a widget lifecycle and
needs `dispose()`; a Notifier has no such hook. The Notifier receives plain
values as method arguments and never reads a controller.

Form controllers and validation are part of the **Page** step of the Ten-File
Formula, not the Notifier step.
}
```

### 2. UI Spacing Guidelines — unchanged

```dart
// ALWAYS .verticalSpace / .horizontalSpace, never a raw SizedBox for spacing
16.verticalSpace
20.horizontalSpace
EdgeInsets.symmetric(horizontal: 16.w)
```

### 3. Reactive UI

```dart
Consumer(
  builder: (context, ref, _) {
    final loginState = ref.watch(loginNotifierProvider);
    return loginState.isLoading
        ? const AppLoadingIndicator()
        : CustomFilledButton(
            text: 'Log In',
            onPressed: () => ref.read(loginNotifierProvider.notifier).submit(),
          );
  },
)
```
`.isLoading` here is `AsyncValue`'s own built-in getter, not a hand-rolled boolean field — this is the same `AsyncValue` from State Management above, just read with a shorthand for a single check. For anything beyond that — actually handling the error case, not just falling through to it — `.when()` is the safer default, the same one used everywhere else in this document.

Most often the whole screen is already a `ConsumerWidget`, so this is just `ref.watch(...)` directly inside `_buildSection1` — no extra wrapper needed.

---

## Routing with go_router

Routing uses `go_router`, wired as a Riverpod provider so guards can read app state.

**File:** `/lib/core/router/app_router.dart`

```dart
@riverpod
GoRouter appRouter(Ref ref) {
  return GoRouter(
    initialLocation: '/login',
    redirect: (context, state) => _guard(ref, state),
    routes: [
      GoRoute(
        path: '/login',
        name: 'login',
        builder: (context, state) => const LoginPage(),
      ),
      GoRoute(
        path: '/users',
        name: 'users',
        builder: (context, state) => const UserListPage(),
      ),
      GoRoute(
        path: '/users/:id',
        name: 'user-detail',
        builder: (context, state) {
          final id = state.pathParameters['id']!;
          return UserDetailPage(userId: id);
        },
      ),
    ],
  );
}

String? _guard(Ref ref, GoRouterState state) {
  final isLoggedIn = ref.read(authStateProvider).valueOrNull != null;
  final isLoggingIn = state.matchedLocation == '/login';
  if (!isLoggedIn && !isLoggingIn) return '/login';
  if (isLoggedIn && isLoggingIn) return '/users';
  return null;
}
```

**Navigation calls:**

| Action | Call |
|---|---|
| Push a new route (back returns here) | `context.push('/users')` |
| Replace current location (no back) | `context.go('/users')` |
| Clear stack, go to login | `context.go('/login')` |
| Go back | `context.pop()` |
| Pass data to the next screen | `context.push('/users', extra: user)` — read via `state.extra` |

**The difference that matters:** `push` adds to the stack, `go` replaces the current location entirely. Picking the wrong one is the most common go_router mistake — a login screen still reachable via back button after a successful login is almost always a `push` where `go` was needed.

Adding a route is one `GoRoute` entry in the list — no separate `GetPage` registration step, no route name string duplicated in two places.

### The Complete `app.dart`

Every piece from this whole document, wired together once — the actual file, not fragments. This is the one to copy.

```dart
// lib/app.dart
import 'package:flutter/material.dart';
import 'package:flutter_localizations/flutter_localizations.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'core/common/styles/app_theme.dart';
import 'core/common/widgets/unsupported_device_screen.dart';
import 'core/providers/theme_mode_provider.dart';
import 'core/router/app_router.dart';
import 'core/utils/sizer.dart';
import 'l10n/app_localizations.dart';

class MyApp extends ConsumerWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(appRouterProvider);
    final themeMode = ref.watch(themeModeNotifierProvider);

    return MaterialApp.router(
      routerConfig: router,
      theme: AppTheme.light,
      darkTheme: AppTheme.dark,
      themeMode: themeMode,
      localizationsDelegates: AppLocalizations.localizationsDelegates,
      supportedLocales: AppLocalizations.supportedLocales,
      builder: (context, child) {
        Sizer.init(context);
        // TODO: remove this gate once real tablet/large-screen layouts exist.
        final isLargeScreen = MediaQuery.sizeOf(context).width >= 600;
        return isLargeScreen ? const UnsupportedDeviceScreen() : child!;
      },
    );
  }
}
```

Every earlier mention of `app.dart` in this document — Sizer, the large-screen gate, theme switching, localization — is a piece of this one file, described in context where it was introduced. This is where they actually meet.

---

## Network Layer

Five files, shared by every feature — none of them know any single feature's name.

```dart
// lib/application/exceptions/network_exceptions.dart
class NetworkException implements Exception {
  final String message;
  final int? statusCode;
  const NetworkException(this.message, {this.statusCode});
  @override
  String toString() => message;
}
```

```dart
// lib/core/network/error_mapper.dart
extension DioErrorMapper on DioException {
  NetworkException toNetworkException() {
    switch (type) {
      case DioExceptionType.connectionTimeout:
      case DioExceptionType.sendTimeout:
      case DioExceptionType.receiveTimeout:
        return const NetworkException('The connection timed out. Please try again.');
      case DioExceptionType.connectionError:
        return const NetworkException('No internet connection.');
      case DioExceptionType.badResponse:
        return NetworkException('Something went wrong.', statusCode: response?.statusCode);
      default:
        return const NetworkException('Something went wrong. Please try again.');
    }
  }
}
```

```dart
// lib/core/network/network_hofs.dart
Future<T> executeApiCall<T>({
  required Future<Response> Function() apiCall,
  required T Function(Map<String, dynamic> json) fromJson,
}) async {
  try {
    final response = await apiCall();
    final data = response.data;
    if (data is! Map<String, dynamic>) {
      throw const NetworkException('Unexpected response format from the server.');
    }
    return fromJson(data);
  } on DioException catch (e) {
    throw e.toNetworkException();
  } on NetworkException {
    rethrow;
  } catch (_) {
    throw const NetworkException('We could not read the server response.');
  }
}
```
A sibling, `executeApiCallForList`, does the same for an endpoint returning an array.

```dart
// lib/core/network/auth_interceptor.dart
class AuthInterceptor extends Interceptor {
  final Ref _ref;
  AuthInterceptor(this._ref);

  @override
  Future<void> onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
    final token = await _ref.read(authSecureStoreProvider).readAccessToken();
    if (token != null) options.headers['Authorization'] = 'Bearer $token';
    handler.next(options);
  }
}
```

```dart
// lib/core/utils/constants/app_urls.dart
class AppUrls {
  AppUrls._();
  static const baseUrl = String.fromEnvironment(
    'BASE_URL',
    defaultValue: 'https://dev-api.example.com',
  );
}
```
Flavor-driven from the start, matching the three native flavors `CLAUDE.md` already commits to (`main_dev.dart` / `main_staging.dart` / `main_prod.dart`). Each flavor's run/build command passes its own value:
```
flutter run --flavor dev -t lib/main_dev.dart --dart-define=BASE_URL=https://dev-api.example.com
```
Nothing that reads `AppUrls.baseUrl` needs to know which flavor is active — the value is baked in at build time.

```dart
// lib/core/providers/dio_provider.dart
@Riverpod(keepAlive: true)
Dio dio(Ref ref) {
  final dio = Dio(BaseOptions(baseUrl: AppUrls.baseUrl));
  dio.interceptors.add(AuthInterceptor(ref));
  return dio;
}
```

---

## Best Practices Applied

### 1. Code Generation Guidelines
- No inline comments in generated code — self-documenting names instead
- Clean implementation — no TODO or placeholder comments left in
- Every implementing class ends in `_impl.dart`

### 2. Clean Code Principles
- Single responsibility — each `_buildX()` method has one purpose
- Extract methods — complex widgets broken into smaller named ones
- Separation of concerns — `data/` never imports Flutter; `presentation/` is the only half that does
- State is Riverpod only — no ChangeNotifier, no ViewModel class, no manual `bool _isLoading`; one `AsyncValue` covers loading, data and error
- Controllers belong to the widget (`ConsumerStatefulWidget`), never to a Notifier

### 3. Performance
- `@Riverpod(keepAlive: true)` on Dio, repository providers, api providers, and the router — never on notifiers
- Watch as high as reasonably possible (usually the page), pass plain data down to leaf widgets
- `AuthInterceptor.onRequest` always calls `handler.next(options)` — skip it and every request hangs silently

### 4. Maintainability
- Consistent structure — every feature follows the same ten-step shape
- Centralized styling — typography, colors, and theme live in one place each
- Reusable components for buttons, fields, backgrounds — never rebuilt per screen
- One loading indicator, project-wide — never a raw `CircularProgressIndicator`

### 5. Responsive Design
- All proportional sizing through Sizer's `.w`/`.h`/`.sp`/`.r`
- Portrait lock is safe on the actual target device; `UnsupportedDeviceScreen` gates the rare large-screen open, honestly rather than silently
- `SingleChildScrollView` for content that can overflow
- `SafeArea` handled through `CommonBackgroundScaffold`, not per-screen

---

## Getting Started

### 1. Setup a New Screen
1. Read `docs/design-specs/<screen>.md` for the exact layout
2. Follow the Ten-File Formula, starting with the model
3. Add the route in `core/router/app_router.dart`
4. Use `CommonBackgroundScaffold` and the existing common widgets
5. Build the screen using the `_buildSection()` extraction pattern

### 2. Creating a Custom Widget
1. Place it in `core/common/widgets/` only if a second feature will need it — otherwise it stays local to the feature's own `widgets/` folder
2. Make it take its content as parameters — never hardcode text or a callback's destination inside it
3. Use Sizer for every dimension

### 3. Adding a New Feature
1. Create the feature folder under `lib/features/`
2. Ten-File Formula, in order — model through the swap
3. Add routes
4. Update `docs/build-status.md`

---

## Example Prompt for a New Screen

```
I'm working on a new screen for this Flutter app. Read
docs/riverpod-state-management-guide.md to understand our established
patterns and conventions.

I need to build a [Profile Settings Screen], following the same
architecture as the existing screens. Design spec:
docs/design-specs/profile-settings.md

Use:
- Riverpod, following the Ten-File Formula — mock first
- CommonBackgroundScaffold
- Sizer for all dimensions (.w/.h/.sp/.r, .verticalSpace/.horizontalSpace)
- getTextStyle and AppColors for all typography and color
- The _buildSection() widget-extraction pattern, not one long build()
- go_router for any navigation, never Navigator.push directly
```