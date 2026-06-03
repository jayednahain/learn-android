# Kotlin + Jetpack Compose Learning Notes 🚀

---

## 1. MainActivity : ComponentActivity()

`MainActivity` is **extending** (inheriting from) `ComponentActivity`.
In Kotlin `:` means extends. In Java this would be `extends`.

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)  // call parent setup first
        enableEdgeToEdge()                  // fullscreen setup
        setContent {                        // entry point for Compose UI
            // your UI goes here
        }
    }
}
```

### Mental Model
```
Android OS
    └── calls onCreate()
            └── enableEdgeToEdge()   → fullscreen setup
            └── setContent { }       → your UI goes here
```

---

## 2. setContent { } — Function Call vs Definition

`setContent { }` is a **function call**, not a definition.
The import just brings the function from the library so you can use it.

```kotlin
import androidx.activity.compose.setContent  // bring the function
// then call it:
setContent { }                               // use the function
```

### Trailing Lambda
If the last parameter of a function is a lambda, you can move `{ }` outside `()`:

```kotlin
setContent({ })   // normal way
setContent { }    // trailing lambda way (cleaner) — same thing!
```

---

## 3. Lambdas / Callbacks — `() -> Unit`

```kotlin
() -> Unit          // takes nothing, returns nothing
(String) -> Unit    // takes a String, returns nothing
(Int, Int) -> Int   // takes two Ints, returns an Int
(String) -> Boolean // takes a String, returns a Boolean
```

### Example — basic callback
```kotlin
fun doSomething(action: () -> Unit) {
    println("Before action")
    action()   // calling the callback
    println("After action")
}

doSomething {
    println("I am doing something!")
}
// Output:
// Before action
// I am doing something!
// After action
```

### Example — lambda with params and return
```kotlin
fun calculate(a: Int, b: Int, action: (Int, Int) -> Int) {
    val result = action(a, b)
    println("Result is: $result")
}

calculate(10, 5) { a, b -> a + b }  // Result is: 15
calculate(10, 5) { a, b -> a - b }  // Result is: 5
calculate(10, 5) { a, b -> a * b }  // Result is: 50
```

### Mental Model
```
{ }  →  a block of code you pass into a function
         the function decides WHEN to run it
         That is exactly what a callback is! 🎯
```

---

## 4. Modifier — Method Chaining

`Modifier` is a class. Using `.` (dot) you access its methods.
Each method **returns a Modifier back**, which allows chaining.

```kotlin
Modifier
    .fillMaxSize()         // returns Modifier
    .background(Color.Red) // called on result above, returns Modifier
    .padding(16.dp)        // called on result above, returns Modifier
```

### Why chaining works
```kotlin
// Imagine each method returns this:
fun fillMaxSize(): Modifier { return this }
fun background(color: Color): Modifier { return this }
```

### OOP Rule — Method Chaining / Fluent Interface
```
Raw Modifier
    → add fullscreen
    → add red background
    → add padding
    → Final Modifier delivered to Box 📦
```

---

## 5. Modifier — Order Matters! ⚠️

Each modifier operates on the **result of the previous one**.
Always think **outside → inside**.

```kotlin
// ❌ Wrong — clip does NOT work
Modifier
    .background(Color.Blue)  // paints first
    .clip(CircleShape)        // clips AFTER painting (too late!)
    .size(100.dp)

// ✅ Correct
Modifier
    .size(100.dp)             // 1. set size
    .clip(CircleShape)        // 2. clip the shape
    .background(Color.Blue)   // 3. paint INSIDE the clipped shape
```

### background vs padding order
```kotlin
// ❌ color misses the padded area
Modifier
    .padding(16.dp)
    .background(Color.Red)

// ✅ color covers everything including padding
Modifier
    .background(Color.Red)
    .padding(16.dp)
```

### Cheat Sheet — Recommended Order
```kotlin
Modifier
    // 1️⃣ Size & Position
    .size(100.dp)
    .fillMaxSize()
    .padding(16.dp)
    // 2️⃣ Shape
    .clip(CircleShape)
    .clip(RoundedCornerShape(16.dp))
    // 3️⃣ Visual Decoration
    .background(Color.Blue)
    .border(2.dp, Color.Red)
    // 4️⃣ Interaction
    .clickable { }
```

### Mental Model
```
Modifier order = Painting a portrait

1. Draw the CANVAS SIZE   → size()
2. Cut canvas into SHAPE  → clip()
3. PAINT on that shape    → background()

Paint BEFORE cutting = paint goes outside the shape! ❌
```

---

## 6. Other Places Where Order Matters

### Composable UI — top to bottom
```kotlin
Column {
    Text("I appear FIRST")   // renders on top
    Text("I appear SECOND")  // renders below
}
```

### if/else — specific conditions first
```kotlin
// ✅ Correct
if (number == 0) {
    println("zero")
} else if (number > 0) {
    println("positive")
} else {
    println("negative")
}
```

### Null checks — always check before accessing
```kotlin
// ✅ Correct
if (name != null) {
    println(name.length)  // safe!
}
```

### State variables — declare at top
```kotlin
@Composable
fun MyScreen() {
    var count by remember { mutableStateOf(0) }  // declare first
    var name by remember { mutableStateOf("") }   // declare first

    Text("Count: $count")                          // then use
    Button(onClick = { count++ }) { Text("Click") }
}
```

---

## 7. Mandatory vs Optional Parameters

```kotlin
// Mandatory - NO default value, MUST provide or compiler error ❌
fun border(width: Dp, color: Color)

