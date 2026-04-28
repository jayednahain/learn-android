# Step 29 — Compose Navigation

---

## 📖 What Is It? (Simple Explanation)

Think of your app like a **book** 📚. Each screen is a **page**. Navigation is the ability to **flip** from one page to another — and flip back.

**Compose Navigation** replaces the old Fragment-based Navigation Component for Compose apps. Instead of XML navigation graphs and fragments, you define your screens as `@Composable` functions and navigate between them in code.

---

## 🧭 Where Do We Use It?

- Any app with **more than one screen**
- Replacing Fragment Navigation in Compose apps
- Passing data between screens
- Building apps with Bottom Navigation Bar
- Any deep link integration (opening app screens from a URL)

---

## 🗺️ Workflow / How It Works

```
NavHost
  │
  ├── "home"  ────────► HomeScreen()
  ├── "profile/{id}" ─► ProfileScreen(id)
  └── "settings" ─────► SettingsScreen()

navController.navigate("profile/42")
    ↓
NavHost routes to ProfileScreen with id = 42
```

### Back Stack Diagram

```
User presses Back
         ↑
  [Settings]   ← current screen
  [Profile]
  [Home]       ← bottom of stack

After Back press:
  [Profile]    ← now on top
  [Home]
```

---

## ☕ Java/XML vs Compose Navigation — What Changed?

### Old Way (Fragment-based, XML nav graph)

```xml
<!-- res/navigation/nav_graph.xml — separate file -->
<navigation app:startDestination="@id/homeFragment">
    <fragment android:id="@+id/homeFragment" android:name=".HomeFragment">
        <action android:id="@+id/action_home_to_profile"
            app:destination="@id/profileFragment"/>
    </fragment>
    <fragment android:id="@+id/profileFragment" android:name=".ProfileFragment"/>
</navigation>
```

```kotlin
// In Fragment — navigate
findNavController().navigate(R.id.action_home_to_profile)

// Passing args — must use Safe Args plugin, bundle
val bundle = Bundle().apply { putInt("userId", 42) }
findNavController().navigate(R.id.action_home_to_profile, bundle)
```

### Compose Way — all in Kotlin, no XML needed

```kotlin
// Define all routes as string constants (no XML)
NavHost(navController = navController, startDestination = "home") {
    composable("home") { HomeScreen(navController) }
    composable("profile/{userId}") { backStackEntry ->
        val userId = backStackEntry.arguments?.getString("userId")
        ProfileScreen(userId = userId)
    }
}

// Navigate
navController.navigate("profile/42")
```

> ✅ No XML. No fragment classes. No IDs. Just strings and composables.

---

## 🔑 Key Concepts

---

### 1. Setup — Add the Dependency

```groovy
// build.gradle (app)
implementation("androidx.navigation:navigation-compose:2.7.7")
```

---

### 2. Three Core Pieces

```
NavController   → knows where you are and where to go
NavHost         → the container that shows each screen
composable()    → registers a route ("address") to a screen
```

---

### 3. `NavController` — The Navigator

```kotlin
val navController = rememberNavController()
// That's it — Compose manages it with remember
```

---

### 4. `NavHost` — The Screen Container

```kotlin
NavHost(
    navController = navController,
    startDestination = "home"   // ← first screen shown
) {
    composable("home") {
        HomeScreen(navController = navController)
    }
    composable("detail") {
        DetailScreen(navController = navController)
    }
}
```

---

### 5. Navigating Between Screens

```kotlin
// Go forward
navController.navigate("detail")

// Go back (like pressing the back button)
navController.popBackStack()

// Go back to a specific destination (clearing the stack on top)
navController.popBackStack("home", inclusive = false)

// Navigate and clear history (like login → home, no back to login)
navController.navigate("home") {
    popUpTo("login") { inclusive = true }
}
```

---

### 6. Passing Arguments Between Screens

```kotlin
// Step 1: Define route with {argument}
composable("profile/{userId}") { backStackEntry ->
    val userId = backStackEntry.arguments?.getString("userId") ?: ""
    ProfileScreen(userId = userId)
}

// Step 2: Navigate with the value
navController.navigate("profile/42")
```

#### Typed Arguments (recommended)

```kotlin
composable(
    route = "profile/{userId}",
    arguments = listOf(navArgument("userId") { type = NavType.IntType })
) { backStackEntry ->
    val userId = backStackEntry.arguments?.getInt("userId") ?: 0
    ProfileScreen(userId = userId)
}

navController.navigate("profile/42") // still a string in the path
```

---

### 7. Bottom Navigation with Compose Navigation

