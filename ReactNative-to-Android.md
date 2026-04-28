# React Native → Android (Kotlin + Compose) Cheat Sheet

> You already know React Native. This maps every RN concept to its Android equivalent.
> Think of this as a **translation dictionary** between two languages that do the same things.

---

## 🧠 The Big Mental Model Shift

| React Native | Android (Kotlin + Compose) |
|---|---|
| JavaScript / TypeScript | Kotlin |
| Functional components | `@Composable` functions |
| JSX (UI in JS) | Composable functions (UI in Kotlin) |
| `npm` / `yarn` | Gradle (`build.gradle`) |
| `node_modules` | Maven dependencies |
| Metro bundler | Gradle build system |
| Hot reload | Live Edit (similar, in Android Studio) |
| `App.tsx` entry point | `MainActivity.kt` + `setContent {}` |

---

## 1. 🏗️ Project Entry Point

### React Native
```tsx
// App.tsx
export default function App() {
  return (
    <NavigationContainer>
      <RootNavigator />
    </NavigationContainer>
  );
}
```

### Android (Kotlin + Compose)
```kotlin
// MainActivity.kt
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {                    // ← replaces return ( ... )
            MyAppTheme {
                AppNavigation()         // ← your root composable
            }
        }
    }
}
```

> `setContent { }` = the `return ( )` of your root component.

---

## 2. 🧩 Components → Composable Functions

### React Native
```tsx
// RN — functional component
function UserCard({ name, email }: { name: string; email: string }) {
  return (
    <View style={styles.card}>
      <Text style={styles.name}>{name}</Text>
      <Text style={styles.email}>{email}</Text>
    </View>
  );
}
```

### Android (Compose)
```kotlin
// Compose — @Composable function (same idea, different syntax)
@Composable
fun UserCard(name: String, email: String) {
    Card(modifier = Modifier.fillMaxWidth().padding(8.dp)) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(text = name, fontWeight = FontWeight.Bold)
            Text(text = email, color = Color.Gray)
        }
    }
}
```

> `@Composable` = `function MyComponent()` in React Native.  
> Props = just function parameters, just like RN!

---

## 3. 📦 Views / Components Mapping

| React Native | Compose | Notes |
|---|---|---|
| `<View>` | `Column` / `Row` / `Box` | Column = vertical, Row = horizontal, Box = stacked |
| `<Text>` | `Text()` | Same concept |
| `<TextInput>` | `TextField()` / `OutlinedTextField()` | |
| `<Image>` | `Image()` | Use **Coil** library for URLs (`AsyncImage`) |
| `<ScrollView>` | `Column` inside `verticalScroll(rememberScrollState())` | |
| `<FlatList>` | `LazyColumn()` | Lazy = virtualized, like FlatList |
| `<FlatList horizontal>` | `LazyRow()` | |
| `<SectionList>` | `LazyColumn` with `stickyHeader { }` | |
| `<TouchableOpacity>` | `Modifier.clickable { }` | Add to any composable |
| `<Pressable>` | `Modifier.clickable { }` / `Button()` | |
| `<Button>` | `Button()` | |
| `<Switch>` | `Switch()` | |
| `<ActivityIndicator>` | `CircularProgressIndicator()` | |
| `<Modal>` | `AlertDialog()` / `BottomSheetDialog` | |
| `<SafeAreaView>` | `Scaffold()` with `paddingValues` | Scaffold handles system bars |
| `<StatusBar>` | `WindowCompat.setDecorFitsSystemWindows()` | |
| `<KeyboardAvoidingView>` | Handled by `imePadding()` Modifier | |

---

### Side-by-Side: FlatList vs LazyColumn

```tsx
// React Native — FlatList
<FlatList
  data={users}
  keyExtractor={(item) => item.id.toString()}
  renderItem={({ item }) => <UserCard name={item.name} email={item.email} />}
  ItemSeparatorComponent={() => <View style={{ height: 8 }} />}
/>
```

```kotlin
// Android Compose — LazyColumn
LazyColumn(
    verticalArrangement = Arrangement.spacedBy(8.dp)
) {
    items(users, key = { it.id }) { user ->
        UserCard(name = user.name, email = user.email)
    }
}
```

> Same concept: virtualized list, only renders visible items.  
> `keyExtractor` = `key = { it.id }`.  
> `renderItem` = the lambda block `{ user -> ... }`.

---

