# 📱 Android Development with Kotlin — Step-by-Step Learning Roadmap

> **For developers who already know Android with Java.**
> Each topic builds on the previous one. Complete them in order for the best learning experience.

---

## 🟢 PHASE 1 — Kotlin Language Basics (Foundation)
> You must know the language before touching Android concepts.

### Step 1 — Kotlin Syntax Essentials
- Variables: `val` (immutable) vs `var` (mutable)
- Data types: `String`, `Int`, `Boolean`, `Long`, `Double`
- Null safety: `?`, `!!`, `?.`, `?:`  (Elvis operator)
- String templates: `"Hello $name"`
- Functions: `fun`, default parameters, named arguments
- `when` expression (replaces Java `switch`)
- Ranges: `1..10`, `for (i in 1..10)`

### Step 2 — Kotlin OOP
- `class`, `data class`, `object`, `companion object`
- `interface` and `abstract class`
- Inheritance: `open` keyword
- `lateinit` and `lazy`
- Visibility modifiers: `private`, `protected`, `internal`, `public`

### Step 3 — Kotlin Collections & Lambdas
- `List`, `MutableList`, `Map`, `Set`
- Lambda expressions: `{ x -> x * 2 }`
- Higher-order functions: `map`, `filter`, `forEach`, `find`, `any`, `all`
- Extension functions: `fun String.myFunction() {}`
- Scope functions: `let`, `run`, `apply`, `also`, `with`

---

## 🟡 PHASE 2 — Core Android Grammar (Refresh + Update)
> You know these concepts from Java. Learn the Kotlin way and modern updates.

### Step 4 — Project Structure & Build System
- `AndroidManifest.xml` — permissions, components declaration
- `build.gradle` (app level) — dependencies, SDK versions
- `build.gradle` (project level) — classpath
- `gradle.properties` — JVM options
- `res/` folder — `layout/`, `drawable/`, `values/`, `mipmap/`
- `R` class — how resources are referenced

### Step 5 — Activity & Lifecycle (Refresher)
- `Activity` lifecycle: `onCreate`, `onStart`, `onResume`, `onPause`, `onStop`, `onDestroy`
- `onCreate(savedInstanceState: Bundle?)`
- `setContentView(R.layout.activity_main)`
- `ViewBinding` — modern replacement for `findViewById`
- Back stack and task management
- `onSaveInstanceState` / `onRestoreInstanceState`

### Step 6 — ViewBinding & DataBinding
- Enable `ViewBinding` in `build.gradle`
- `ActivityMainBinding.inflate(layoutInflater)`
- Accessing views: `binding.textView.text = "Hello"`
- `DataBinding` — two-way binding with XML layout variables
- Difference between ViewBinding and DataBinding

### Step 7 — Intents (Refresher + Updates)
- **Explicit Intent** — start specific Activity
- **Implicit Intent** — open camera, share, browser
- Passing data with `putExtra()` / `getStringExtra()`
- `startActivity()` vs `startActivityForResult()` (deprecated)
- **ActivityResultLauncher** — modern replacement for `startActivityForResult`
- `Intent Flags`: `FLAG_ACTIVITY_CLEAR_TOP`, `FLAG_ACTIVITY_NEW_TASK`, `FLAG_ACTIVITY_SINGLE_TOP`
- **PendingIntent** — used with Notifications and Widgets
- **Deep Links** — open app screens via URL

### Step 8 — Fragments & Fragment Lifecycle
- Fragment lifecycle: `onCreateView`, `onViewCreated`, `onDestroyView`
- `FragmentManager` and `FragmentTransaction`
- Fragment `BackStack`
- Communicating between Fragment and Activity
- Fragment arguments: `Bundle` and `arguments`
- `BottomSheetDialogFragment`, `DialogFragment`

### Step 9 — RecyclerView (Replaces ListView)
- `RecyclerView` vs old `ListView`
- `RecyclerView.Adapter` — `onCreateViewHolder`, `onBindViewHolder`, `getItemCount`
- `ViewHolder` pattern
- `LayoutManager`: `LinearLayoutManager`, `GridLayoutManager`, `StaggeredGridLayoutManager`
- `ListAdapter` + `DiffUtil` — efficient list updates
- Item click handling
- `ItemDecoration` — dividers and spacing
- `ItemAnimator`

