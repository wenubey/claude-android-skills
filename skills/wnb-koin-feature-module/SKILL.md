---
name: wnb-koin-feature-module
description: Use this skill when adding a new feature to an Android + Koin project, or when reviewing dependency injection wiring. Enforces feature-scoped Koin modules — one `Module` per feature package (customerModule, sellerModule, authModule, …) that bundles the feature's ViewModels and feature-only bindings, alongside concern-scoped modules (databaseModule, ktorModule, dispatcherModule, firebaseModule) for cross-cutting infrastructure. Requires `viewModelOf(::XxxViewModel)` for simple constructors, `viewModel { … }` for manual wiring, `single` vs `factory` semantics, `named(...)` qualifier for parallel bindings of the same type, all modules merged into a single `appModules` list, `startKoin { modules(appModules) }` only in `Application.onCreate`. Triggers on "add koin module", "koin binding", "viewModelOf", "koin module", "single vs factory", "named qualifier", "startKoin", "loadKoinModules", "feature module", "DI wiring", "inject viewmodel".
---

# Koin — feature-scoped module structure

This skill covers **how to organize Koin modules in a growing Android codebase**. The core rule: as an app scales, a single monolithic `viewModelModule` listing every ViewModel becomes a merge-conflict magnet and a review-friction generator. Split by feature; keep infrastructure by concern.

Pairs with `[[wnb-viewmodel-udf]]` — the VM shape the module binds.

## Non-negotiables

1. **Feature-scoped modules for feature code.** One `Module` per feature package (`customerModule`, `sellerModule`, `authModule`, `adminModule`, …). Each module bundles that feature's ViewModels *and* any bindings only that feature uses.
2. **Concern-scoped modules for cross-cutting infrastructure.** `databaseModule`, `ktorModule` / `firebaseModule`, `dispatcherModule`, `preferencesModule`, `connectivityModule`, `workerModule`. These stay concern-scoped because they are consumed by every feature.
3. **`viewModelOf(::XxxViewModel)` when the constructor is Koin-injectable end-to-end.** Only fall back to `viewModel { XxxViewModel(get(), get(named("foo")), get()) }` when you need qualifiers, `SavedStateHandle`, or manual argument massaging.
4. **`single` vs `factory` vs `viewModel`:**
   - `single { }` — one instance per Koin scope. Use for repositories, DAOs, HTTP clients, dispatcher providers.
   - `factory { }` — new instance every `get()`. Use for lightweight helpers you don't want to leak state across.
   - `viewModel { }` / `viewModelOf(...)` — one instance per `ViewModelStoreOwner`. Never `single` a ViewModel.
5. **`named("...")` qualifier when two bindings share a type.** Example: two `DataStore<Preferences>` instances (`named("pendingSync")`, `named("session")`). Consumers must use the same qualifier at `get(named("..."))`.
6. **All modules merged into a single `appModules: List<Module>`.** One import point.
7. **`startKoin { modules(appModules) }` only in `Application.onCreate()`.** Never anywhere else. Tests use `loadKoinModules` / `unloadKoinModules` inside a `KoinTestRule`, not `startKoin`.
8. **No circular module dependencies.** Feature modules depend on infrastructure modules; infrastructure never depends on features. If a "cross-feature" binding is needed, promote it to a concern module.

## Canonical shape

