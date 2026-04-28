# Step 28 — Compose State Management

---

## 📖 What Is It? (Simple Explanation)

Imagine a whiteboard 📋 in a classroom. Every time the teacher changes what's written on it (the **state**), all students (the **UI**) look up and see the new information.

In Compose, **State** is data that can **change over time** and when it changes, the **UI automatically redraws** itself to show the new value. You never manually say "update this text view" — Compose watches the state and does it for you.

---

## 🧭 Where Do We Use It?

- Any time your UI needs to **react to changes** (user typing, button clicks, data loading)
- Counter screens, form inputs, toggle switches, tabs
- Connecting `ViewModel` data to the Compose UI
- Loading / error / success states

---

## 🗺️ Workflow / How It Works

```
User clicks Button
        │
        ▼
State variable changes (count = count + 1)
        │
        ▼
Compose detects the change (via mutableStateOf)
        │
        ▼
Only the Composables that READ this state recompose
        │
        ▼
Screen updates — Text shows new count
```

---

## ☕ Java vs Kotlin/Compose — What Changed?

### Old Way (Java XML — manual UI updates)

```java
// Java — you manually update every view yourself
int count = 0;
Button btn = findViewById(R.id.btn);
TextView tv = findViewById(R.id.tv);

btn.setOnClickListener(v -> {
    count++;
    tv.setText("Count: " + count);  // ← YOU must do this manually
});
```

### Compose Way — state drives UI automatically

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    // Change count → Text updates automatically, no manual call needed
    Column {
        Text("Count: $count")
        Button(onClick = { count++ }) { Text("Increment") }
    }
}
```

> No `setText()`. No manual view update. State changes → UI reacts. ✅

---

## 🔑 Key Concepts

---

### 1. `mutableStateOf` — The Reactive Variable

```kotlin
var name = mutableStateOf("John")
// Change it:
name.value = "Jane"   // UI that reads 'name' will redraw

// Delegates (cleaner with 'by'):
var name by mutableStateOf("John")
name = "Jane"  // same but no .value needed
```

---

### 2. `remember` — Keep State Alive During Recomposition

Compose re-runs composable functions often. Without `remember`, your variable resets every time. `remember` stores it in Compose's memory.

```kotlin
@Composable
fun Example() {
    // WITHOUT remember — resets to 0 every recomposition ❌
    var count = mutableStateOf(0)

    // WITH remember — survives recompositions ✅
    var count by remember { mutableStateOf(0) }
}
```

---

### 3. `rememberSaveable` — Survives Screen Rotation

`remember` is lost when the screen rotates (like an activity restart). `rememberSaveable` saves it to the instance state — just like `onSaveInstanceState` in the old days.

```kotlin
var count by rememberSaveable { mutableStateOf(0) }
// ✅ Survives rotation, process death
```

| | `remember` | `rememberSaveable` |
|---|---|---|
| Survives recomposition | ✅ | ✅ |
| Survives screen rotation | ❌ | ✅ |

---

### 4. State Hoisting — The Golden Rule

**Don't keep state inside a composable if two composables need it.** Move ("hoist") it **up** to the parent so both children can access it.

```
❌ Problem — both siblings need 'name' but it's stuck in Child A:

Parent()
  ├── ChildA()  ← has 'name' state
  └── ChildB()  ← also needs 'name'... can't reach it!

✅ Solution — hoist state to Parent:

Parent()  ← holds 'name', passes it down
  ├── ChildA(name, onNameChange)
  └── ChildB(name)
```

```kotlin
// ✅ Hoisted state pattern
@Composable
fun ParentScreen() {
    var name by remember { mutableStateOf("") }
    Column {
        NameInput(name = name, onNameChange = { name = it })
        NameDisplay(name = name)
    }
}

@Composable
fun NameInput(name: String, onNameChange: (String) -> Unit) {
    TextField(value = name, onValueChange = onNameChange, label = { Text("Name") })
}

@Composable
fun NameDisplay(name: String) {
    Text(text = "Hello, $name!")
}
```

> Rule: **State lives as high as it needs to, but no higher.**

---

### 5. `ViewModel` + `StateFlow` with Compose

For real apps, don't keep complex state in composables. Put it in `ViewModel` and expose it via `StateFlow`. Compose observes it with `collectAsStateWithLifecycle()`.

```kotlin
// ViewModel
@HiltViewModel
class CounterViewModel @Inject constructor() : ViewModel() {
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count.asStateFlow()

    fun increment() { _count.value++ }
    fun reset() { _count.value = 0 }
}