### Step 10 — UI Components & Layouts
- `ConstraintLayout` — flat, performant layouts
- `LinearLayout`, `FrameLayout`, `RelativeLayout` (when to use each)
- `MaterialDesign` components: `MaterialButton`, `TextInputLayout`, `Snackbar`, `Chip`
- `BottomNavigationView`
- `DrawerLayout` + `NavigationView`
- `TabLayout` + `ViewPager2`
- `CoordinatorLayout` + `AppBarLayout` + `CollapsingToolbarLayout`
- `SwipeRefreshLayout`

---

## 🟠 PHASE 3 — Modern Android Architecture
> This is the most important phase. Learn this before networking or databases.

### Step 11 — Coroutines (Replaces AsyncTask)
- What is a Coroutine? `suspend` functions
- `CoroutineScope`, `GlobalScope`, `lifecycleScope`, `viewModelScope`
- Dispatchers: `Main`, `IO`, `Default`
- `launch` vs `async` / `await`
- `withContext` — switching threads
- `Flow` — cold stream of data
- `StateFlow` and `SharedFlow` — hot streams
- `try/catch` in coroutines
- Coroutine cancellation and `Job`

### Step 12 — ViewModel
- Why ViewModel? Survives screen rotation
- `ViewModel` class — `viewModelScope`
- `AndroidViewModel` — when you need `Application` context
- `ViewModelProvider` / `by viewModels()` delegate
- Sharing ViewModel between Fragments: `by activityViewModels()`

### Step 13 — LiveData & StateFlow
- `LiveData` — lifecycle-aware observable
- `MutableLiveData` — `setValue()` vs `postValue()`
- `observe()` in Activity / Fragment
- `MediatorLiveData` — combining multiple LiveData
- `StateFlow` — modern Kotlin alternative to LiveData
- `collectAsState()` usage
- When to use `LiveData` vs `StateFlow`

### Step 14 — Repository Pattern
- Why Repository? Separation of concerns
- Repository sits between ViewModel and data sources
- Data sources: Remote API, Local Database, Cache
- Single source of truth principle
- Repository with coroutines
- Offline-first architecture

### Step 15 — MVVM Architecture Pattern
- **Model** — data classes, repository
- **View** — Activity, Fragment (only UI logic)
- **ViewModel** — business logic, holds UI state
- How all three communicate
- Why MVVM is Google's recommended pattern
- `UiState` data class pattern

---

## 🔵 PHASE 4 — Data & Networking

### Step 16 — Room Database (Replaces SQLiteOpenHelper)
- `@Entity` — defines a table
- `@PrimaryKey`, `@ColumnInfo`
- `@Dao` — Data Access Object with queries
- `@Insert`, `@Update`, `@Delete`, `@Query`
- `@Database` — database class with version
- `RoomDatabase.Builder`
- Migrations: `Migration` class
- Room + Coroutines (`suspend` functions in DAO)
- Room + Flow — reactive database queries
- Type converters: `@TypeConverter`

### Step 17 — DataStore (Replaces SharedPreferences)
- `SharedPreferences` is deprecated for complex use cases
- **Preferences DataStore** — key-value storage (replaces SharedPreferences)
- **Proto DataStore** — typed object storage
- `DataStore<Preferences>` with `preferencesDataStore`
- Reading data: `dataStore.data.map { }`
- Writing data: `dataStore.edit { }`
- DataStore + Flow + ViewModel

### Step 18 — Retrofit + OkHttp (Networking)
- `Retrofit` setup and `@GET`, `@POST`, `@PUT`, `@DELETE`
- `@Path`, `@Query`, `@Body`, `@Header`
- `OkHttp` interceptors — logging, auth headers
- JSON parsing with `Gson` / `Moshi` / `Kotlinx Serialization`
- `Response<T>` — handling success and error
- Retrofit + Coroutines (`suspend` functions)
- Error handling: `try/catch`, `Result` wrapper
- API service interface pattern

---

## 🟣 PHASE 5 — Jetpack Components

