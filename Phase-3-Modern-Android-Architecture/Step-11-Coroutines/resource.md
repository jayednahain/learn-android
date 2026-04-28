# Step 11 — Coroutines (Replaces AsyncTask)

---

## 📖 What Is It? (Simple Definition)

Imagine you're at a restaurant. You order food (network request), but you don't stand at the counter staring at the chef (blocking the main thread). You sit down, talk to friends, and the waiter brings the food when it's ready (callback). Your evening (main thread / UI) continues without freezing!

**Coroutines** are Kotlin's way to do work in the background **without making your app freeze**. In Java, you used `AsyncTask` for this — but it was complicated and had many bugs. Coroutines are simpler, safer, and much more powerful.

---

## 🎯 Where Do We Use It?

- Fetching data from internet (network calls)
- Reading/writing to database (Room)
- Any work that takes time and shouldn't freeze the UI
- Replacing: `AsyncTask`, `Thread`, `Handler`, `RxJava`

---

## 🔄 Coroutines Flow Diagram

```
UI Thread (Main Thread)  ─────────────────────────────────────►
    │                                                    │
    │ launch - start coroutine                           │
    ↓                                                    │
 withContext(IO)  ← switches to background Thread        │
    │                                                    │
    │  [API call or DB query runs here — no UI freeze!]  │
    │                                                    │
    ↑  result comes back to Main Thread automatically    │
    →  update UI with result  ────────────────────────►  │

Compare to Java AsyncTask:
  doInBackground()  →  onPostExecute()  (same idea, much harder code)
```

```
Coroutine Scopes (who manages the coroutine's lifetime):
  viewModelScope   → cancelled when ViewModel is cleared (use in ViewModel)
  lifecycleScope   → cancelled when screen is destroyed (use in Activity/Fragment)
  GlobalScope      → lives forever (avoid! use only for app-wide tasks)
```

---

## ☕ Java vs Kotlin — What Changed?

### Old Java Way — AsyncTask (100 lines for a simple API call!)

```java
// Java AsyncTask — confusing, deprecated, dangerous
private class FetchUserTask extends AsyncTask<Integer, Void, User> {
    @Override
    protected User doInBackground(Integer... params) {
        int userId = params[0];
        try {
            return apiService.getUser(userId); // network call
        } catch (Exception e) {
            return null;
        }
    }

    @Override
    protected void onPostExecute(User user) {
        if (user != null) {
            textView.setText(user.getName()); // update UI
        }
    }
}
// Calling it:
new FetchUserTask().execute(42);
// Problems: memory leaks, no cancellation, activity rotation crashes, callbacks hell...
```

### New Kotlin Way — Coroutines (10 lines, reads like normal code!)

```kotlin
// In ViewModel:
fun fetchUser(userId: Int) {
    viewModelScope.launch {           // start coroutine (tied to ViewModel)
        val user = withContext(Dispatchers.IO) {  // switch to background thread
            apiService.getUser(userId)             // this runs in background
        }
        // Back on Main thread automatically!
        _userName.value = user.name  // update LiveData → UI updates
    }
}
```

---

### Key Coroutine Concepts

#### `suspend` — Mark a function as "can pause"

```kotlin
// suspend function = can be paused and resumed without blocking a thread
suspend fun fetchUser(id: Int): User {
    return withContext(Dispatchers.IO) {
        apiService.getUser(id)
    }
}
// You can only call suspend functions from inside a coroutine or another suspend function!
```

#### `launch` vs `async`

```kotlin
// launch — "fire and forget" — doesn't return a result
viewModelScope.launch {
    saveUserToDatabase(user)  // just do it, no result needed
}

// async — returns a result (Deferred<T>)
viewModelScope.launch {
    val userDeferred = async { fetchUser(1) }
    val postsDeferred = async { fetchPosts(1) }
    
    // Both run in PARALLEL! Then we wait for both results:
    val user = userDeferred.await()
    val posts = postsDeferred.await()
    // user and posts fetched simultaneously — twice as fast!
}
```

#### Dispatchers — Which Thread?

```kotlin
Dispatchers.Main    // Main/UI thread — update UI here
Dispatchers.IO      // Background thread — network, file, database
Dispatchers.Default // Background thread — heavy computation, sorting

// Switch threads with withContext:
viewModelScope.launch {
    // Currently on Main
    val result = withContext(Dispatchers.IO) {
        // Now on IO thread — do network/db work
        repository.getUsers()
    }
    // Back on Main automatically
    binding.text.text = result.name
}
```

