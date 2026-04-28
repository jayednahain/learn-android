# Flutter → Android (Kotlin + Compose) Cheat Sheet

> You already know Flutter. This maps every Flutter/Dart concept to its Android (Kotlin + Jetpack Compose) equivalent.
> Think of this as a **translation dictionary** — same ideas, different syntax.

---

## 🧠 The Big Mental Model Shift

| Flutter | Android (Kotlin + Compose) |
|---|---|
| Dart | Kotlin |
| `Widget` | `@Composable` function |
| `StatelessWidget` | `@Composable` function (no state) |
| `StatefulWidget` + `State<T>` | `@Composable` + `remember { mutableStateOf() }` |
| `pub.dev` + `pubspec.yaml` | Maven + `build.gradle` |
| `flutter pub get` | Gradle sync |
| `flutter run` | Run button in Android Studio |
| Hot reload (r) | Live Edit / Apply Changes in Android Studio |
| `main.dart` entry point | `MainActivity.kt` + `setContent {}` |
| `MaterialApp` | `MaterialTheme` wrapping `setContent {}` |
| `Scaffold` | `Scaffold()` composable (same concept!) |

> 💡 Flutter and Compose are the **closest pair** in this comparison. Both are declarative, both rebuild UI when state changes, both use a widget/composable tree. The concepts are nearly identical — only the syntax differs.

---

## 1. 🏗️ Project Entry Point

### Flutter
```dart
// main.dart
void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'My App',
      theme: ThemeData(colorSchemeSeed: Colors.blue),
      home: const HomeScreen(),
    );
  }
}
```

### Android (Kotlin + Compose)
```kotlin
// MainActivity.kt
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {                          // replaces runApp()
            MyAppTheme {                      // replaces MaterialApp + theme
                AppNavigation()              // replaces home: HomeScreen()
            }
        }
    }
}
```

| Flutter | Android |
|---|---|
| `runApp(MyApp())` | `setContent { MyAppTheme { ... } }` |
| `MaterialApp(home: HomeScreen())` | `NavHost` with start destination |
| `ThemeData(colorSchemeSeed: ...)` | `MaterialTheme(colorScheme = ...)` |
| `Widget build(BuildContext context)` | `@Composable fun MyScreen()` |

---

## 2. 🧩 Widgets → Composable Functions

### Flutter — StatelessWidget
```dart
class UserCard extends StatelessWidget {
  final String name;
  final String email;

  const UserCard({super.key, required this.name, required this.email});

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(name, style: const TextStyle(fontWeight: FontWeight.bold)),
            Text(email, style: const TextStyle(color: Colors.grey)),
          ],
        ),
      ),
    );
  }
}
```

### Android (Compose)
```kotlin
@Composable
fun UserCard(name: String, email: String) {
    Card(modifier = Modifier.fillMaxWidth()) {
        Column(
            modifier = Modifier.padding(16.dp),
            verticalArrangement = Arrangement.spacedBy(4.dp)
        ) {
            Text(text = name, fontWeight = FontWeight.Bold)
            Text(text = email, color = Color.Gray)
        }
    }
}
```

> The structure is almost **identical** — both take parameters as constructor/function args, both return a tree of UI elements. Flutter needs a class + `build()`; Compose just needs a function with `@Composable`.

---

## 3. 📦 Widgets / Components Mapping

