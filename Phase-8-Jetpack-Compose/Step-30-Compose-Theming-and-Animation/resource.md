# Step 30 — Compose Theming & Animation

---

## 📖 What Is It? (Simple Explanation)

### Theming
Imagine you're decorating a house 🏠. Instead of painting every room individually with different colors, you pick **one color scheme** for the entire house (walls = blue, furniture = white, accents = gold). Every room follows that scheme automatically.

**MaterialTheme** in Compose is that color scheme. Set it once at the top of your app — every component (button, text, card) reads from it automatically. Change the theme → everything updates.

### Animation
Animation is making things **move smoothly** instead of snapping instantly. A button that fades in, a card that slides up, text that changes with a cross-fade — Compose makes these really easy compared to the Java days of XML animators.

---

## 🧭 Where Do We Use It?

- **Theming**: Every Compose app should have `MaterialTheme` wrapping the whole app
- **Dark/Light theme**: Automatic switching based on system setting
- **Animations**: Loading states, screen transitions, UI feedback (hide/show, color changes)

---

## 🗺️ Workflow

### Theme Structure

```
MaterialTheme
    ├── colorScheme  ← all colors (primary, surface, error, etc.)
    ├── typography   ← all text styles (headline, body, label)
    └── shapes       ← rounded corners for cards/buttons

Your App Components read from MaterialTheme automatically
Button  →  uses MaterialTheme.colorScheme.primary
Text    →  uses MaterialTheme.typography.bodyMedium
Card    →  uses MaterialTheme.shapes.medium
```

### Animation Flow

```
State changes (visible = true/false)
         │
         ▼
AnimatedVisibility / animateAsState
         │
         ▼
Compose smoothly interpolates between old and new values
         │
         ▼
UI animates — no AnimatorSet, no XML animators needed
```

---

## ☕ Java/XML vs Compose — What Changed?

### Old Way — XML Colors and Themes (scattered across many files)

```xml
<!-- res/values/colors.xml -->
<color name="primaryColor">#6200EE</color>
<color name="primaryDarkColor">#3700B3</color>

<!-- res/values/themes.xml -->
<style name="Theme.MyApp" parent="Theme.MaterialComponents.DayNight.DarkActionBar">
    <item name="colorPrimary">@color/primaryColor</item>
    <item name="colorPrimaryDark">@color/primaryDarkColor</item>
</style>

<!-- res/values-night/themes.xml (duplicate file for dark theme!) -->
<style name="Theme.MyApp" parent="...">
    <item name="colorPrimary">@color/darkPrimary</item>
</style>
```

### Compose Way — all in one Kotlin file

```kotlin
// ui/theme/Theme.kt
private val LightColors = lightColorScheme(
    primary = Color(0xFF6200EE),
    secondary = Color(0xFF03DAC5)
)
private val DarkColors = darkColorScheme(
    primary = Color(0xFFBB86FC),
    secondary = Color(0xFF03DAC5)
)

@Composable
fun MyAppTheme(darkTheme: Boolean = isSystemInDarkTheme(), content: @Composable () -> Unit) {
    val colors = if (darkTheme) DarkColors else LightColors
    MaterialTheme(colorScheme = colors, typography = Typography, content = content)
}
```

> ✅ One file. No duplicate XML. Dark theme with one `if` statement.

---

## 🔑 Key Concepts

---

### PART A — THEMING

---

### 1. `MaterialTheme` — The Theme Provider

Wrap your whole app in `MaterialTheme`. Child composables automatically pick up colors, fonts, shapes.

```kotlin
// MainActivity.kt
setContent {
    MyAppTheme {               // ← custom theme wrapper
        Surface(
            modifier = Modifier.fillMaxSize(),
            color = MaterialTheme.colorScheme.background
        ) {
            AppNavigation()
        }
    }
}
```

---

### 2. Color Scheme