// Optional - HAS default value, can skip ✅
fun border(width: Dp = 1.dp, color: Color = Color.Black)
```

### Example
```kotlin
fun createUser(
    name: String,               // ❗ mandatory
    age: Int,                   // ❗ mandatory
    city: String = "NYC",       // ✅ optional
    isActive: Boolean = true    // ✅ optional
) { }

createUser("John", 25)                  // ✅ only mandatory
createUser("John", 25, city = "LA")     // ✅ one optional override
createUser("John", 25, "LA", false)     // ✅ all provided
```

### Mental Model
```
No default value  =  Mandatory  →  Compiler FORCES you ❗
Has default value =  Optional   →  Compiler lets you skip ✅
```

---

## 8. Why Separate Imports?

Each function/class lives in a **different package (folder)** in the library.

```kotlin
import androidx.compose.foundation.border          // foundation dept
import androidx.compose.foundation.layout.Box      // layout dept
import androidx.compose.ui.draw.clip               // drawing dept
import androidx.compose.ui.graphics.Color          // graphics dept
import androidx.compose.foundation.layout.size     // layout dept
```

### Wildcard import (use carefully)
```kotlin
import androidx.compose.foundation.layout.*  // imports everything from layout
// ⚠️ Can cause naming conflicts between packages
```

### Mental Model
```
Package  =  a folder/department in the library

import = "bring the tool from the toolbox" 🧰
calling = "use that tool" 🔨

Android Studio auto-imports for you!
Just press Alt+Enter (Win) or Option+Enter (Mac) ✅
```

---

## 9. Null Safety — `Int` vs `Int?`

In Kotlin **nothing can be null unless you add `?`**.

```kotlin
Int     // CANNOT hold null ❌
Int?    // CAN hold null    ✅

String  // CANNOT hold null ❌
String? // CAN hold null    ✅
```

### Example
```kotlin
// ✅ Correct nullable parameter
fun createUser(name: String = "test", age: Int? = null) {

    // ✅ Way 1 - null check
    if (age != null) {
        val nextYear = age + 1
    }

    // ✅ Way 2 - Elvis operator ?: (use default if null)
    val safeAge = age ?: 0

    // ✅ Way 3 - safe call ?.
    Log.i("tag", "$name is ${age?.toString() ?: "unknown"} years old")
}
```

### Mental Model
```
Int    = 📦 box that MUST have a number
Int?   = 📦 box that can have number OR be empty

? = "this box is allowed to be empty"
```

---

## 10. `object` vs `class` — Companion Object

```kotlin
// object = singleton, already built, use directly
object CircleShape : Shape { }
.clip(CircleShape)              // ✅ no () needed

// class = blueprint, must create an instance with ()
class RoundedCornerShape(val corner: Dp) : Shape { }
.clip(RoundedCornerShape(10.dp))  // ✅ must use ()
.clip(RoundedCornerShape)         // ❌ error! no companion object
```

### Mental Model
```
object  →  already built house 🏠  → use directly
class   →  blueprint           📄  → must build with ()

CircleShape          → 🏠 ready!
RoundedCornerShape   → 📄 build it! → RoundedCornerShape(10.dp) 🏠
```

---

## 11. Reading Android Studio Error Popups

```
internal constructor TextUnit(
─────────  ──────────────────
    │               │
    │               └── constructor signature (params & types)
    │
    └── access modifier
        internal = locked 🔒
        public   = you can use ✅
```

### Always read the description in the popup!
```
"It can be created with sp or em (e.g. 15.sp or 18.em)"
 ───────────────────────────────────────────────────────
 👆 The popup is literally telling you the correct way!
```

---

## 12. Text fontSize — Use `.sp`

```kotlin
// ❌ Wrong — TextUnit constructor is internal (locked)
fontSize = TextUnit(20.em)

// ✅ Correct — simple shortcut
fontSize = 20.sp    // best for text! respects accessibility settings
fontSize = 20.em    // relative to parent font size
```

### Why `.sp` is best for text
```
sp  →  Scale Pixels     →  respects user's phone font size settings ✅
em  →  relative unit    →  relative to parent (be careful!)
dp  →  Density Pixels   →  use for layout, NOT for font size ❌
```

---

## 13. OOP Access Modifiers

```kotlin
class BankAccount(val owner: String) {
    var balance: Int = 1000          // public    - everyone
    internal var tax: Int = 0        // internal  - same app only
    protected var type = "Basic"     // protected - this class + children
    private var pin: Int = 1234      // private   - this class ONLY
}
```

### Who can access what?
```
                        public  internal  protected  private
────────────────────    ──────  ────────  ─────────  ───────
Same class              ✅      ✅        ✅         ✅
Child class             ✅      ✅        ✅         ❌
Same app/module         ✅      ✅        ❌         ❌
Outside app/library     ✅      ❌        ❌         ❌
```

### Mental Model
```
private   = 🔒 your diary    (only you)
protected = 🏠 your house    (you + family/children)
internal  = 🏢 your company  (you + same app colleagues)
public    = 🌍 the internet  (everyone)
```

---

## 🏆 Golden Rules Summary

| # | Rule |
|---|---|
| 1 | `super.onCreate()` always first in lifecycle methods |
| 2 | Trailing lambda `{ }` = last param moved outside `()` |
| 3 | Modifier order = **outside → inside** (size → clip → background) |
| 4 | `background` before `padding` to cover full area |
| 5 | Specific `if/else` conditions first, general ones last |
| 6 | Always declare state variables at top of Composable |
| 7 | No default value = mandatory param, has default = optional |
| 8 | No `?` = never null, add `?` = nullable, must handle safely |
| 9 | `object` = use directly, `class` = must create with `()` |
| 10 | Always use `.sp` for font sizes |
| 11 | `private/internal` in error popup = use the public API instead |
| 12 | Read the description in Android Studio popups — it shows the fix! |