## 4. 🎨 Styles → Modifier + MaterialTheme

This is the **biggest difference** you'll feel.

### React Native — StyleSheet
```tsx
const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#fff',
    padding: 16,
    alignItems: 'center',
    justifyContent: 'center',
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 8,
  },
});

<View style={styles.container}>
  <Text style={styles.title}>Hello</Text>
</View>
```

### Android Compose — Modifier chain
```kotlin
// No separate StyleSheet file — style is chained directly on each component
Column(
    modifier = Modifier
        .fillMaxSize()          // flex: 1
        .background(Color.White) // backgroundColor: '#fff'
        .padding(16.dp),        // padding: 16
    horizontalAlignment = Alignment.CenterHorizontally,  // alignItems: 'center'
    verticalArrangement = Arrangement.Center             // justifyContent: 'center'
) {
    Text(
        text = "Hello",
        fontSize = 24.sp,           // fontSize: 24
        fontWeight = FontWeight.Bold,// fontWeight: 'bold'
        color = Color(0xFF333333),  // color: '#333'
        modifier = Modifier.padding(bottom = 8.dp) // marginBottom: 8
    )
}
```

### Style Quick Reference

| React Native Style | Compose Modifier / Property |
|---|---|
| `flex: 1` | `Modifier.fillMaxSize()` |
| `flexDirection: 'row'` | Use `Row { }` instead of `Column { }` |
| `padding: 16` | `Modifier.padding(16.dp)` |
| `paddingHorizontal: 16` | `Modifier.padding(horizontal = 16.dp)` |
| `margin: 8` | `Modifier.padding(8.dp)` on the parent |
| `backgroundColor: '#fff'` | `Modifier.background(Color.White)` |
| `borderRadius: 12` | `Modifier.clip(RoundedCornerShape(12.dp))` |
| `width: '100%'` | `Modifier.fillMaxWidth()` |
| `height: 200` | `Modifier.height(200.dp)` |
| `fontSize: 16` | `fontSize = 16.sp` |
| `fontWeight: 'bold'` | `fontWeight = FontWeight.Bold` |
| `color: '#333'` | `color = Color(0xFF333333)` |
| `alignItems: 'center'` | `horizontalAlignment = Alignment.CenterHorizontally` |
| `justifyContent: 'center'` | `verticalArrangement = Arrangement.Center` |
| `alignSelf: 'flex-end'` | `modifier = Modifier.align(Alignment.End)` |
| `gap: 8` (RN 0.71+) | `Arrangement.spacedBy(8.dp)` in Column/Row |
| `overflow: 'hidden'` | `Modifier.clip(shape)` |
| `elevation: 4` | `Card(elevation = CardDefaults.cardElevation(4.dp))` |
| `position: 'absolute'` | `Box { }` + `Modifier.align()` |
| `display: 'none'` | `if (visible) { MyComponent() }` |
| `opacity: 0.5` | `Modifier.alpha(0.5f)` |

---

## 5. ⚡ State — `useState` → `remember` + `mutableStateOf`

### React Native
```tsx
function Counter() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('');

  return (
    <View>
      <Text>Count: {count}</Text>
      <Button title="Add" onPress={() => setCount(count + 1)} />
      <TextInput value={name} onChangeText={setName} />
    </View>
  );
}
```

### Android Compose
```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }      // useState(0)
    var name by remember { mutableStateOf("") }       // useState('')

    Column {
        Text("Count: $count")
        Button(onClick = { count++ }) { Text("Add") } // setCount(count + 1)
        TextField(value = name, onValueChange = { name = it }) // onChangeText={setName}
    }
}
```

| React Native | Compose |
|---|---|
| `useState(initialValue)` | `remember { mutableStateOf(initialValue) }` |
| `setValue(newValue)` | `value = newValue` (with `by` delegate) |
| `const [x, setX] = useState()` | `var x by remember { mutableStateOf() }` |
| State lost on re-render | State saved by `remember` |
| State lost on screen rotation | Use `rememberSaveable` instead of `remember` |

---

## 6. 🏪 Global State — Redux/Zustand/Context → ViewModel + StateFlow

This is the biggest conceptual leap. Instead of a store, Android uses `ViewModel`.

