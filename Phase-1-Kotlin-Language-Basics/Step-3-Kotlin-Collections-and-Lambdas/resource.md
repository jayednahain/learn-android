# Step 3 — Kotlin Collections & Lambdas

---

## 📖 What Is It? (Simple Definition)

Imagine you have a bag of toys. The **collection** is the bag, and the **toys** are the items inside. Kotlin gives you different types of bags:
- A list 📋 (items in order, can repeat)
- A set 🎯 (items in order, no repeats)
- A map 🗺️ (items with labels/keys, like a dictionary)

**Lambdas** are like tiny disposable machines. Instead of building a whole factory (class/method) to do one small job, you just write a quick instruction right there.

---

## 🎯 Where Do We Use It?

- Everywhere you deal with lists of data (users, products, messages)
- In RecyclerView adapters
- Filtering search results
- Transforming API data into UI models
- Button click listeners (a lambda!)
- Coroutines and Flow

---

## 🔄 Workflow — How Collections Flow in Android

```
API Response (JSON list)
        ↓
  Kotlin List<ApiModel>
        ↓
  .filter { ... }       ← keep only what you need
  .map { ... }          ← transform shape
        ↓
  List<UiModel>
        ↓
  RecyclerView Adapter  ← shows on screen
```

---

## ☕ Java vs Kotlin — What Changed?

### 1. Creating Collections
```java
// Java
List<String> names = new ArrayList<>(Arrays.asList("Alice", "Bob", "Charlie"));
Map<String, Integer> scores = new HashMap<>();
scores.put("Alice", 90);
```
```kotlin
// Kotlin — clean factory functions
val names = listOf("Alice", "Bob", "Charlie")    // read-only
val mutableNames = mutableListOf("Alice", "Bob") // can add/remove

val scores = mapOf("Alice" to 90, "Bob" to 85)   // read-only
val mutableScores = mutableMapOf("Alice" to 90)   // can add/remove

val uniqueItems = setOf("a", "b", "a")            // {"a", "b"} — no duplicates
```

---

### 2. Lambda Expressions
```java
// Java — anonymous inner class (old way, very verbose)
button.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        System.out.println("Clicked!");
    }
});
// Java 8 lambda
button.setOnClickListener(v -> System.out.println("Clicked!"));
```
```kotlin
// Kotlin lambda
button.setOnClickListener { println("Clicked!") }

// Lambda with parameter
button.setOnClickListener { view -> println("Clicked: $view") }

// Lambda stored in a variable
val double: (Int) -> Int = { number -> number * 2 }
println(double(5))  // 10

// "it" — automatic name for single parameter
val triple: (Int) -> Int = { it * 3 }
println(triple(5))  // 15
```
> 💡 `it` is Kotlin's shortcut. When a lambda has **one** parameter, you can just call it `it` instead of naming it.

---

### 3. Higher-Order Functions — The Power Tools

#### `map` — Transform every item
```java
// Java — need a loop and a new list
List<String> upper = new ArrayList<>();
for (String name : names) {
    upper.add(name.toUpperCase());
}
```
```kotlin
// Kotlin — one line!
val upper = names.map { it.uppercase() }  // ["ALICE", "BOB", "CHARLIE"]
```

#### `filter` — Keep only matching items
```java
// Java — loop + if
List<String> longNames = new ArrayList<>();
for (String name : names) {
    if (name.length() > 3) longNames.add(name);
}
```
```kotlin
// Kotlin — one line!
val longNames = names.filter { it.length > 3 }  // ["Alice", "Charlie"]
```

#### `forEach` — Loop over items
```kotlin
names.forEach { println(it) }
names.forEachIndexed { index, name -> println("$index: $name") }
```

#### `find` — Get first matching item
```kotlin
val found = names.find { it.startsWith("B") }  // "Bob"
val notFound = names.find { it.startsWith("Z") } // null
```

#### `any` and `all` — Check conditions
```kotlin
val hasAlice = names.any { it == "Alice" }   // true
val allShort = names.all { it.length < 10 }  // true
val noneEmpty = names.none { it.isEmpty() }  // true
```

#### Chaining — The Real Power
```java
// Java — ugly nested loops and conditionals
```
```kotlin
// Kotlin — read like English!
val result = names
    .filter { it.length > 3 }      // keep long names
    .map { it.uppercase() }        // make them uppercase
    .sorted()                      // sort alphabetically

// Result: ["ALICE", "CHARLIE"]
```