// Composable
@Composable
fun CounterScreen(viewModel: CounterViewModel = hiltViewModel()) {
    val count by viewModel.count.collectAsStateWithLifecycle()

    Column {
        Text("Count: $count")
        Button(onClick = { viewModel.increment() }) { Text("Add") }
        Button(onClick = { viewModel.reset() }) { Text("Reset") }
    }
}
```

---

### 6. UI State Data Class Pattern

Instead of many separate `StateFlow` variables, wrap your entire screen state in one data class.

```kotlin
// Data class representing the full screen state
data class LoginUiState(
    val email: String = "",
    val password: String = "",
    val isLoading: Boolean = false,
    val errorMessage: String? = null,
    val isLoggedIn: Boolean = false
)

// ViewModel
class LoginViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(LoginUiState())
    val uiState: StateFlow<LoginUiState> = _uiState.asStateFlow()

    fun onEmailChanged(email: String) {
        _uiState.update { it.copy(email = email) }
    }

    fun onPasswordChanged(password: String) {
        _uiState.update { it.copy(password = password) }
    }

    fun login() {
        _uiState.update { it.copy(isLoading = true) }
        viewModelScope.launch {
            // call API...
            _uiState.update { it.copy(isLoading = false, isLoggedIn = true) }
        }
    }
}
```

---

### 7. Side Effects — `LaunchedEffect`, `DisposableEffect`

Sometimes you need to do something **when** a composable appears, like loading data or playing a sound. That's a **side effect**.

#### `LaunchedEffect` — run a coroutine when a key changes

```kotlin
@Composable
fun UserScreen(userId: String, viewModel: UserViewModel) {
    // Runs when userId changes (or when first composed)
    LaunchedEffect(userId) {
        viewModel.loadUser(userId)
    }

    val user by viewModel.user.collectAsStateWithLifecycle()
    Text("User: ${user?.name}")
}
```

#### `DisposableEffect` — run code and clean up when composable leaves

```kotlin
@Composable
fun EventListener(activity: Activity) {
    DisposableEffect(Unit) {
        val listener = object : SomeListener {
            override fun onEvent() { /* handle */ }
        }
        activity.addListener(listener)
        
        onDispose {
            activity.removeListener(listener) // cleanup!
        }
    }
}
```

---

## 💡 Full Practical Example — Login Form

```kotlin
data class LoginUiState(
    val email: String = "",
    val password: String = "",
    val isLoading: Boolean = false,
    val error: String? = null
)

class LoginViewModel : ViewModel() {
    private val _state = MutableStateFlow(LoginUiState())
    val state = _state.asStateFlow()

    fun onEmailChange(v: String) = _state.update { it.copy(email = v) }
    fun onPasswordChange(v: String) = _state.update { it.copy(password = v) }

    fun login() {
        _state.update { it.copy(isLoading = true, error = null) }
        viewModelScope.launch {
            delay(1500) // simulate API call
            if (_state.value.email.isBlank()) {
                _state.update { it.copy(isLoading = false, error = "Email required") }
            } else {
                _state.update { it.copy(isLoading = false) }
                // navigate to home
            }
        }
    }
}

@Composable
fun LoginScreen(viewModel: LoginViewModel = viewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    Column(
        modifier = Modifier.fillMaxSize().padding(32.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text("Login", fontSize = 28.sp, fontWeight = FontWeight.Bold)
        Spacer(Modifier.height(24.dp))

        OutlinedTextField(
            value = state.email,
            onValueChange = { viewModel.onEmailChange(it) },
            label = { Text("Email") },
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(12.dp))

        OutlinedTextField(
            value = state.password,
            onValueChange = { viewModel.onPasswordChange(it) },
            label = { Text("Password") },
            visualTransformation = PasswordVisualTransformation(),
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(8.dp))

        state.error?.let { errorMsg ->
            Text(errorMsg, color = Color.Red, fontSize = 14.sp)
        }
        Spacer(Modifier.height(16.dp))

        if (state.isLoading) {
            CircularProgressIndicator()
        } else {
            Button(
                onClick = { viewModel.login() },
                modifier = Modifier.fillMaxWidth()
            ) {
                Text("Login")
            }
        }
    }
}
```

---

## 📊 Quick Comparison Summary

| Old (Java/XML + ViewModel) | Compose State |
|---|---|
| `tvName.text = "x"` manual update | `var name by mutableStateOf("x")` auto-updates |
| `onSaveInstanceState` for rotation | `rememberSaveable` |
| `LiveData.observe(this) { }` | `collectAsStateWithLifecycle()` |
| Separate `observe` callbacks | Inline state reading in composable |
| Multiple `liveData` fields | One `UiState` data class pattern |
| `LaunchWhenStarted` for coroutines | `LaunchedEffect(key)` |