| Flutter Widget | Compose Equivalent | Notes |
|---|---|---|
| `Text` | `Text()` | Same concept |
| `TextField` | `TextField()` / `OutlinedTextField()` | |
| `ElevatedButton` | `Button()` | |
| `TextButton` | `TextButton()` | Same name! |
| `OutlinedButton` | `OutlinedButton()` | Same name! |
| `IconButton` | `IconButton()` | Same name! |
| `FloatingActionButton` | `FloatingActionButton()` | Same name! |
| `Image.asset()` | `Image(painterResource(...))` | |
| `Image.network()` | `AsyncImage()` (Coil library) | |
| `Icon` | `Icon()` | |
| `Column` | `Column()` | Same name! |
| `Row` | `Row()` | Same name! |
| `Stack` | `Box()` | Flutter = Stack, Compose = Box |
| `Container` | `Box()` + `Modifier` | `Container` has no direct match; use Box + Modifier |
| `SizedBox(width, height)` | `Spacer(Modifier.size())` / `Modifier.width()/.height()` | |
| `SizedBox.expand()` | `Modifier.fillMaxSize()` | |
| `Padding` widget | `Modifier.padding()` | In Compose, padding is a Modifier, not a Widget |
| `Center` | `Box(contentAlignment = Alignment.Center)` | |
| `Align` | `Modifier.align()` inside a Box | |
| `Expanded` | `Modifier.weight(1f)` | |
| `Flexible` | `Modifier.weight(flex.toFloat())` | |
| `Spacer()` | `Spacer(Modifier.height/width())` | Same name! |
| `ListView` | `LazyColumn()` | |
| `ListView.builder` | `LazyColumn { items(list) { } }` | |
| `GridView.builder` | `LazyVerticalGrid()` | |
| `SingleChildScrollView` | `Column(Modifier.verticalScroll(rememberScrollState()))` | |
| `Scaffold` | `Scaffold()` | Same concept, almost same API! |
| `AppBar` | `TopAppBar()` | |
| `BottomNavigationBar` | `NavigationBar()` | |
| `Drawer` | `ModalNavigationDrawer()` | |
| `SnackBar` | `Snackbar` via `SnackbarHostState` | |
| `Dialog` | `AlertDialog()` | |
| `BottomSheet` | `ModalBottomSheet()` | |
| `CircularProgressIndicator` | `CircularProgressIndicator()` | Same name! |
| `LinearProgressIndicator` | `LinearProgressIndicator()` | Same name! |
| `Divider` | `Divider()` | Same name! |
| `Chip` | `FilterChip()` / `AssistChip()` | |
| `Switch` | `Switch()` | Same name! |
| `Checkbox` | `Checkbox()` | Same name! |
| `Slider` | `Slider()` | Same name! |
| `Card` | `Card()` | Same name! |
| `InkWell` / `GestureDetector` | `Modifier.clickable { }` | |
| `Visibility` | `if (visible) { Widget() }` | |

---

### Side-by-Side: ListView.builder vs LazyColumn

```dart
// Flutter
ListView.builder(
  itemCount: users.length,
  itemBuilder: (context, index) {
    final user = users[index];
    return UserCard(name: user.name, email: user.email);
  },
)
```

```kotlin
// Compose
LazyColumn {
    items(users, key = { it.id }) { user ->
        UserCard(name = user.name, email = user.email)
    }
}
```

> `itemBuilder` = the trailing lambda `{ user -> ... }`.  
> Both are virtualized — only visible items are rendered.

---

## 4. 🎨 Styles → Modifier

This is the **biggest syntactic difference**. Flutter uses nested widget trees for styling. Compose uses a `Modifier` chain.

### Flutter — Nested Widgets for Styling
```dart
Container(
  width: double.infinity,   // full width
  color: Colors.white,
  padding: const EdgeInsets.all(16),
  margin: const EdgeInsets.only(bottom: 8),
  decoration: BoxDecoration(
    borderRadius: BorderRadius.circular(12),
    boxShadow: [BoxShadow(blurRadius: 4, color: Colors.black26)],
  ),
  child: Text(
    'Hello',
    style: TextStyle(
      fontSize: 24,
      fontWeight: FontWeight.bold,
      color: Colors.black87,
    ),
  ),
)
```

### Android Compose — Modifier chain
```kotlin
Box(
    modifier = Modifier
        .fillMaxWidth()                          // width: double.infinity
        .background(Color.White, RoundedCornerShape(12.dp)) // color + borderRadius
        .padding(16.dp)                          // padding: EdgeInsets.all(16)
        .shadow(elevation = 4.dp, shape = RoundedCornerShape(12.dp)) // boxShadow
) {
    Text(
        text = "Hello",
        fontSize = 24.sp,                        // fontSize: 24
        fontWeight = FontWeight.Bold,            // fontWeight: FontWeight.bold
        color = Color(0xDD000000)               // color: Colors.black87
    )
}
```

### Style Quick Reference

