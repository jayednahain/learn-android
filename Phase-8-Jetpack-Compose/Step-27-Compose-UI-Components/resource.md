# Step 27 — Compose UI Components

---

## 📖 What Is It? (Simple Explanation)

In the old XML world, you had building blocks like `TextView`, `Button`, `EditText`, `ListView`, `LinearLayout`, `FrameLayout`. Each one had its own XML tag.

In Compose, you have **Composable functions** instead. They look like function calls, not XML tags. Same idea, cleaner code.

Think of them like **LEGO bricks** 🧱 — small pieces you snap together to build any screen.

---

## 🧭 Where Do We Use It?

Every single screen you build in Compose uses these components. They replace every XML view and layout you ever wrote.

---

## 🗺️ Layout Components — The Containers

```
XML Layout  →  Compose Equivalent
────────────────────────────────────────
LinearLayout  (vertical)   →  Column
LinearLayout  (horizontal) →  Row
FrameLayout / RelativeLayout → Box
RecyclerView (vertical)    →  LazyColumn
RecyclerView (horizontal)  →  LazyRow
```

### Visual Diagram

```
Column                Row                 Box
  ┌─────┐           ┌──┬──┬──┐        ┌─────────┐
  │  A  │           │A │B │C │        │    C    │ ← on top
  ├─────┤           └──┴──┴──┘        │  B ─────│ 
  │  B  │       items side by side    │A        │ ← stacked
  ├─────┤                             └─────────┘
  │  C  │
  └─────┘
items stacked top-to-bottom
```

---

## ☕ Java/XML vs Compose — What Changed?

### Old Way — XML (`activity_main.xml` + `MainActivity.kt`)

```xml
<!-- activity_main.xml — 20 lines just for layout -->
<LinearLayout
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:padding="16dp">

    <TextView
        android:id="@+id/tvTitle"
        android:text="My Title"
        android:textSize="20sp"/>

    <EditText
        android:id="@+id/etName"
        android:hint="Enter name"/>

    <Button
        android:id="@+id/btnSubmit"
        android:text="Submit"/>

</LinearLayout>
```

```kotlin
// MainActivity.kt — wire everything up
val tv = findViewById<TextView>(R.id.tvTitle)
val et = findViewById<EditText>(R.id.etName)
val btn = findViewById<Button>(R.id.btnSubmit)
btn.setOnClickListener { tv.text = "Hello ${et.text}" }
```

### New Way — Compose (~12 lines, one place)

```kotlin
@Composable
fun MyForm() {
    var name by remember { mutableStateOf("") }
    Column(modifier = Modifier.padding(16.dp)) {
        Text(text = "My Title", fontSize = 20.sp)
        TextField(value = name, onValueChange = { name = it }, label = { Text("Enter name") })
        Button(onClick = { /* use name */ }) { Text("Submit") }
    }
}
```

---

## 🔑 Key Concepts — All the Core Components

---

### 1. `Text` — Replaces `TextView`

```kotlin
Text(
    text = "Hello, Compose!",
    fontSize = 24.sp,
    fontWeight = FontWeight.Bold,
    color = Color.DarkGray,
    textAlign = TextAlign.Center,
    maxLines = 2,
    overflow = TextOverflow.Ellipsis
)
```

---

### 2. `Button` — Replaces `Button` (but cleaner)

```kotlin
Button(
    onClick = { /* action */ },
    modifier = Modifier.fillMaxWidth(),
    colors = ButtonDefaults.buttonColors(containerColor = Color.Blue)
) {
    Text("Click Me") // Button takes a composable content lambda
}

// Outlined button (no fill)
OutlinedButton(onClick = { }) { Text("Outlined") }

// Text button (flat)
TextButton(onClick = { }) { Text("Text Button") }
```

---

### 3. `TextField` — Replaces `EditText`

```kotlin
var text by remember { mutableStateOf("") }

TextField(
    value = text,
    onValueChange = { text = it },
    label = { Text("Username") },
    placeholder = { Text("Enter your username") },
    singleLine = true,
    keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email)
)

// Material styled version
OutlinedTextField(
    value = text,
    onValueChange = { text = it },
    label = { Text("Password") },
    visualTransformation = PasswordVisualTransformation()
)
```

---

### 4. `Image` — Replaces `ImageView`

```kotlin
// From resources
Image(
    painter = painterResource(id = R.drawable.my_image),
    contentDescription = "Profile picture", // accessibility
    modifier = Modifier.size(80.dp).clip(CircleShape),
    contentScale = ContentScale.Crop
)

// From URL — use Coil library
AsyncImage(
    model = "https://example.com/photo.jpg",
    contentDescription = "Remote image"
)
```

---

### 5. `Column` — Vertical Stack (Replaces `LinearLayout vertical`)

```kotlin
Column(
    modifier = Modifier.fillMaxSize().padding(16.dp),
    verticalArrangement = Arrangement.spacedBy(8.dp), // gap between items
    horizontalAlignment = Alignment.CenterHorizontally
) {
    Text("Item 1")
    Text("Item 2")
    Text("Item 3")
}
```

---

