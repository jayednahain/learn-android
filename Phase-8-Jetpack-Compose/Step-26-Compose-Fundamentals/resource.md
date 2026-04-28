# Step 26 — Compose Fundamentals

---

## 📖 What Is It? (Simple Explanation)

Imagine you are drawing a picture 🖼️. The **old way** (XML) is like filling out a **form** — you describe your drawing in a separate paper (XML file), then attach it to your code. Every change needs you to go back and forth between two files.

**Jetpack Compose** is the **new way** — you draw the picture **directly in code**, in one place. You tell Kotlin: *"draw a red button with text 'Click Me'"*, and it appears. No XML file at all!

This is called **Declarative UI** — you describe **what** the screen should look like, and Compose figures out **how** to draw it.

---

## 🧭 Where Do We Use It?

- Building **any** new Android screen (Google's recommended modern approach)
- Replacing XML layouts entirely
- Any project started from scratch (Android Studio will offer "Empty Compose Activity" template)
- Gradually migrating old XML screens to Compose

---

## 🗺️ Workflow / How It Works

```
Old Way (XML + Java/Kotlin)          New Way (Jetpack Compose)
─────────────────────────────        ──────────────────────────────
activity_main.xml  ←──────┐          MainActivity.kt
     (layout)             │                │
     + XML views          │                ▼
MainActivity.kt ──────────┘         setContent {
     inflate layout                     MyScreen()   ← @Composable function
     find views by ID                 }
     set click listeners
     update UI manually          UI redraws automatically
                                 when State changes (Recomposition)
```

### Recomposition Cycle

```
State changes (e.g., counter = 5)
          │
          ▼
Compose re-runs only affected @Composable functions
          │
          ▼
UI on screen updates — no manual "update this view"
```

---

## ☕ Java (XML) vs Compose — What Changed?

### Old Way — XML + Java/Kotlin (~25 lines across 2 files)

**`activity_main.xml`**
```xml
<LinearLayout ...>
    <TextView
        android:id="@+id/tvHello"
        android:text="Hello World"
        android:textSize="20sp"/>
    <Button
        android:id="@+id/btnClick"
        android:text="Click Me"/>
</LinearLayout>
```

**`MainActivity.kt`**
```kotlin
override fun onCreate(...) {
    setContentView(R.layout.activity_main)
    val tvHello = findViewById<TextView>(R.id.tvHello)
    val btn = findViewById<Button>(R.id.btnClick)
    btn.setOnClickListener {
        tvHello.text = "You clicked!"
    }
}
```

### New Way — Compose (one file, ~12 lines)

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MyScreen() // call composable function
        }
    }
}

@Composable
fun MyScreen() {
    var text by remember { mutableStateOf("Hello World") }
    Column {
        Text(text = text, fontSize = 20.sp)
        Button(onClick = { text = "You clicked!" }) {
            Text("Click Me")
        }
    }
}
```

> ✅ No XML file. No `findViewById`. No IDs. Everything in one place.

---

## 🔑 Key Concepts

### 1. `@Composable` — The Magic Annotation

Any function that draws UI **must** have `@Composable` on it. Think of it like saying "this is a drawing function."

```kotlin
@Composable
fun Greeting(name: String) {
    Text(text = "Hello, $name!")
}
```

- ✅ Can call other `@Composable` functions
- ❌ Cannot be called from normal functions
- By convention: name starts with **Capital Letter** (`MyButton`, not `myButton`)

---

### 2. `setContent {}` — Replaces `setContentView()`

```kotlin
// Old XML way
setContentView(R.layout.activity_main)

// Compose way
setContent {
    MyApp()  // your root composable
}
```

---

### 3. `remember` — Compose's Memory

When Compose redraws the screen, it re-runs the function. Without `remember`, your variable resets every time. `remember` saves the value between redraws.

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    //          ↑ SAVE this value between recompositions

    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```

Think of `remember` as a sticky note 📌 that Compose keeps even when it redraws.

---

### 4. `mutableStateOf` — Reactive Variable

When this value changes, Compose automatically redraws the parts of the screen that use it. Like a magical variable!

```kotlin
var name by remember { mutableStateOf("") }
// Change it → UI updates automatically
name = "Kotlin"  // Text("Hello, $name") will redraw!
```

---

### 5. Recomposition — Smart Redrawing

Compose is **smart**. When state changes, it only redraws the composables that **actually use** that state — not the whole screen.

```
State: count = 5
         │
         ▼
Only Counter() redraws
Header() stays the same ← not touched
```

---

### 6. `Modifier` — Style and Layout

`Modifier` is like CSS for Compose. You chain it to add padding, size, background color, click behavior, etc.

```kotlin
Text(
    text = "Hello",
    modifier = Modifier
        .padding(16.dp)         // spacing inside
        .background(Color.Blue) // background color
        .fillMaxWidth()         // take full width
        .clickable { }          // make it tappable
)
```

---

## 💡 Full Practical Example

### A simple screen with a counter button

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            // Wrap in MaterialTheme for proper styling
            MaterialTheme {
                CounterScreen()
            }
        }
    }
}

@Composable
fun CounterScreen() {
    // State — remember saves value between recompositions
    var count by remember { mutableStateOf(0) }

    // Column = vertical layout (like LinearLayout vertical)
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(32.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text(
            text = "You pressed $count times",
            fontSize = 24.sp
        )

        Spacer(modifier = Modifier.height(16.dp)) // empty gap

        Button(onClick = { count++ }) {
            Text(text = "Press Me")
        }

        Spacer(modifier = Modifier.height(8.dp))

        Button(
            onClick = { count = 0 },
            colors = ButtonDefaults.buttonColors(containerColor = Color.Red)
        ) {
            Text(text = "Reset")
        }
    }
}
```

### Preview in Android Studio

```kotlin
// No need to run the app to see the UI!
@Preview(showBackground = true)
@Composable
fun CounterScreenPreview() {
    MaterialTheme {
        CounterScreen()
    }
}
```

> ✅ `@Preview` lets you see the UI live in Android Studio's canvas — no emulator needed!

---

## 🔧 Setup — Add Compose to Your Project

**`build.gradle (app)`**
```groovy
android {
    buildFeatures {
        compose = true
    }
    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.1"
    }
}

dependencies {
    implementation(platform("androidx.compose:compose-bom:2024.02.00"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.ui:ui-tooling-preview")
    debugImplementation("androidx.compose.ui:ui-tooling")
    implementation("androidx.activity:activity-compose:1.8.2")
}
```

---

## 📊 Quick Comparison Summary

| Old (XML + View System) | New (Jetpack Compose) |
|---|---|
| Separate XML layout file | All UI in Kotlin code |
| `setContentView(R.layout.x)` | `setContent { MyScreen() }` |
| `findViewById<TextView>(R.id.tv)` | No IDs needed |
| Manual `tvName.text = "x"` | State change auto-updates UI |
| `ConstraintLayout` XML (100+ lines) | `Column`, `Row`, `Box` (5-10 lines) |
| XML Preview in layout editor | `@Preview` in code |
| 2 files per screen (XML + Kotlin) | 1 file per screen |