| Flutter | Compose Modifier / Property |
|---|---|
| `double.infinity` (width/height) | `Modifier.fillMaxWidth()` / `Modifier.fillMaxSize()` |
| `EdgeInsets.all(16)` | `Modifier.padding(16.dp)` |
| `EdgeInsets.symmetric(horizontal: 16)` | `Modifier.padding(horizontal = 16.dp)` |
| `EdgeInsets.only(top: 8)` | `Modifier.padding(top = 8.dp)` |
| `BoxDecoration(color: ...)` | `Modifier.background(Color...)` |
| `BorderRadius.circular(12)` | `RoundedCornerShape(12.dp)` in `background()` or `clip()` |
| `BoxShadow(blurRadius: 4)` | `Modifier.shadow(elevation = 4.dp)` |
| `ClipRRect(borderRadius: ...)` | `Modifier.clip(RoundedCornerShape(...))` |
| `Opacity(opacity: 0.5)` | `Modifier.alpha(0.5f)` |
| `Transform.rotate(angle)` | `Modifier.rotate(degrees)` |
| `SizedBox(width: 100, height: 50)` | `Modifier.size(100.dp, 50.dp)` |
| `Expanded(flex: 2)` | `Modifier.weight(2f)` |
| `CrossAxisAlignment.center` | `horizontalAlignment = Alignment.CenterHorizontally` |
| `MainAxisAlignment.center` | `verticalArrangement = Arrangement.Center` |
| `MainAxisAlignment.spaceBetween` | `Arrangement.SpaceBetween` |
| `MainAxisAlignment.spaceEvenly` | `Arrangement.SpaceEvenly` |
| `TextStyle(fontSize: 16)` | `fontSize = 16.sp` |
| `TextStyle(fontWeight: FontWeight.bold)` | `fontWeight = FontWeight.Bold` |
| `TextStyle(color: Colors.grey)` | `color = Color.Gray` |
| `TextAlign.center` | `textAlign = TextAlign.Center` |
| `TextOverflow.ellipsis` | `overflow = TextOverflow.Ellipsis` |
| `Colors.blue` | `Color.Blue` or `Color(0xFF2196F3)` |
| `Colors.blue.shade700` | `Color(0xFF1976D2)` |

---

## 5. ⚡ State — `setState` → `remember` + `mutableStateOf`

### Flutter — StatefulWidget
```dart
class Counter extends StatefulWidget {
  const Counter({super.key});
  @override
  State<Counter> createState() => _CounterState();
}

class _CounterState extends State<Counter> {
  int count = 0;
  String name = '';

  @override
  Widget build(BuildContext context) {
    return Column(children: [
      Text('Count: $count'),
      ElevatedButton(
        onPressed: () => setState(() => count++),  // triggers rebuild
        child: const Text('Add'),
      ),
      TextField(onChanged: (v) => setState(() => name = v)),
    ]);
  }
}
```

### Android Compose
```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }      // int count = 0
    var name by remember { mutableStateOf("") }       // String name = ''

    Column {
        Text("Count: $count")
        Button(onClick = { count++ }) { Text("Add") } // setState(() => count++)
        TextField(value = name, onValueChange = { name = it }) // onChanged
    }
}
```

> `setState(() => count++)` = just `count++` in Compose. The state variable itself is reactive.  
> No `StatefulWidget` class needed — just one function.

| Flutter | Compose |
|---|---|
| `StatefulWidget` + `State<T>` class | `@Composable` function |
| `int count = 0` (in State) | `var count by remember { mutableStateOf(0) }` |
| `setState(() { count++ })` | `count++` (state is reactive automatically) |
| Rotated screen → state lost (unless `AutomaticKeepAliveClientMixin`) | `rememberSaveable` survives rotation |
| `initState()` | `LaunchedEffect(Unit) { }` |
| `dispose()` | `DisposableEffect(Unit) { onDispose { } }` |

---

## 6. 🏪 Global State — Provider/Riverpod/BLoC → ViewModel + StateFlow

### Flutter — Riverpod
```dart
// provider
final counterProvider = StateNotifierProvider<CounterNotifier, int>((ref) {
  return CounterNotifier();
});

class CounterNotifier extends StateNotifier<int> {
  CounterNotifier() : super(0);
  void increment() => state++;
  void reset() => state = 0;
}

// Widget
class CounterScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider);
    return Column(children: [
      Text('Count: $count'),
      ElevatedButton(
        onPressed: () => ref.read(counterProvider.notifier).increment(),
        child: const Text('Add'),
      ),
    ]);
  }
}
```

### Flutter — BLoC
```dart
// BLoC
class CounterBloc extends Bloc<CounterEvent, int> {
  CounterBloc() : super(0) {
    on<Increment>((event, emit) => emit(state + 1));
    on<Reset>((event, emit) => emit(0));
  }
}

// Widget
BlocBuilder<CounterBloc, int>(
  builder: (context, count) => Text('Count: $count'),
)
```

### Android — ViewModel + StateFlow
```kotlin
// ViewModel — like StateNotifier / Bloc
class CounterViewModel : ViewModel() {
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count.asStateFlow()

    fun increment() { _count.value++ }
    fun reset() { _count.value = 0 }
}

// Composable — like ConsumerWidget / BlocBuilder
@Composable
fun CounterScreen(viewModel: CounterViewModel = viewModel()) {
    val count by viewModel.count.collectAsStateWithLifecycle()

    Column {
        Text("Count: $count")
        Button(onClick = { viewModel.increment() }) { Text("Add") }
    }
}
```