### 6. `Row` — Horizontal Stack (Replaces `LinearLayout horizontal`)

```kotlin
Row(
    modifier = Modifier.fillMaxWidth(),
    horizontalArrangement = Arrangement.SpaceBetween,
    verticalAlignment = Alignment.CenterVertically
) {
    Text("Left")
    Text("Right")
}
```

---

### 7. `Box` — Overlapping Stack (Replaces `FrameLayout`)

```kotlin
Box(modifier = Modifier.size(200.dp)) {
    Image(painterResource(R.drawable.bg), contentDescription = null)
    Text(
        text = "On top!",
        modifier = Modifier.align(Alignment.BottomCenter)
    )
}
```

---

### 8. `LazyColumn` — Scrollable List (Replaces `RecyclerView`)

```kotlin
val items = listOf("Apple", "Banana", "Cherry", "Durian", "Elderberry")

LazyColumn(
    modifier = Modifier.fillMaxSize(),
    contentPadding = PaddingValues(16.dp),
    verticalArrangement = Arrangement.spacedBy(8.dp)
) {
    items(items) { fruit ->
        Card(modifier = Modifier.fillMaxWidth()) {
            Text(
                text = fruit,
                modifier = Modifier.padding(16.dp),
                fontSize = 18.sp
            )
        }
    }
}
```

> ✅ No `Adapter`. No `ViewHolder`. No `onBindViewHolder`. Just a loop!

---

### 9. `LazyRow` — Horizontal Scrollable List

```kotlin
LazyRow(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
    items(categoryList) { category ->
        Chip(onClick = { }) { Text(category) }
    }
}
```

---

### 10. `Scaffold` — App Structure Template

`Scaffold` gives you the standard Android app structure (top bar, bottom bar, FAB, content area) in one composable.

```kotlin
Scaffold(
    topBar = {
        TopAppBar(title = { Text("My App") })
    },
    bottomBar = {
        BottomNavigationBar()
    },
    floatingActionButton = {
        FloatingActionButton(onClick = { }) {
            Icon(Icons.Default.Add, contentDescription = "Add")
        }
    }
) { paddingValues ->
    // Main content — paddingValues keeps content from going under bars
    LazyColumn(contentPadding = paddingValues) {
        // list items
    }
}
```

---

### 11. `Card` — Material Design Card

```kotlin
Card(
    modifier = Modifier
        .fillMaxWidth()
        .padding(8.dp),
    elevation = CardDefaults.cardElevation(defaultElevation = 4.dp),
    shape = RoundedCornerShape(12.dp)
) {
    Column(modifier = Modifier.padding(16.dp)) {
        Text("Card Title", fontWeight = FontWeight.Bold)
        Text("Card description text goes here.")
    }
}
```

---

### 12. `Spacer` — Empty Gap

```kotlin
Column {
    Text("Above")
    Spacer(modifier = Modifier.height(16.dp)) // 16dp gap
    Text("Below")
}
```

---

## 💡 Full Practical Example — User Profile Card Screen

```kotlin
@Composable
fun UserProfileScreen() {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .background(Color(0xFFF5F5F5))
            .padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        // Header card
        Card(modifier = Modifier.fillMaxWidth()) {
            Row(
                modifier = Modifier.padding(16.dp),
                verticalAlignment = Alignment.CenterVertically,
                horizontalArrangement = Arrangement.spacedBy(16.dp)
            ) {
                Box(
                    modifier = Modifier
                        .size(60.dp)
                        .background(Color.Blue, CircleShape),
                    contentAlignment = Alignment.Center
                ) {
                    Text("JD", color = Color.White, fontWeight = FontWeight.Bold)
                }
                Column {
                    Text("John Doe", fontSize = 18.sp, fontWeight = FontWeight.Bold)
                    Text("Android Developer", color = Color.Gray)
                }
            }
        }

        // Skills list
        Text("Skills", fontWeight = FontWeight.Bold, fontSize = 16.sp)

        val skills = listOf("Kotlin", "Jetpack Compose", "Coroutines", "Room", "Hilt")
        LazyColumn(verticalArrangement = Arrangement.spacedBy(8.dp)) {
            items(skills) { skill ->
                Card(modifier = Modifier.fillMaxWidth()) {
                    Text(
                        text = "✅ $skill",
                        modifier = Modifier.padding(12.dp)
                    )
                }
            }
        }
    }
}
```

---

## 📊 Quick Comparison Summary

| XML / Java | Compose |
|---|---|
| `TextView` | `Text()` |
| `Button` | `Button()` |
| `EditText` | `TextField()` |
| `ImageView` | `Image()` |
| `LinearLayout (vertical)` | `Column()` |
| `LinearLayout (horizontal)` | `Row()` |
| `FrameLayout` | `Box()` |
| `RecyclerView` | `LazyColumn()` / `LazyRow()` |
| `CardView` layout | `Card()` |
| `CoordinatorLayout` + `AppBarLayout` | `Scaffold()` |
| Adapter + ViewHolder | Just `items(list) { item -> }` |
| `View.GONE` / `View.VISIBLE` | `if (show) Text(...)` |