### React Native — Zustand Store
```tsx
// store.ts
interface CounterStore {
  count: number;
  increment: () => void;
  reset: () => void;
}

const useCounterStore = create<CounterStore>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  reset: () => set({ count: 0 }),
}));

// Component
function CounterScreen() {
  const { count, increment, reset } = useCounterStore();
  return (
    <View>
      <Text>Count: {count}</Text>
      <Button title="Add" onPress={increment} />
      <Button title="Reset" onPress={reset} />
    </View>
  );
}
```

### Android — ViewModel + StateFlow
```kotlin
// CounterViewModel.kt — like the Zustand store
class CounterViewModel : ViewModel() {
    private val _count = MutableStateFlow(0)        // private state
    val count: StateFlow<Int> = _count.asStateFlow() // public read-only

    fun increment() { _count.value++ }
    fun reset() { _count.value = 0 }
}

// CounterScreen.kt — like the Component
@Composable
fun CounterScreen(viewModel: CounterViewModel = viewModel()) {
    val count by viewModel.count.collectAsStateWithLifecycle()

    Column {
        Text("Count: $count")
        Button(onClick = { viewModel.increment() }) { Text("Add") }
        Button(onClick = { viewModel.reset() }) { Text("Reset") }
    }
}
```

### Full UiState Pattern (Redux-style)
```kotlin
// Like a Redux state shape
data class HomeUiState(
    val items: List<Item> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)

class HomeViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(HomeUiState())
    val uiState = _uiState.asStateFlow()

    fun loadItems() {
        _uiState.update { it.copy(isLoading = true) }  // like dispatch(setLoading(true))
        viewModelScope.launch {
            val result = repository.getItems()
            _uiState.update { it.copy(items = result, isLoading = false) }
        }
    }
}
```

| Zustand / Redux | Android ViewModel |
|---|---|
| `create((set) => ({ ... }))` | `class MyViewModel : ViewModel()` |
| `const state = useStore()` | `val state by viewModel.state.collectAsStateWithLifecycle()` |
| `set({ count: 0 })` | `_state.update { it.copy(count = 0) }` |
| Action functions in store | Functions in ViewModel |
| Store survives re-renders | ViewModel survives screen rotation |
| Global store singleton | Scoped to screen, shared via `activityViewModels()` |

---

## 7. 🔁 Side Effects — `useEffect` → `LaunchedEffect`

### React Native
```tsx
function UserScreen({ userId }: { userId: string }) {
  const [user, setUser] = useState(null);

  // Run when userId changes
  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, [userId]);

  // Run once on mount
  useEffect(() => {
    analytics.trackScreen('UserScreen');
    return () => {
      analytics.removeTracking(); // cleanup on unmount
    };
  }, []);

  return <Text>{user?.name}</Text>;
}
```

### Android Compose
```kotlin
@Composable
fun UserScreen(userId: String, viewModel: UserViewModel = viewModel()) {
    // Run when userId changes — LaunchedEffect(key) = useEffect([dep])
    LaunchedEffect(userId) {
        viewModel.loadUser(userId)
    }

    // Run once on appearance — LaunchedEffect(Unit) = useEffect([], [])
    LaunchedEffect(Unit) {
        analytics.trackScreen("UserScreen")
    }

    // Run on disappear (cleanup) — DisposableEffect = useEffect return cleanup
    DisposableEffect(Unit) {
        onDispose {
            analytics.removeTracking()
        }
    }

    val user by viewModel.user.collectAsStateWithLifecycle()
    Text(user?.name ?: "")
}
```

| React Native `useEffect` | Compose equivalent |
|---|---|
| `useEffect(() => { }, [dep])` | `LaunchedEffect(dep) { }` |
| `useEffect(() => { }, [])` | `LaunchedEffect(Unit) { }` |
| `useEffect(() => { return cleanup }, [])` | `DisposableEffect(Unit) { onDispose { } }` |
| `useCallback` | `remember { }` with a lambda |
| `useMemo` | `remember(key) { computedValue }` |

---

## 8. 🗺️ Navigation — React Navigation → Compose Navigation

### React Native — React Navigation
```tsx
// Stack Navigator setup
const Stack = createNativeStackNavigator();

function AppNavigator() {
  return (
    <Stack.Navigator initialRouteName="Home">
      <Stack.Screen name="Home" component={HomeScreen} />
      <Stack.Screen name="Detail" component={DetailScreen} />
      <Stack.Screen name="Profile" component={ProfileScreen} />
    </Stack.Navigator>
  );
}

// Navigate
const navigation = useNavigation();
navigation.navigate('Detail', { itemId: 42 });
navigation.goBack();

// Read params
const route = useRoute();
const { itemId } = route.params;
```