### Full UiState Pattern (like BLoC State class)
```dart
// Flutter BLoC state
abstract class HomeState {}
class HomeLoading extends HomeState {}
class HomeLoaded extends HomeState { final List<Item> items; HomeLoaded(this.items); }
class HomeError extends HomeState { final String message; HomeError(this.message); }
```

```kotlin
// Android — single data class (cleaner)
data class HomeUiState(
    val items: List<Item> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)

class HomeViewModel : ViewModel() {
    private val _state = MutableStateFlow(HomeUiState())
    val state = _state.asStateFlow()

    fun loadItems() {
        _state.update { it.copy(isLoading = true) }
        viewModelScope.launch {
            val result = repository.getItems()
            _state.update { it.copy(items = result, isLoading = false) }
        }
    }
}
```

| Flutter | Android |
|---|---|
| `StateNotifier` / `Cubit` / `Bloc` | `ViewModel` |
| `StateNotifierProvider` / `BlocProvider` | `viewModel()` / `hiltViewModel()` |
| `ref.watch(counterProvider)` | `viewModel.count.collectAsStateWithLifecycle()` |
| `ref.read(provider.notifier).increment()` | `viewModel.increment()` |
| `emit(newState)` | `_state.update { it.copy(...) }` |
| BLoC state sealed class | `data class UiState(...)` |
| `BlocBuilder<Bloc, State>` | `collectAsStateWithLifecycle()` |
| `BlocListener` | `LaunchedEffect` observing a flow |

---

## 7. 🔁 Lifecycle — `initState` / `dispose` → `LaunchedEffect` / `DisposableEffect`

### Flutter
```dart
class MyWidget extends StatefulWidget { ... }

class _MyWidgetState extends State<MyWidget> {
  late final ScrollController _controller;

  @override
  void initState() {            // runs once when widget mounts
    super.initState();
    _controller = ScrollController();
    _loadData();
  }

  @override
  void didUpdateWidget(MyWidget old) {  // runs when widget's props change
    super.didUpdateWidget(old);
    if (old.userId != widget.userId) _loadData();
  }

  @override
  void dispose() {              // cleanup when widget unmounts
    _controller.dispose();
    super.dispose();
  }
}
```

### Android Compose
```kotlin
@Composable
fun MyScreen(userId: String, viewModel: MyViewModel = viewModel()) {

    // initState() equivalent — runs once
    LaunchedEffect(Unit) {
        viewModel.loadData()
    }

    // didUpdateWidget(userId) equivalent — runs when userId changes
    LaunchedEffect(userId) {
        viewModel.loadDataForUser(userId)
    }

    // dispose() equivalent — cleanup on unmount
    val scrollState = rememberScrollState()
    DisposableEffect(Unit) {
        // setup...
        onDispose {
            // cleanup — like dispose()
        }
    }
}
```

| Flutter Lifecycle | Compose Equivalent |
|---|---|
| `initState()` | `LaunchedEffect(Unit) { }` |
| `didUpdateWidget(old)` | `LaunchedEffect(key) { }` — re-runs when key changes |
| `dispose()` | `DisposableEffect(Unit) { onDispose { } }` |
| `didChangeDependencies()` | `LaunchedEffect(dependency) { }` |
| `build()` | The `@Composable` function body itself |
| Re-render on `setState` | Recomposition on state change |

---

## 8. 🗺️ Navigation — GoRouter / Navigator → Compose Navigation

### Flutter — GoRouter
```dart
// router setup
final router = GoRouter(
  initialLocation: '/home',
  routes: [
    GoRoute(path: '/home', builder: (ctx, state) => const HomeScreen()),
    GoRoute(
      path: '/detail/:id',
      builder: (ctx, state) {
        final id = state.pathParameters['id']!;
        return DetailScreen(id: id);
      },
    ),
    GoRoute(path: '/profile', builder: (ctx, state) => const ProfileScreen()),
  ],
);

// Navigate
context.go('/detail/42');
context.pop();
context.push('/profile');
```

### Android Compose — Navigation Component
```kotlin
// Same structure — routes + destinations
object Routes {
    const val HOME = "home"
    const val DETAIL = "detail/{id}"
    const val PROFILE = "profile"
    fun detail(id: String) = "detail/$id"
}

@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = Routes.HOME) {
        composable(Routes.HOME) {
            HomeScreen(navController = navController)
        }
        composable(
            route = Routes.DETAIL,
            arguments = listOf(navArgument("id") { type = NavType.StringType })
        ) { entry ->
            val id = entry.arguments?.getString("id") ?: ""
            DetailScreen(id = id, navController = navController)
        }
        composable(Routes.PROFILE) {
            ProfileScreen(navController = navController)
        }
    }
}

// Navigate
navController.navigate(Routes.detail("42"))  // context.go('/detail/42')
navController.popBackStack()                 // context.pop()
navController.navigate(Routes.PROFILE)       // context.push('/profile')
```