---

### 4. Extension Functions — Add Methods to Existing Classes
```kotlin
// Add a function to String class (even though you don't own String!)
fun String.capitalizeWords(): String {
    return split(" ").joinToString(" ") { it.replaceFirstChar { c -> c.uppercase() } }
}

"hello world android".capitalizeWords()  // "Hello World Android"

// Add to List
fun List<Int>.average(): Double = sum().toDouble() / size
listOf(80, 90, 70).average()  // 80.0

// In Android you'll see extensions everywhere:
fun Context.showToast(message: String) {
    Toast.makeText(this, message, Toast.LENGTH_SHORT).show()
}
// Later in Activity:
showToast("Hello!")  // instead of Toast.makeText(this, ...).show()
```

---

### 5. Scope Functions — `let`, `run`, `apply`, `also`, `with`

These are like small helper blocks. They reduce repetition.

```kotlin
// let — transform a nullable value safely
val name: String? = "Jayed"
val length = name?.let { it.length }  // length = 5, if name was null → null

// apply — configure an object and return it (builder pattern)
val textView = TextView(context).apply {
    text = "Hello"
    textSize = 18f
    setTextColor(Color.BLACK)
}
// Without apply:
// val textView = TextView(context)
// textView.text = "Hello"
// textView.textSize = 18f
// textView.setTextColor(Color.BLACK)

// also — do something extra (like logging) and return the same object
val user = User("Jayed", 28).also {
    println("Created user: ${it.name}")
}

// with — call multiple methods on same object, no need to repeat name
with(binding.textView) {
    text = "Hello"
    textSize = 20f
    visibility = View.VISIBLE
}

// run — like let but uses "this" instead of "it"
val summary = user.run {
    "Name: $name, Age: $age"  // 'this' is user
}
```

---

## 🔑 Key Concepts

| Concept | What It Does | Memory Trick |
|---------|-------------|-------------|
| `listOf()` | Read-only list | "I can read, not change" |
| `mutableListOf()` | List you can change | "I can add/remove" |
| `map {}` | Transform each item | "Change shape of each item" |
| `filter {}` | Keep matching items | "Coffee filter — keep good, remove bad" |
| `find {}` | Find first match | "Find the first one" |
| `any {}` | True if at least one matches | "Any one?" |
| `all {}` | True if every item matches | "All of them?" |
| `it` | Auto name for single lambda param | "It" = the current item |
| `apply {}` | Configure & return same object | "Apply settings" |
| `let {}` | Transform value, safe for nulls | "Let me use it" |

---

## 💡 Good Example — Real Android Use Case

```kotlin
data class Product(
    val name: String,
    val price: Double,
    val category: String,
    val inStock: Boolean
)

val products = listOf(
    Product("Kotlin Book",   25.0,  "Education", true),
    Product("Android Phone", 500.0, "Electronics", true),
    Product("Java Book",     20.0,  "Education",   false),
    Product("Tablet",        300.0, "Electronics", true),
    Product("Headphones",    80.0,  "Electronics", false)
)

// 1. Find all in-stock electronics under $400
val affordableElectronics = products
    .filter { it.category == "Electronics" }
    .filter { it.inStock }
    .filter { it.price < 400 }
    .sortedBy { it.price }
    .map { "${it.name} - $${it.price}" }

println(affordableElectronics)
// [Headphones - $80.0, Tablet - $300.0]  ← wait, Headphones is out of stock
// Actually: [Tablet - $300.0]

// 2. Check if any book is available
val hasBook = products.any { it.category == "Education" && it.inStock }
println("Books available: $hasBook")  // true (Kotlin Book)

// 3. Total price of in-stock items
val totalStock = products
    .filter { it.inStock }
    .sumOf { it.price }
println("Total inventory value: $$totalStock")  // $825.0

// 4. Group by category
val byCategory = products.groupBy { it.category }
byCategory.forEach { (category, items) ->
    println("$category: ${items.map { it.name }}")
}
// Education: [Kotlin Book, Java Book]
// Electronics: [Android Phone, Tablet, Headphones]

// 5. Extension function for discount
fun List<Product>.withDiscount(percent: Double) =
    map { it.copy(price = it.price * (1 - percent / 100)) }

val discounted = products.withDiscount(10.0)
println(discounted.first().price)  // 22.5 (Kotlin Book at 10% off)
```