### Android Compose — Navigation Component
```kotlin
// Routes — like screen name strings in RN
object Routes {
    const val HOME = "home"
    const val DETAIL = "detail/{itemId}"
    const val PROFILE = "profile"
    fun detail(id: Int) = "detail/$id"
}

// NavHost — like Stack.Navigator
@Composable
fun AppNavigator() {
    val navController = rememberNavController()     // useNavigation()

    NavHost(navController = navController, startDestination = Routes.HOME) {
        composable(Routes.HOME) {
            HomeScreen(navController = navController)
        }
        composable(
            route = Routes.DETAIL,
            arguments = listOf(navArgument("itemId") { type = NavType.IntType })
        ) { backStackEntry ->
            val itemId = backStackEntry.arguments?.getInt("itemId") ?: 0
            DetailScreen(itemId = itemId, navController = navController)
        }
        composable(Routes.PROFILE) {
            ProfileScreen(navController = navController)
        }
    }
}

// Navigate — in any screen
navController.navigate(Routes.detail(42))  // navigation.navigate('Detail', { itemId: 42 })
navController.popBackStack()               // navigation.goBack()
```

### Bottom Tab Navigation

```tsx
// React Native — Bottom Tabs
const Tab = createBottomTabNavigator();

function TabNavigator() {
  return (
    <Tab.Navigator>
      <Tab.Screen name="Home" component={HomeScreen}
        options={{ tabBarIcon: ({ color }) => <Icon name="home" color={color} /> }} />
      <Tab.Screen name="Search" component={SearchScreen} />
      <Tab.Screen name="Profile" component={ProfileScreen} />
    </Tab.Navigator>
  );
}
```

```kotlin
// Android Compose — NavigationBar
@Composable
fun TabNavigator() {
    val navController = rememberNavController()
    val tabs = listOf(
        Triple("home", Icons.Default.Home, "Home"),
        Triple("search", Icons.Default.Search, "Search"),
        Triple("profile", Icons.Default.Person, "Profile"),
    )
    val currentRoute by navController.currentBackStackEntryAsState()

    Scaffold(
        bottomBar = {
            NavigationBar {
                tabs.forEach { (route, icon, label) ->
                    NavigationBarItem(
                        selected = currentRoute?.destination?.route == route,
                        onClick = { navController.navigate(route) {
                            launchSingleTop = true
                            restoreState = true
                        }},
                        icon = { Icon(icon, label) },
                        label = { Text(label) }
                    )
                }
            }
        }
    ) { padding ->
        NavHost(navController, startDestination = "home",
            modifier = Modifier.padding(padding)) {
            composable("home") { HomeScreen() }
            composable("search") { SearchScreen() }
            composable("profile") { ProfileScreen() }
        }
    }
}
```

---

## 9. 🌐 Networking — fetch/Axios → Retrofit

### React Native
```tsx
// fetch / Axios
interface User { id: number; name: string; email: string; }

async function getUser(id: number): Promise<User> {
  const response = await fetch(`https://api.example.com/users/${id}`);
  if (!response.ok) throw new Error('Failed');
  return response.json();
}

// In component
useEffect(() => {
  getUser(1).then(setUser).catch(setError);
}, []);
```

### Android — Retrofit
```kotlin
// 1. API Interface — like your fetch function
interface ApiService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Int): User
}

// 2. Retrofit instance — like creating an Axios instance
val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .addConverterFactory(GsonConverterFactory.create())
    .build()
val api = retrofit.create(ApiService::class.java)

// 3. In ViewModel — like useEffect + fetch
class UserViewModel : ViewModel() {
    private val _user = MutableStateFlow<User?>(null)
    val user = _user.asStateFlow()

    fun loadUser(id: Int) {
        viewModelScope.launch {                     // like async/await
            try {
                _user.value = api.getUser(id)
            } catch (e: Exception) {
                // handle error
            }
        }
    }
}
```

| React Native | Android |
|---|---|
| `fetch(url)` / `axios.get(url)` | Retrofit `@GET` interface method |
| `async/await` | `suspend` functions + coroutines |
| `useEffect(() => { fetchData() }, [])` | `LaunchedEffect(Unit) { viewModel.loadData() }` |
| `try/catch` for errors | Same — `try/catch` in `viewModelScope.launch` |
| Axios interceptors | OkHttp interceptors |
| Axios instance base URL | `Retrofit.Builder().baseUrl()` |

---

## 10. 💾 Storage — AsyncStorage → DataStore

### React Native
```tsx
// AsyncStorage
import AsyncStorage from '@react-native-async-storage/async-storage';