#### `Flow` — Stream of Data Over Time

```kotlin
// Single value: suspend fun → returns one result
// Multiple values over time: Flow → like a pipe that keeps sending values

// Emit values over time:
fun getPriceUpdates(stockId: String): Flow<Double> = flow {
    while (true) {
        val price = stockApi.getPrice(stockId)
        emit(price)            // send value to collector
        delay(5000)            // wait 5 seconds
    }
}

// Collect values in ViewModel or Fragment:
viewModelScope.launch {
    getPriceUpdates("AAPL")
        .collect { price ->
            _stockPrice.value = price  // update StateFlow → UI updates
        }
}
```

#### `StateFlow` — Hot Stream for UI State

```kotlin
// In ViewModel — StateFlow holds the CURRENT state
private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
val uiState: StateFlow<UiState> = _uiState.asStateFlow()

// In Fragment — collect StateFlow
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state ->
            when (state) {
                is UiState.Loading -> showLoading()
                is UiState.Success -> showData(state.data)
                is UiState.Error   -> showError(state.message)
            }
        }
    }
}
```

---

### Error Handling in Coroutines

```kotlin
viewModelScope.launch {
    try {
        val user = repository.fetchUser(userId)
        _user.value = user
    } catch (e: IOException) {
        _error.value = "No internet connection"
    } catch (e: HttpException) {
        _error.value = "Server error: ${e.code()}"
    }
}
```

---

## 🔑 Key Concepts

| Concept | What It Does | Analogy |
|---------|-------------|---------|
| `suspend fun` | Function that can pause/resume | Pause a movie then continue |
| `launch` | Start a coroutine, no result | Send a letter, don't wait for reply |
| `async` / `await` | Start coroutine, wait for result | Order food, wait for waiter |
| `withContext(IO)` | Switch to background thread | Ask assistant to do boring work |
| `Dispatchers.Main` | UI thread | Only place to update UI |
| `Dispatchers.IO` | Background thread for I/O | File, network, database |
| `viewModelScope` | Coroutine tied to ViewModel life | Auto-cancelled on screen close |
| `Flow` | Stream of values over time | Live news feed |
| `StateFlow` | Flow with a current state | Real-time stock price display |
| `collect {}` | Listen to Flow values | Subscribe to a YouTube channel |

---

## 💡 Good Example — Complete ViewModel with Coroutines

```kotlin
// UiState — sealed class to represent all possible states
sealed class UiState<out T> {
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String) : UiState<Nothing>()
}

class UserViewModel(private val repository: UserRepository) : ViewModel() {

    // StateFlow holds current UI state
    private val _uiState = MutableStateFlow<UiState<List<User>>>(UiState.Loading)
    val uiState: StateFlow<UiState<List<User>>> = _uiState.asStateFlow()

    // Call this when screen loads
    fun loadUsers() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                // Parallel fetching — both run at the same time!
                val usersDeferred = async(Dispatchers.IO) { repository.getUsers() }
                val settingsDeferred = async(Dispatchers.IO) { repository.getSettings() }
                
                val users = usersDeferred.await()
                settingsDeferred.await()  // just wait for settings to complete
                
                _uiState.value = UiState.Success(users)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Unknown error")
            }
        }
    }

    fun deleteUser(userId: Int) {
        viewModelScope.launch {
            try {
                withContext(Dispatchers.IO) { repository.deleteUser(userId) }
                loadUsers()  // refresh list after delete
            } catch (e: Exception) {
                _uiState.value = UiState.Error("Failed to delete: ${e.message}")
            }
        }
    }
}

// In Fragment:
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state ->
            when (state) {
                is UiState.Loading -> {
                    binding.progressBar.isVisible = true
                    binding.recyclerView.isVisible = false
                }
                is UiState.Success -> {
                    binding.progressBar.isVisible = false
                    binding.recyclerView.isVisible = true
                    adapter.submitList(state.data)
                }
                is UiState.Error -> {
                    binding.progressBar.isVisible = false
                    Snackbar.make(binding.root, state.message, Snackbar.LENGTH_LONG).show()
                }
            }
        }
    }
}
```