### Bottom Navigation

```dart
// Flutter — BottomNavigationBar
Scaffold(
  body: _screens[_currentIndex],
  bottomNavigationBar: BottomNavigationBar(
    currentIndex: _currentIndex,
    onTap: (i) => setState(() => _currentIndex = i),
    items: const [
      BottomNavigationBarItem(icon: Icon(Icons.home), label: 'Home'),
      BottomNavigationBarItem(icon: Icon(Icons.search), label: 'Search'),
      BottomNavigationBarItem(icon: Icon(Icons.person), label: 'Profile'),
    ],
  ),
)
```

```kotlin
// Android Compose — NavigationBar
Scaffold(
    bottomBar = {
        NavigationBar {
            val currentRoute by navController.currentBackStackEntryAsState()
            listOf(
                Triple("home", Icons.Default.Home, "Home"),
                Triple("search", Icons.Default.Search, "Search"),
                Triple("profile", Icons.Default.Person, "Profile"),
            ).forEach { (route, icon, label) ->
                NavigationBarItem(
                    selected = currentRoute?.destination?.route == route,
                    onClick = { navController.navigate(route) { launchSingleTop = true } },
                    icon = { Icon(icon, label) },
                    label = { Text(label) }
                )
            }
        }
    }
) { padding ->
    NavHost(navController, "home", Modifier.padding(padding)) {
        composable("home") { HomeScreen() }
        composable("search") { SearchScreen() }
        composable("profile") { ProfileScreen() }
    }
}
```

| Flutter | Android |
|---|---|
| `GoRouter` / `Navigator` | `NavController` + `NavHost` |
| `GoRoute(path: '/x', builder: ...)` | `composable("x") { Screen() }` |
| `context.go('/detail/42')` | `navController.navigate("detail/42")` |
| `context.pop()` | `navController.popBackStack()` |
| `state.pathParameters['id']` | `entry.arguments?.getString("id")` |
| `router.initialLocation` | `startDestination` in `NavHost` |
| Deep links in GoRouter | `navDeepLink { uriPattern = "..." }` |

---

## 9. 🌐 Networking — `http` / `Dio` → Retrofit

### Flutter — Dio
```dart
// Dio setup
final dio = Dio(BaseOptions(baseUrl: 'https://api.example.com/'));

// API call
Future<User> getUser(int id) async {
  final response = await dio.get('/users/$id');
  return User.fromJson(response.data);
}

// Error handling
try {
  final user = await getUser(1);
  setState(() => _user = user);
} on DioException catch (e) {
  setState(() => _error = e.message);
}
```

### Android — Retrofit
```kotlin
// Retrofit setup
interface ApiService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Int): User
}

val api = Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .addConverterFactory(GsonConverterFactory.create())
    .build()
    .create(ApiService::class.java)

// In ViewModel (same try/catch pattern!)
fun loadUser(id: Int) {
    viewModelScope.launch {
        try {
            _user.value = api.getUser(id)
        } catch (e: Exception) {
            _error.value = e.message
        }
    }
}
```

| Flutter | Android |
|---|---|
| `Dio` / `http` package | Retrofit + OkHttp |
| `BaseOptions(baseUrl: ...)` | `Retrofit.Builder().baseUrl(...)` |
| `dio.get('/users/$id')` | `@GET("users/{id}") suspend fun getUser(...)` |
| `dio.post('/users', data: body)` | `@POST("users") suspend fun createUser(@Body user: User)` |
| `fromJson()` / `toJson()` (manual) | Gson / Moshi auto-converts |
| `Interceptor` in Dio | `OkHttp Interceptor` |
| `Future<T>` | `suspend fun` returning `T` |
| `async / await` | `suspend` + coroutines (`launch`, `withContext`) |
| `try on DioException` | `try catch (e: HttpException)` |

---

## 10. 💾 Storage — `SharedPreferences` / `Hive` → DataStore / Room

### Flutter — SharedPreferences
```dart
// SharedPreferences
final prefs = await SharedPreferences.getInstance();

// Save
await prefs.setString('token', 'abc123');
await prefs.setBool('darkMode', true);

// Read
final token = prefs.getString('token');
final isDark = prefs.getBool('darkMode') ?? false;
```