// Save
await AsyncStorage.setItem('token', 'abc123');
await AsyncStorage.setItem('theme', JSON.stringify({ dark: true }));

// Read
const token = await AsyncStorage.getItem('token');
const theme = JSON.parse(await AsyncStorage.getItem('theme') ?? '{}');
```

### Android — DataStore
```kotlin
// Preferences DataStore
val Context.dataStore by preferencesDataStore("user_prefs")

val TOKEN_KEY = stringPreferencesKey("token")
val DARK_MODE_KEY = booleanPreferencesKey("dark_mode")

// Save — like AsyncStorage.setItem
suspend fun saveToken(context: Context, token: String) {
    context.dataStore.edit { prefs ->
        prefs[TOKEN_KEY] = token
    }
}

// Read — like AsyncStorage.getItem (returns Flow, reactive!)
fun getToken(context: Context): Flow<String?> {
    return context.dataStore.data.map { prefs -> prefs[TOKEN_KEY] }
}

// In ViewModel
viewModelScope.launch {
    getToken(context).collect { token ->
        // reacts whenever token changes — like a subscription
    }
}
```

| React Native | Android |
|---|---|
| `AsyncStorage` | `DataStore<Preferences>` |
| `setItem(key, value)` | `dataStore.edit { prefs[KEY] = value }` |
| `getItem(key)` | `dataStore.data.map { it[KEY] }` (returns Flow) |
| Async, Promise-based | Async, Coroutine + Flow based |
| String only (need JSON.stringify) | Typed — `stringPreferencesKey`, `booleanPreferencesKey`, `intPreferencesKey` |

---

## 11. 📱 Platform-Specific Concepts

| React Native | Android |
|---|---|
| `Platform.OS === 'android'` | You're always on Android — no check needed |
| `BackHandler` | System back button handled by NavController |
| `Linking.openURL(url)` | `startActivity(Intent(Intent.ACTION_VIEW, Uri.parse(url)))` |
| `Share.share({ message })` | `startActivity(Intent.createChooser(shareIntent, "Share"))` |
| `Vibration.vibrate()` | `vibrator.vibrate(VibrationEffect.createOneShot(200, 255))` |
| `Clipboard.setString(text)` | `clipboardManager.setPrimaryClip(ClipData.newPlainText("", text))` |
| `PermissionsAndroid.request()` | `ActivityResultContracts.RequestPermission()` launcher |
| `CameraRoll` / `launchImageLibrary` | Photo Picker API / MediaStore |
| `react-native-camera` | `CameraX` library |
| `@react-navigation/native-stack` | Jetpack Navigation Component |
| `react-native-maps` | Google Maps SDK for Android |
| `react-native-push-notification` | Firebase Cloud Messaging (FCM) |
| `react-native-async-storage` | Room Database / DataStore |
| `react-native-sqlite-storage` | Room Database |

---

## 12. 🧵 Async / Concurrency — Promises → Coroutines

### React Native
```tsx
// Promises / async-await
async function loadData(): Promise<void> {
  setLoading(true);
  try {
    const data = await fetchData();
    setData(data);
  } catch (error) {
    setError(error.message);
  } finally {
    setLoading(false);
  }
}
```

### Android — Coroutines
```kotlin
// Coroutines — same structure, different keywords
fun loadData() {
    viewModelScope.launch {          // like starting an async function
        _isLoading.value = true
        try {
            val data = fetchData()   // suspend fun — like await
            _data.value = data
        } catch (e: Exception) {
            _error.value = e.message
        } finally {
            _isLoading.value = false
        }
    }
}
```

| JavaScript | Kotlin Coroutines |
|---|---|
| `async function` | `suspend fun` |
| `await somePromise` | calling a `suspend fun` |
| `Promise.all([a, b])` | `awaitAll(async { a }, async { b })` |
| `setTimeout(() => {}, 1000)` | `delay(1000)` (inside a coroutine) |
| `.then().catch()` | `try { } catch { }` |
| Running in background thread | `withContext(Dispatchers.IO) { }` |

---

## 13. 🔄 Context API → CompositionLocal

### React Native
```tsx
// React Context
const ThemeContext = createContext({ isDark: false });