```kotlin
// di/CustomerModule.kt — feature-scoped
val customerModule = module {
    // ViewModels for this feature
    viewModelOf(::CustomerHomeViewModel)
    viewModelOf(::CustomerProductDetailViewModel)
    viewModelOf(::CartViewModel)
    viewModelOf(::WishlistViewModel)

    viewModel {
        CheckoutViewModel(
            paymentRepository = get(),
            cartRepository = get(),
            addressRepository = get(),
            authRepository = get(),
            connectivityObserver = get(),
            discountRepository = get(),
            dispatcherProvider = get(),
        )
    }

    // Feature-only helper — not used outside customer package
    single { CustomerPricingFormatter(get()) }
}

// di/DataModule.kt — concern-scoped: repositories are consumed by every feature
val repositoryModule = module {
    single<AuthRepository>       { AuthRepositoryImpl(get(), get(), get()) }
    single<CartRepository>       { CartRepositoryImpl(get(), get()) }
    single<AddressRepository>    { AddressRepositoryImpl(get(), get()) }
    single<PaymentRepository>    { PaymentRepositoryImpl(get(), get(), get()) }
    single<DiscountRepository>   { DiscountRepositoryImpl(get(), get()) }
}

// di/DispatcherModule.kt — the injectable IO/Main/Default abstraction
val dispatcherModule = module {
    single<DispatcherProvider> { DefaultDispatcherProvider() }
}

// di/PreferencesModule.kt — named qualifiers for parallel DataStore bindings
val preferencesModule = module {
    single(named("session")) {
        PreferenceDataStoreFactory.create { get<Context>().preferencesDataStoreFile("session") }
    }
    single(named("pendingSync")) {
        PreferenceDataStoreFactory.create { get<Context>().preferencesDataStoreFile("pending_sync") }
    }
}

// di/AppModules.kt — single composition point
val appModules = listOf(
    // Infrastructure (concern-scoped)
    firebaseModule,
    databaseModule,
    dispatcherModule,
    preferencesModule,
    connectivityModule,
    workerModule,
    ktorModule,
    notificationModule,
    repositoryModule,
    // Features (feature-scoped)
    authModule,
    customerModule,
    sellerModule,
    adminModule,
)

// App.kt — the only place startKoin appears
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidLogger()
            androidContext(this@App)
            modules(appModules)
        }
    }
}
```

## Common mistakes

- **A single `viewModelModule` that lists every ViewModel in the app** — canonical merge-conflict magnet, and impossible to see at a glance "what does this feature own?". Split by feature.
- **A single `dataModule` that binds every repository and every DAO** — same issue at a smaller scale. Split at least by layer (`repositoryModule`, `databaseModule`, `networkModule`).
- **`single<XxxViewModel>()`** — a ViewModel is never a singleton. Use `viewModel` / `viewModelOf`. `single` will outlive the screen and leak.
- **Two `DataStore<Preferences>` bindings without `named(...)`** — Koin can't disambiguate; you get a runtime `NoBeanDefFoundException` or, worse, the wrong instance. Always qualify parallel bindings.
- **Feature module importing another feature module** — cross-feature coupling in the DI graph. Promote the shared binding to a concern module, or split it out (`sharedCommerceModule`).
- **`get()` inside a Composable body** — Koin lookups are runtime; use `koinInject<T>()` / `koinViewModel<T>()` from `koin-androidx-compose`. Or hoist the injection to the ViewModel and pass state down.
- **`startKoin` in a test** — collides with the running `App`'s Koin. Use `KoinTestRule` + `modules(testAuthModule, testDataModule)` where the test modules override the real ones.
- **Forgetting to append a new module to `appModules`** — the app compiles, then throws at runtime the first time the missing binding is requested. Add the module to `appModules` in the same commit.
- **Circular dependency** (`customerModule` depends on `sellerModule` binding X; `sellerModule` depends on `customerModule` binding Y). Koin fails at graph construction. Refactor X and Y into a shared concern module.
- **Injecting a `DispatcherProvider` on every consumer but binding `Dispatchers.IO` directly somewhere** — split brain. One canonical `DispatcherProvider` binding lives in `dispatcherModule`.

## Testing the DI graph

- **Static verification:** call `verify()` on each feature module in a JVM unit test. Fails fast if a binding is missing.
- **Overriding for tests:** `KoinTestRule` + `modules(testXxxModule)` where the test module rebinds specific `single<XxxRepository> { FakeXxxRepository() }`. Prefer overriding at the repository layer, not the ViewModel layer — ViewModels should be constructed directly in tests (see `[[wnb-viewmodel-test]]`), not resolved through Koin.
- **Dynamic loading:** `loadKoinModules(testAuthModule)` and `unloadKoinModules(testAuthModule)` in `@Before` / `@After` when the test needs to swap a binding mid-suite.

## Related skills

- `[[wnb-viewmodel-udf]]` — the ViewModel shape these modules bind.
- `[[wnb-viewmodel-test]]` — why VMs are constructed directly in tests, not resolved through Koin.