```kotlin
sealed class BottomNavItem(val route: String, val icon: ImageVector, val label: String) {
    object Home : BottomNavItem("home", Icons.Default.Home, "Home")
    object Search : BottomNavItem("search", Icons.Default.Search, "Search")
    object Profile : BottomNavItem("profile", Icons.Default.Person, "Profile")
}

@Composable
fun MainScreen() {
    val navController = rememberNavController()
    val items = listOf(BottomNavItem.Home, BottomNavItem.Search, BottomNavItem.Profile)

    Scaffold(
        bottomBar = {
            NavigationBar {
                val currentRoute = currentRoute(navController)
                items.forEach { item ->
                    NavigationBarItem(
                        icon = { Icon(item.icon, contentDescription = item.label) },
                        label = { Text(item.label) },
                        selected = currentRoute == item.route,
                        onClick = {
                            navController.navigate(item.route) {
                                popUpTo(navController.graph.startDestinationId) { saveState = true }
                                launchSingleTop = true
                                restoreState = true
                            }
                        }
                    )
                }
            }
        }
    ) { paddingValues ->
        NavHost(
            navController = navController,
            startDestination = BottomNavItem.Home.route,
            modifier = Modifier.padding(paddingValues)
        ) {
            composable(BottomNavItem.Home.route) { HomeScreen() }
            composable(BottomNavItem.Search.route) { SearchScreen() }
            composable(BottomNavItem.Profile.route) { ProfileScreen() }
        }
    }
}

// Helper to get current route
@Composable
fun currentRoute(navController: NavController): String? {
    val navBackStackEntry by navController.currentBackStackEntryAsState()
    return navBackStackEntry?.destination?.route
}
```

---

### 8. Deep Links in Compose Navigation

```kotlin
composable(
    route = "product/{productId}",
    deepLinks = listOf(
        navDeepLink { uriPattern = "myapp://product/{productId}" }
    )
) { backStackEntry ->
    val productId = backStackEntry.arguments?.getString("productId")
    ProductScreen(productId)
}
```

**`AndroidManifest.xml`**
```xml
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <category android:name="android.intent.category.BROWSABLE"/>
        <data android:scheme="myapp" android:host="product"/>
    </intent-filter>
</activity>
```

---

## 💡 Full Practical Example — 3-Screen App

```kotlin
// Routes
object Routes {
    const val HOME = "home"
    const val DETAIL = "detail/{itemId}"
    const val SETTINGS = "settings"
    fun detail(id: Int) = "detail/$id"
}

// App entry point
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = Routes.HOME) {

        composable(Routes.HOME) {
            HomeScreen(
                onItemClick = { id -> navController.navigate(Routes.detail(id)) },
                onSettingsClick = { navController.navigate(Routes.SETTINGS) }
            )
        }

        composable(
            route = Routes.DETAIL,
            arguments = listOf(navArgument("itemId") { type = NavType.IntType })
        ) { entry ->
            val itemId = entry.arguments?.getInt("itemId") ?: 0
            DetailScreen(
                itemId = itemId,
                onBack = { navController.popBackStack() }
            )
        }

        composable(Routes.SETTINGS) {
            SettingsScreen(onBack = { navController.popBackStack() })
        }
    }
}

@Composable
fun HomeScreen(onItemClick: (Int) -> Unit, onSettingsClick: () -> Unit) {
    Column(modifier = Modifier.fillMaxSize().padding(16.dp)) {
        Row(
            modifier = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceBetween
        ) {
            Text("Home", fontSize = 24.sp, fontWeight = FontWeight.Bold)
            IconButton(onClick = onSettingsClick) {
                Icon(Icons.Default.Settings, contentDescription = "Settings")
            }
        }
        Spacer(Modifier.height(16.dp))
        LazyColumn {
            items(10) { index ->
                Card(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(vertical = 4.dp)
                        .clickable { onItemClick(index) }
                ) {
                    Text("Item #$index", modifier = Modifier.padding(16.dp))
                }
            }
        }
    }
}
```

---

## 📊 Quick Comparison Summary

| Old (Fragment Navigation) | Compose Navigation |
|---|---|
| XML `nav_graph.xml` | Kotlin `NavHost { composable() }` |
| Fragment classes | `@Composable` functions |
| `findNavController().navigate(R.id.action)` | `navController.navigate("route")` |
| Safe Args plugin for arguments | `navArgument()` with `NavType` |
| `Bundle` for passing data | URL-style `"route/{arg}"` |
| `NavHostFragment` in XML | `NavHost()` composable |
| Fragment backstack | Compose backstack (same concept, no XML) |