### Android — DataStore
```kotlin
val Context.dataStore by preferencesDataStore("user_prefs")

val TOKEN_KEY = stringPreferencesKey("token")
val DARK_MODE_KEY = booleanPreferencesKey("darkMode")

// Save
suspend fun saveToken(context: Context, token: String) {
    context.dataStore.edit { it[TOKEN_KEY] = token }
}

// Read (reactive Flow — auto-updates observers!)
fun getToken(context: Context): Flow<String?> =
    context.dataStore.data.map { it[TOKEN_KEY] }
```

### Flutter — Hive (local DB) vs Room
```dart
// Hive model
@HiveType(typeId: 0)
class User extends HiveObject {
  @HiveField(0) late String name;
  @HiveField(1) late String email;
}

// CRUD
final box = await Hive.openBox<User>('users');
box.add(User()..name = 'Alice'..email = 'alice@test.com');
final users = box.values.toList();
```

```kotlin
// Room (Android)
@Entity
data class User(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val name: String,
    val email: String
)

@Dao
interface UserDao {
    @Insert suspend fun insert(user: User)
    @Query("SELECT * FROM user") fun getAll(): Flow<List<User>>
}
```

| Flutter | Android |
|---|---|
| `SharedPreferences` | `DataStore<Preferences>` |
| `prefs.setString(key, value)` | `dataStore.edit { it[KEY] = value }` |
| `prefs.getString(key)` | `dataStore.data.map { it[KEY] }` (Flow) |
| `Hive` box (local DB) | `Room` database |
| `@HiveType` model | `@Entity` data class |
| `HiveObject` | Room entity (no parent class needed) |
| `box.add(obj)` | `dao.insert(obj)` |
| `box.values.toList()` | `dao.getAll()` — returns `Flow<List<T>>` |
| `Isar` (newer DB) | Room + Kotlin Flows (equivalent power) |

---

## 11. 🎨 Theming — `ThemeData` → `MaterialTheme`

### Flutter
```dart
MaterialApp(
  theme: ThemeData(
    colorSchemeSeed: Colors.blue,
    brightness: Brightness.light,
    textTheme: const TextTheme(
      headlineLarge: TextStyle(fontSize: 32, fontWeight: FontWeight.bold),
      bodyMedium: TextStyle(fontSize: 16),
    ),
    useMaterial3: true,
  ),
  darkTheme: ThemeData(
    colorSchemeSeed: Colors.blue,
    brightness: Brightness.dark,
    useMaterial3: true,
  ),
  themeMode: ThemeMode.system,  // auto dark/light
)
```

### Android Compose
```kotlin
private val LightColors = lightColorScheme(primary = Color(0xFF1565C0))
private val DarkColors  = darkColorScheme(primary = Color(0xFF90CAF9))

val AppTypography = Typography(
    headlineLarge = TextStyle(fontSize = 32.sp, fontWeight = FontWeight.Bold),
    bodyMedium = TextStyle(fontSize = 16.sp),
)

@Composable
fun MyAppTheme(content: @Composable () -> Unit) {
    val darkTheme = isSystemInDarkTheme()      // ThemeMode.system
    MaterialTheme(
        colorScheme = if (darkTheme) DarkColors else LightColors,
        typography = AppTypography,
        content = content
    )
}
```

### Reading Theme Values

```dart
// Flutter — Theme.of(context)
Text(
  'Hello',
  style: Theme.of(context).textTheme.headlineLarge,
)
Container(color: Theme.of(context).colorScheme.primary)
```

```kotlin
// Compose — MaterialTheme.*
Text(
    text = "Hello",
    style = MaterialTheme.typography.headlineLarge
)
Box(modifier = Modifier.background(MaterialTheme.colorScheme.primary))
```

| Flutter | Android |
|---|---|
| `ThemeData(colorSchemeSeed: ...)` | `lightColorScheme(primary = ...)` |
| `ThemeMode.system` | `isSystemInDarkTheme()` |
| `Theme.of(context).colorScheme.primary` | `MaterialTheme.colorScheme.primary` |
| `Theme.of(context).textTheme.bodyLarge` | `MaterialTheme.typography.bodyLarge` |
| `useMaterial3: true` | Material 3 is default in Compose |

---

## 12. ✨ Animation — Flutter Animations → Compose Animations

### Flutter
```dart
// AnimatedContainer — auto-animates changes
AnimatedContainer(
  duration: const Duration(milliseconds: 300),
  width: _expanded ? 200 : 100,
  color: _expanded ? Colors.blue : Colors.grey,
  child: const Text('Tap me'),
)

// AnimatedOpacity
AnimatedOpacity(
  opacity: _visible ? 1.0 : 0.0,
  duration: const Duration(milliseconds: 300),
  child: const Text('Hello'),
)

// AnimatedSwitcher (replace content with animation)
AnimatedSwitcher(
  duration: const Duration(milliseconds: 200),
  child: Text('$count', key: ValueKey(count)),
)
```