function App() {
  return (
    <ThemeContext.Provider value={{ isDark: true }}>
      <MyScreen />
    </ThemeContext.Provider>
  );
}

function DeepChild() {
  const { isDark } = useContext(ThemeContext); // access anywhere deep
  return <View style={{ backgroundColor: isDark ? '#000' : '#fff' }} />;
}
```

### Android — CompositionLocal
```kotlin
val LocalIsDark = compositionLocalOf { false }

@Composable
fun App() {
    CompositionLocalProvider(LocalIsDark provides true) {
        MyScreen()
    }
}

@Composable
fun DeepChild() {
    val isDark = LocalIsDark.current   // access anywhere deep
    Box(modifier = Modifier.background(if (isDark) Color.Black else Color.White))
}
```

---

## 14. 📋 Custom Hooks → Extracted Composable / ViewModel

### React Native
```tsx
// Custom hook — reusable logic
function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  const increment = () => setCount(c => c + 1);
  const reset = () => setCount(0);
  return { count, increment, reset };
}

function MyComponent() {
  const { count, increment } = useCounter(5);
  return <Button title={`Count: ${count}`} onPress={increment} />;
}
```

### Android — ViewModel or extracted Composable state
```kotlin
// Option 1: ViewModel (for logic with business code)
class CounterViewModel(initial: Int = 0) : ViewModel() {
    private val _count = MutableStateFlow(initial)
    val count = _count.asStateFlow()
    fun increment() { _count.value++ }
    fun reset() { _count.value = 0 }
}

// Option 2: Extract state into a holder class (for pure UI state)
class CounterState(initial: Int) {
    var count by mutableStateOf(initial)
    fun increment() { count++ }
    fun reset() { count = 0 }
}

@Composable
fun rememberCounterState(initial: Int = 0) = remember { CounterState(initial) }

// Usage
@Composable
fun MyComponent() {
    val counter = rememberCounterState(5)
    Button(onClick = { counter.increment() }) {
        Text("Count: ${counter.count}")
    }
}
```

---

## 🗺️ Full Architecture Comparison

```
React Native App                  Android App
────────────────────────          ──────────────────────────
App.tsx                           MainActivity.kt
  └── NavigationContainer           └── setContent { AppNavigation() }
        └── Stack.Navigator               └── NavHost
              ├── HomeScreen                    ├── HomeScreen (Composable)
              └── DetailScreen                  └── DetailScreen (Composable)

HomeScreen.tsx                    HomeScreen.kt
  useState() ←── local state        remember { mutableStateOf() }
  useEffect() ←── side effects      LaunchedEffect()
  useFetchItems() ←── data         homeViewModel.items (StateFlow)

Zustand Store / Redux             ViewModel
  state: { items, loading }         data class HomeUiState(items, isLoading)
  actions: fetchItems()             fun loadItems() in ViewModel

AsyncStorage                      DataStore / Room
axios / fetch                     Retrofit + OkHttp
StyleSheet                        Modifier chain
<FlatList>                        LazyColumn
useContext                        CompositionLocal / ViewModel
```

---

## ⚡ Quickfire "How do I...?" Reference

| What you want to do | Android (Compose) way |
|---|---|
| Show/hide a component | `if (isVisible) { MyComponent() }` |
| Conditional rendering | Same as above — just `if/else` in Kotlin |
| Map over a list in UI | `items.forEach { item -> ItemView(item) }` or `LazyColumn` |
| Pass a callback as prop | Function parameter: `onPress: () -> Unit` |
| Pass children to a component | `content: @Composable () -> Unit` parameter |
| Format strings | `"Hello, $name! You have $count items"` — Kotlin string templates |
| Navigate to screen | `navController.navigate("screenRoute")` |
| Go back | `navController.popBackStack()` |
| Load data on screen open | `LaunchedEffect(Unit) { viewModel.load() }` |
| Show a Toast | `Toast.makeText(context, "msg", Toast.LENGTH_SHORT).show()` |
| Open a URL | `startActivity(Intent(Intent.ACTION_VIEW, Uri.parse(url)))` |
| Get screen dimensions | `LocalConfiguration.current.screenWidthDp.dp` |
| Dark mode check | `isSystemInDarkTheme()` |
| Debounce a search input | `snapshotFlow { query }.debounce(300).collect { viewModel.search(it) }` |