### Step 19 — Jetpack Navigation Component
- Navigation graph (`nav_graph.xml`)
- `NavController` — navigate between destinations
- `NavHostFragment` — container for fragments
- `navigate()` with actions
- Passing arguments with `Safe Args` plugin
- **Deep Links** — linking URLs to destinations
- Back stack management with Navigation
- Bottom navigation + Navigation component
- Nested navigation graphs

### Step 20 — Notifications (Major Updates)
- **Notification Channels** — required since Android 8.0 (API 26)
- `NotificationManager` vs `NotificationManagerCompat`
- `NotificationCompat.Builder`
- Notification styles: `BigTextStyle`, `InboxStyle`, `BigPictureStyle`
- Notification actions with `PendingIntent`
- **Foreground Service Notification** — required for long-running services
- Notification permission (`POST_NOTIFICATIONS`) — required since Android 13
- Notification groups and bundled notifications

### Step 21 — Services (Updated)
- `Service` vs `IntentService` (deprecated)
- **Foreground Service** — user-visible background work (music player, location)
- **Bound Service** — client-server interface with `IBinder`
- `startService()` vs `bindService()`
- `startForeground()` with notification
- Service limitations in modern Android (background restrictions)

### Step 22 — BroadcastReceiver (Updated)
- Static receiver — declared in `AndroidManifest.xml`
- Dynamic receiver — registered in code with `registerReceiver()`
- System broadcasts: `BOOT_COMPLETED`, `CONNECTIVITY_CHANGE`, `BATTERY_LOW`
- Ordered broadcasts
- `LocalBroadcastManager` — deprecated, use `Flow` or `EventBus` instead
- Background broadcast restrictions (Android 8+)
- `PendingIntent` with broadcasts

### Step 23 — WorkManager (Replaces AlarmManager / JobScheduler)
- When to use WorkManager (guaranteed background work)
- `Worker` class — `doWork()` override
- `CoroutineWorker` — with coroutine support
- `WorkRequest`: `OneTimeWorkRequest` vs `PeriodicWorkRequest`
- `Constraints` — network, charging, storage
- Chaining work: `then()`, `combine()`
- Observing work status with `WorkInfo`
- Input data and output data with `Data`
- Retry and backoff policies

---

## ⚪ PHASE 6 — Dependency Injection

### Step 24 — Hilt (Replaces Dagger)
- What is Dependency Injection (DI)?
- Why Hilt? (Simplified Dagger for Android)
- `@HiltAndroidApp` — application class
- `@AndroidEntryPoint` — inject into Activity/Fragment
- `@Inject constructor` — injectable class
- `@Module` and `@InstallIn`
- `@Provides` — provide third-party dependencies
- `@Binds` — interface bindings
- Scopes: `@Singleton`, `@ActivityScoped`, `@ViewModelScoped`
- Hilt + ViewModel: `@HiltViewModel`
- Hilt + Room + Retrofit together

---

## 🔴 PHASE 7 — Permissions & Security

### Step 25 — Runtime Permissions (Updated)
- Normal vs Dangerous permissions
- `ActivityResultLauncher` for permission requests
- `requestPermissions()` flow
- `shouldShowRequestPermissionRationale()`
- Handling permission denial
- **New restricted permissions** (Android 12+, 13+):
  - `MANAGE_EXACT_ALARM`
  - `NEARBY_WIFI_DEVICES`
  - `READ_MEDIA_IMAGES`, `READ_MEDIA_VIDEO`
  - `POST_NOTIFICATIONS`
- **Photo Picker API** — replaces `READ_EXTERNAL_STORAGE` for media

---

## 🌟 PHASE 8 — Jetpack Compose (Modern UI)
> This is a separate UI paradigm. Learn it after mastering Phases 1–7.

### Step 26 — Compose Fundamentals
- What is Jetpack Compose? (Declarative UI)
- `@Composable` functions
- `setContent {}` — replaces `setContentView()`
- State in Compose: `remember`, `mutableStateOf`
- Recomposition — how Compose updates UI
- `Modifier` — layout and styling

### Step 27 — Compose UI Components
- `Text`, `Button`, `TextField`, `Image`
- `Column`, `Row`, `Box` — replaces LinearLayout, FrameLayout
- `LazyColumn`, `LazyRow` — replaces RecyclerView
- `Scaffold` — app structure (TopBar, BottomBar, FAB)
- `TopAppBar`, `BottomNavigationBar`