### Android Compose
```kotlin
// animateAsState — like AnimatedContainer
val width by animateDpAsState(
    targetValue = if (expanded) 200.dp else 100.dp,
    animationSpec = tween(300),
    label = "width"
)
val color by animateColorAsState(
    targetValue = if (expanded) Color.Blue else Color.Gray,
    label = "color"
)
Box(modifier = Modifier.width(width).background(color)) { Text("Tap me") }

// AnimatedVisibility — like AnimatedOpacity
AnimatedVisibility(
    visible = visible,
    enter = fadeIn(tween(300)),
    exit = fadeOut(tween(300))
) {
    Text("Hello")
}

// AnimatedContent — like AnimatedSwitcher
AnimatedContent(targetState = count, label = "counter") { targetCount ->
    Text("$targetCount")
}
```

| Flutter | Compose |
|---|---|
| `AnimatedContainer(duration: ...)` | `animateDpAsState()` / `animateColorAsState()` |
| `AnimatedOpacity` | `AnimatedVisibility` with `fadeIn/fadeOut` |
| `AnimatedSwitcher` | `AnimatedContent` |
| `AnimationController` | `updateTransition` / `Animatable` |
| `Curves.easeInOut` | `tween(easing = FastOutSlowInEasing)` |
| `Duration(milliseconds: 300)` | `tween(durationMillis = 300)` |
| `SlideTransition` | `slideInHorizontally()` + `slideOutHorizontally()` |
| `FadeTransition` | `fadeIn()` + `fadeOut()` |
| `ScaleTransition` | `scaleIn()` + `scaleOut()` |

---

## 13. 🧵 Async — `Future` / `Stream` → Coroutines / Flow

### Flutter
```dart
// Future (one-shot async) → Coroutine
Future<List<Item>> fetchItems() async {
  await Future.delayed(const Duration(seconds: 1)); // simulate delay
  return [Item('A'), Item('B')];
}

// Stream (continuous data) → Flow
Stream<int> countStream() async* {
  for (int i = 0; i < 5; i++) {
    await Future.delayed(const Duration(seconds: 1));
    yield i;
  }
}
```

### Android Kotlin
```kotlin
// suspend fun (like Future)
suspend fun fetchItems(): List<Item> {
    delay(1000)                   // Future.delayed
    return listOf(Item("A"), Item("B"))
}

// Flow (like Stream)
fun countFlow(): Flow<Int> = flow {
    for (i in 0..4) {
        delay(1000)
        emit(i)                   // yield i
    }
}
```

### Using Streams / Flows in UI

```dart
// Flutter — StreamBuilder
StreamBuilder<int>(
  stream: countStream(),
  builder: (context, snapshot) {
    if (snapshot.hasData) return Text('${snapshot.data}');
    return const CircularProgressIndicator();
  },
)
```

```kotlin
// Compose — collectAsStateWithLifecycle
@Composable
fun CounterDisplay(viewModel: MyViewModel = viewModel()) {
    val count by viewModel.countFlow.collectAsStateWithLifecycle(initialValue = 0)
    Text("$count")
}
```

| Flutter / Dart | Android / Kotlin |
|---|---|
| `Future<T>` | `suspend fun` returning `T` |
| `async / await` | `suspend` + coroutines |
| `Stream<T>` | `Flow<T>` |
| `async*` / `yield` | `flow { emit() }` |
| `StreamController` | `MutableSharedFlow` / `MutableStateFlow` |
| `StreamBuilder` | `collectAsStateWithLifecycle()` |
| `FutureBuilder` | State + `LaunchedEffect` |
| `Future.wait([a, b])` | `awaitAll(async{a}, async{b})` |
| `Future.delayed(Duration(...))` | `delay(millis)` |
| `then().catchError()` | `try { } catch { }` |

---

## 14. 🔌 Dependency Injection — `get_it` / `Riverpod` → Hilt

### Flutter — get_it
```dart
// service locator setup
final getIt = GetIt.instance;

void setup() {
  getIt.registerSingleton<ApiService>(ApiService());
  getIt.registerFactory<UserRepository>(() => UserRepositoryImpl(getIt<ApiService>()));
}

// Usage
final repo = getIt<UserRepository>();
```