```kotlin
// Reading theme colors inside a composable
Text(
    text = "Hello",
    color = MaterialTheme.colorScheme.onBackground  // proper text color for background
)
Card(colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.surface)) {
    // card content
}
```

**Key color roles:**

| Role | Usage |
|---|---|
| `primary` | Main brand color (buttons, FAB) |
| `onPrimary` | Text/icons on top of primary |
| `surface` | Card and sheet background |
| `background` | Screen background |
| `error` | Error state color |
| `onBackground` | Text color on background |

---

### 3. Typography

```kotlin
// Define in ui/theme/Type.kt
val Typography = Typography(
    headlineLarge = TextStyle(fontSize = 32.sp, fontWeight = FontWeight.Bold),
    bodyMedium = TextStyle(fontSize = 16.sp),
    labelSmall = TextStyle(fontSize = 12.sp, color = Color.Gray)
)

// Use in composables
Text(text = "Title", style = MaterialTheme.typography.headlineLarge)
Text(text = "Body", style = MaterialTheme.typography.bodyMedium)
```

---

### 4. Dark Theme (Auto)

```kotlin
@Composable
fun MyAppTheme(content: @Composable () -> Unit) {
    val darkTheme = isSystemInDarkTheme()  // reads system setting automatically!
    val colors = if (darkTheme) DarkColors else LightColors
    MaterialTheme(colorScheme = colors, content = content)
}
```

> No more `res/values-night/` folder!

---

### 5. `CompositionLocal` — Pass Data Down Without Parameters

For data that many composables need without being passed explicitly (like a theme, locale, or user info).

```kotlin
val LocalUser = compositionLocalOf<User?> { null }

// High up in tree — provide value
CompositionLocalProvider(LocalUser provides currentUser) {
    MyScreen()
}

// Deep inside MyScreen — read without passing as parameter
@Composable
fun UserAvatar() {
    val user = LocalUser.current  // gets the user without it being passed down
    Text(user?.name ?: "Guest")
}
```

---

### PART B — ANIMATION

---

### 6. `animateAsState` — Animate a Single Value

The easiest animation. Just change a value and Compose animates the transition.

```kotlin
@Composable
fun PulseButton() {
    var expanded by remember { mutableStateOf(false) }
    
    val size by animateDpAsState(
        targetValue = if (expanded) 200.dp else 100.dp,
        animationSpec = tween(durationMillis = 300),
        label = "size animation"
    )

    Box(
        modifier = Modifier
            .size(size)
            .background(Color.Blue, CircleShape)
            .clickable { expanded = !expanded }
    )
}
```

**Variants:**

```kotlin
animateDpAsState()      // animate Dp values (size, padding)
animateFloatAsState()   // animate Float (alpha, rotation)
animateColorAsState()   // animate Color
animateOffsetAsState()  // animate position
```

---

### 7. `AnimatedVisibility` — Show/Hide with Animation

```kotlin
@Composable
fun RevealCard() {
    var visible by remember { mutableStateOf(false) }

    Column {
        Button(onClick = { visible = !visible }) {
            Text(if (visible) "Hide" else "Show")
        }

        AnimatedVisibility(
            visible = visible,
            enter = fadeIn() + slideInVertically(),  // enter animation
            exit = fadeOut() + slideOutVertically()  // exit animation
        ) {
            Card(modifier = Modifier.fillMaxWidth().padding(16.dp)) {
                Text("Secret content!", modifier = Modifier.padding(16.dp))
            }
        }
    }
}
```

**Enter/Exit animations you can combine:**

```kotlin
fadeIn()   / fadeOut()
slideInHorizontally()  / slideOutHorizontally()
slideInVertically()    / slideOutVertically()
expandVertically()     / shrinkVertically()
scaleIn()  / scaleOut()
```

---

### 8. `AnimatedContent` — Animate Content Change

When the same area shows different content based on state:

```kotlin
@Composable
fun ToggleDisplay() {
    var count by remember { mutableStateOf(0) }

    Column {
        Button(onClick = { count++ }) { Text("Next") }

        AnimatedContent(
            targetState = count,
            transitionSpec = {
                slideInHorizontally { it } togetherWith slideOutHorizontally { -it }
            },
            label = "count animation"
        ) { targetCount ->
            Text("Count: $targetCount", fontSize = 32.sp)
        }
    }
}
```

---

### 9. `Transition` — Coordinated Multi-Property Animation

When you want to animate multiple values together as one state changes:

```kotlin
@Composable
fun AnimatedCard(isSelected: Boolean) {
    val transition = updateTransition(targetState = isSelected, label = "card selection")

    val borderWidth by transition.animateDp(label = "border") {
        if (it) 3.dp else 0.dp
    }
    val backgroundColor by transition.animateColor(label = "bg") {
        if (it) Color(0xFFEDE7F6) else Color.White
    }

    Card(
        modifier = Modifier
            .fillMaxWidth()
            .border(borderWidth, Color.Purple, RoundedCornerShape(12.dp)),
        colors = CardDefaults.cardColors(containerColor = backgroundColor)
    ) {
        Text("Card", modifier = Modifier.padding(16.dp))
    }
}
```

---

## 💡 Full Practical Example — Themed App with Animations

```kotlin
// 1. Theme
@Composable
fun ShopAppTheme(content: @Composable () -> Unit) {
    val darkTheme = isSystemInDarkTheme()
    MaterialTheme(
        colorScheme = if (darkTheme) darkColorScheme(primary = Color(0xFFBB86FC))
                      else lightColorScheme(primary = Color(0xFF6200EE)),
        content = content
    )
}

// 2. Product Card with animation
@Composable
fun ProductCard(name: String, price: String, onAddToCart: () -> Unit) {
    var added by remember { mutableStateOf(false) }
    val bgColor by animateColorAsState(
        targetValue = if (added) Color(0xFFE8F5E9) else MaterialTheme.colorScheme.surface,
        label = "card bg"
    )

    Card(
        modifier = Modifier.fillMaxWidth().padding(8.dp),
        colors = CardDefaults.cardColors(containerColor = bgColor)
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment = Alignment.CenterVertically
        ) {
            Column {
                Text(name, style = MaterialTheme.typography.titleMedium)
                Text(price, style = MaterialTheme.typography.bodySmall,
                    color = MaterialTheme.colorScheme.primary)
            }
            AnimatedContent(targetState = added, label = "button") { isAdded ->
                if (isAdded) {
                    Icon(Icons.Default.Check, contentDescription = "Added", tint = Color.Green)
                } else {
                    Button(onClick = { added = true; onAddToCart() }) {
                        Text("Add")
                    }
                }
            }
        }
    }
}

// 3. Main app
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            ShopAppTheme {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    LazyColumn(contentPadding = PaddingValues(16.dp)) {
                        items(5) { i ->
                            ProductCard(
                                name = "Product ${i + 1}",
                                price = "$${(i + 1) * 10}.99",
                                onAddToCart = { }
                            )
                        }
                    }
                }
            }
        }
    }
}
```

---

## 📊 Quick Comparison Summary

| Old (XML + Java) | Compose |
|---|---|
| `res/values/colors.xml` + `res/values/themes.xml` | `MaterialTheme` in Kotlin |
| `res/values-night/themes.xml` (duplicate) | `isSystemInDarkTheme()` in one file |
| `ObjectAnimator`, `ValueAnimator`, XML animators | `animateAsState`, `AnimatedVisibility` |
| `View.animate().alpha(0f).duration(300)` | `animateFloatAsState(targetValue = 0f)` |
| `TransitionManager.beginDelayedTransition()` | `AnimatedContent { }` |
| `<transition>` XML files | `Transition` API in Kotlin |
| `sp` in XML | `.sp` extension in Compose |