### Step 28 — Compose State Management
- `State hoisting` pattern
- `ViewModel` + `StateFlow` with Compose
- `collectAsStateWithLifecycle()`
- Side effects: `LaunchedEffect`, `DisposableEffect`, `SideEffect`
- `rememberCoroutineScope`

### Step 29 — Compose Navigation
- `NavHost` and `NavController` in Compose
- `composable()` destinations
- Passing arguments between screens
- `BottomNavigation` with Compose Navigation
- Deep links in Compose Navigation

### Step 30 — Compose Theming & Animation
- `MaterialTheme` — colors, typography, shapes
- Dark theme in Compose
- `animateAsState`, `AnimatedVisibility`, `AnimatedContent`
- `Transition` animations
- Custom `CompositionLocal`

---

## 📦 PHASE 9 — Advanced Topics

### Step 31 — Paging 3 (Large Lists)
- `PagingSource` — load pages of data
- `PagingData` and `PagingDataAdapter`
- Remote mediator — network + database paging
- `LoadState` — loading, error, empty states

### Step 32 — App Widgets
- `AppWidgetProvider` — BroadcastReceiver subclass
- `RemoteViews` — limited UI for widgets
- Widget configuration Activity
- Updating widgets with `AppWidgetManager`
- Glance (Compose-based widgets)

### Step 33 — Content Providers & File Access
- `ContentProvider` — share data between apps
- `FileProvider` — share files safely
- `MediaStore` API — access photos, videos, audio
- `SAF` (Storage Access Framework) — document picker
- `BlobStoreManager` — sharing large data

### Step 34 — Testing
- Unit tests with `JUnit` + `Mockito` / `MockK`
- `ViewModel` unit tests
- `Room` database tests
- `Espresso` — UI instrumentation tests
- `Compose` UI testing with `ComposeTestRule`
- `Hilt` test components

---

## 🏁 Suggested Learning Schedule

| Week | Phase | Topics |
|------|-------|--------|
| Week 1–2 | Phase 1 | Steps 1–3 (Kotlin Language) |
| Week 3–4 | Phase 2 | Steps 4–7 (Core Android Refresh) |
| Week 5–6 | Phase 2 | Steps 8–10 (Fragments, RecyclerView, UI) |
| Week 7–8 | Phase 3 | Steps 11–13 (Coroutines, ViewModel, LiveData) |
| Week 9 | Phase 3 | Steps 14–15 (Repository, MVVM) |
| Week 10–11 | Phase 4 | Steps 16–18 (Room, DataStore, Retrofit) |
| Week 12 | Phase 5 | Steps 19–20 (Navigation, Notifications) |
| Week 13 | Phase 5 | Steps 21–23 (Services, BroadcastReceiver, WorkManager) |
| Week 14 | Phase 6 | Step 24 (Hilt) |
| Week 15 | Phase 7 | Step 25 (Permissions) |
| Week 16–20 | Phase 8 | Steps 26–30 (Jetpack Compose) |
| Week 21+ | Phase 9 | Steps 31–34 (Advanced Topics) |

---

## ⚡ Key Things That Changed Since Your Java Days

| Old (Java Era) | New (Kotlin Era) |
|----------------|-----------------|
| `AsyncTask` | Coroutines |
| `ListView` | `RecyclerView` + `ListAdapter` |
| `SQLiteOpenHelper` | `Room` |
| `SharedPreferences` | `DataStore` |
| `startActivityForResult` | `ActivityResultLauncher` |
| `AlarmManager` / `JobScheduler` | `WorkManager` |
| `Dagger` | `Hilt` |
| XML UI only | `Jetpack Compose` |
| `LocalBroadcastManager` | `Flow` / `SharedFlow` |
| `findViewById` | `ViewBinding` / `DataBinding` |
| Manual `onSaveInstanceState` | `ViewModel` |

---

> 💡 **Tip:** After completing Steps 1–25, you will be able to build any production Android app.
> Jetpack Compose (Phase 8) is the future of Android UI — prioritize it after you're comfortable with the basics.