### Android — Hilt
```kotlin
// Hilt module
@Module
@InstallIn(SingletonComponent::class)
object AppModule {

    @Provides
    @Singleton
    fun provideApiService(): ApiService = Retrofit.Builder()
        .baseUrl("https://api.example.com/")
        .build()
        .create(ApiService::class.java)

    @Provides
    @Singleton
    fun provideUserRepository(api: ApiService): UserRepository = UserRepositoryImpl(api)
}

// Inject into ViewModel
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() { ... }

// Inject into Activity/Fragment
@AndroidEntryPoint
class MainActivity : ComponentActivity() { ... }
```

| Flutter | Android |
|---|---|
| `get_it` service locator | Hilt (compile-time DI) |
| `registerSingleton<T>(T())` | `@Provides @Singleton fun provide...` |
| `registerFactory<T>(() => T())` | `@Provides fun provide...` (no Singleton) |
| `getIt<MyService>()` | `@Inject constructor(val service: MyService)` |
| Riverpod Provider as DI | `@HiltViewModel` + `@Inject` |

---

## 🗺️ Full Architecture Comparison

```
Flutter App                              Android App
─────────────────────────────────────    ──────────────────────────────────────
main.dart                                MainActivity.kt
  runApp(MyApp())                          setContent { AppTheme { AppNav() } }
  MaterialApp(router: goRouter)

GoRouter                                 NavHost(navController, startDest)
  GoRoute('/home', HomeScreen)             composable("home") { HomeScreen() }
  GoRoute('/detail/:id', DetailScreen)     composable("detail/{id}") { ... }

HomeScreen (StatefulWidget)              HomeScreen (@Composable)
  state: items, isLoading, error           val uiState by vm.state.collectAs...
  initState() → fetchItems()              LaunchedEffect(Unit) { vm.load() }
  setState() to update                     _state.update { it.copy(...) }
  dispose()                                DisposableEffect { onDispose { } }

Riverpod Provider / BLoC                 ViewModel + StateFlow
  StateNotifier / Cubit                    class MyViewModel : ViewModel()
  emit(newState)                           _state.update { it.copy(...) }
  ref.watch(provider)                      collectAsStateWithLifecycle()

Hive / SharedPreferences                 Room / DataStore
Dio / http                               Retrofit + OkHttp
ThemeData + ThemeMode.system             MaterialTheme + isSystemInDarkTheme()
AnimatedContainer / AnimatedOpacity      animateAsState / AnimatedVisibility
Stream<T>                                Flow<T>
Future<T>                                suspend fun + Coroutine
```

---

## ⚡ Quickfire "How do I...?" Reference

| What you want to do | Flutter | Android (Compose) |
|---|---|---|
| Conditional widget | `condition ? Widget() : SizedBox()` | `if (condition) { Widget() }` |
| Map list to widgets | `items.map((i) => Widget(i)).toList()` | `items { item -> Widget(item) }` in LazyColumn |
| String interpolation | `'Hello $name'` | `"Hello $name"` (identical!) |
| Null safety | `name ?? 'Guest'` | `name ?: "Guest"` |
| Named constructor params | `Widget(key: value)` | `Widget(key = value)` |
| Spread a list | `[...list1, ...list2]` | `list1 + list2` or `buildList { addAll(l1); addAll(l2) }` |
| Show a SnackBar | `ScaffoldMessenger.of(ctx).showSnackBar(...)` | `snackbarHostState.showSnackbar(...)` |
| Show a Dialog | `showDialog(context, builder: ...)` | `if (showDialog) AlertDialog(onDismiss = {...})` |
| Navigate forward | `context.go('/route')` | `navController.navigate("route")` |
| Navigate back | `context.pop()` | `navController.popBackStack()` |
| Check dark mode | `Theme.of(context).brightness == Brightness.dark` | `isSystemInDarkTheme()` |
| Get screen size | `MediaQuery.of(context).size.width` | `LocalConfiguration.current.screenWidthDp.dp` |
| Run code once on appear | `initState()` | `LaunchedEffect(Unit) { }` |
| Cleanup on disappear | `dispose()` | `DisposableEffect { onDispose { } }` |
| Debounce a search | `RxDart debounceTime` / custom timer | `snapshotFlow { query }.debounce(300).collect { }` |
| Open URL in browser | `launchUrl(Uri.parse(url))` | `startActivity(Intent(ACTION_VIEW, Uri.parse(url)))` |
| Share text | `Share.share(text)` | `startActivity(Intent.createChooser(shareIntent, ""))` |
| Request permission | `Permission.camera.request()` | `ActivityResultContracts.RequestPermission()` launcher |
| Load image from URL | `Image.network(url)` | `AsyncImage(model = url)` (Coil) |
| Format a date | `DateFormat.yMMMd().format(date)` | `SimpleDateFormat("MMM d, yyyy").format(date)` |
