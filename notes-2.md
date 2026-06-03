# Kotlin Learning Notes — Jetpack Compose & Language Basics

---

## 1. Size Units in Jetpack Compose

When building UI in Compose, you'll use three main units:

| Unit | Full Name | Use For |
|------|-----------|---------|
| `dp` | Density-independent Pixels | Layouts, sizing, padding, spacing |
| `sp` | Scale-independent Pixels | Font sizes only |
| `em` | Relative to font size | Letter spacing only |

### `dp` — Density-independent Pixels

Used for anything related to layout — width, height, padding, margin.
Scales based on screen density so it **looks the same size on all screens**.

```kotlin
Modifier.width(200.dp)
Modifier.height(50.dp)
Modifier.padding(16.dp)
```

### `sp` — Scale-independent Pixels

Same as `dp`, but also respects the user's **system font size setting**.
Always use `sp` for font sizes.

```kotlin
Text(
    "Hello!",
    fontSize = 20.sp
)
```

### `em` — Relative Unit

`1.em` equals the current font size. Only used for **letter spacing** in Compose.

```kotlin
Text(
    "Hello!",
    fontSize = 20.sp,
    letterSpacing = 0.1.em
)
```

> ❌ **Common mistake:** Using `TextUnitType.Em` for `fontSize` causes an error.
> `Em` is only valid for letter spacing.

```kotlin
// ❌ Wrong
fontSize = TextUnit(20.0f, TextUnitType.Em)

// ✅ Correct
fontSize = TextUnit(20.0f, TextUnitType.Sp)

// ✅ Even better (shorthand)
fontSize = 20.sp
```

---

## 2. Alignment vs Arrangement in Compose

### Alignment — *Where items are positioned*

Controls how items are placed on the **cross axis** (perpendicular direction).

```kotlin
// In Column → controls horizontal position
Alignment.Start
Alignment.CenterHorizontally
Alignment.End

// In Row → controls vertical position
Alignment.Top
Alignment.CenterVertically
Alignment.Bottom
```

### Arrangement — *How items are spaced*

Controls how items are distributed along the **main axis** (flow direction).

```kotlin
Arrangement.Start         // pack items to start
Arrangement.Center        // pack items to center
Arrangement.End           // pack items to end
Arrangement.SpaceBetween  // equal space BETWEEN items
Arrangement.SpaceAround   // equal space AROUND items
Arrangement.SpaceEvenly   // equal space everywhere
```

### Visual Guide

```
SpaceBetween:  [A]-------[B]-------[C]
SpaceAround:   ---[A]----[B]----[C]---
SpaceEvenly:   ----[A]---[B]---[C]----
Start:         [A][B][C]--------------
Center:        ------[A][B][C]--------
End:           --------------[A][B][C]
```

### Quick Reference Table

| Parameter | Used In | Correct Type | Example |
|-----------|---------|--------------|---------|
| `horizontalAlignment` | `Column` | `Alignment.Horizontal` | `Alignment.CenterHorizontally` |
| `verticalArrangement` | `Column` | `Arrangement.Vertical` | `Arrangement.Center` |
| `verticalAlignment` | `Row` | `Alignment.Vertical` | `Alignment.CenterVertically` |
| `horizontalArrangement` | `Row` | `Arrangement.Horizontal` | `Arrangement.Center` |

### Full Example

```kotlin
Column(
    modifier = Modifier
        .fillMaxSize()
        .background(Color.Green),
    horizontalAlignment = Alignment.CenterHorizontally,
    verticalArrangement = Arrangement.SpaceBetween
) {
    Text("Item 1", fontSize = 20.sp)
    Text("Item 2", fontSize = 20.sp)
    Text("Item 3", fontSize = 20.sp)
}
```

> **Simple rule:**
> - `Alignment` → position on the **cross axis**
> - `Arrangement` → spacing on the **main axis**

---

## 3. Extension Properties in Kotlin

### What is an Extension Property?

Kotlin lets you **add new properties to existing types** — without modifying the original class.
This is called an **Extension Property**.

```
Normal property:    person.name      → defined INSIDE the class
Extension property: person.fullName  → defined OUTSIDE the class
```

### How `200.dp` Actually Works

`200` is just an `Int`. Kotlin allows adding properties to `Int` from outside.

Internally, Compose defines `.dp` like this:

```kotlin
val Int.dp: Dp get() = Dp(this.toFloat())
```

So `200.dp` is just a clean way of writing `Dp(200f)`.

```kotlin
200.dp    // → Dp(200f)
20.sp     // → TextUnit(20f, TextUnitType.Sp)
0.1.em    // → TextUnit(0.1f, TextUnitType.Em)
```

Everything is an **object** — `.dp`, `.sp`, `.em` are all extension properties that create objects.

---

### Extension Property Examples

#### Example 1 — Check if a number is even or odd

```kotlin
val Int.isEven: Boolean get() = this % 2 == 0
val Int.isOdd: Boolean get() = this % 2 != 0

println(4.isEven)  // true
println(7.isOdd)   // true
```

#### Example 2 — String utilities

```kotlin
val String.isPalindrome: Boolean get() = this == this.reversed()
val String.wordCount: Int get() = this.trim().split(" ").size

println("racecar".isPalindrome)   // true
println("hello world".wordCount)  // 2
```

#### Example 3 — Add property to your own class

```kotlin
data class Person(val firstName: String, val lastName: String)

val Person.fullName: String get() = "$firstName $lastName"

val person = Person("John", "Doe")
println(person.fullName)  // John Doe
```

#### Example 4 — List utilities

```kotlin
val List<Int>.sum: Int get() = this.fold(0) { acc, n -> acc + n }
val List<Int>.average: Double get() = this.sum.toDouble() / this.size

val numbers = listOf(10, 20, 30)
println(numbers.sum)      // 60
println(numbers.average)  // 20.0
```

### Key Points

- You are **not modifying** the original class
- You are **adding new properties** from outside
- Works on **any type** — your own classes or built-in types like `Int`, `String`, `List`
- This is exactly how `200.dp`, `20.sp`, and `0.1.em` work in Compose!

---

## Summary

| Topic | Key Takeaway |
|-------|--------------|
| `dp` | Use for all layout sizing |
| `sp` | Use only for font sizes |
| `em` | Use only for letter spacing |
| `Alignment` | Positions items on the cross axis |
| `Arrangement` | Spaces items on the main axis |
| Extension Properties | Add properties to any type from outside the class |
| `200.dp` | Is `Int.dp` extension property returning a `Dp` object |