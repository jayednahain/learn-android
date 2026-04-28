# Step 13 — LiveData & StateFlow

---

## 📖 What Is It? (Simple Definition)

Imagine a scoreboard at a football match. Whenever a team scores, the scoreboard **automatically** updates for everyone watching. You don't need to keep refreshing it.

**LiveData** and **StateFlow** are like that scoreboard. When your data changes (new score!), the UI automatically updates. You don't write code like "go check if data changed, then update the screen" — it just happens!

The difference:
- **LiveData** = the older Android-specific scoreboard (lifecycle-aware)
- **StateFlow** = the modern Kotlin scoreboard (more powerful, works everywhere)

---

## 🎯 Where Do We Use It?

- ViewModel → UI communication (always!)
- Show/hide loading spinner based on network state
- Update a list when new data arrives from database
- Show error messages when API fails
- Real-time updates (chat, live scores, prices)

---

## 🔄 Data Flow Diagram

```
With LiveData:
  Repository / API
       ↓
  ViewModel holds MutableLiveData<T>
       ↓ (observe in Activity/Fragment)
  UI auto-updates when value changes
  (Only updates when UI is active — SAFE from leaks!)

With StateFlow:
  Repository / API / Flow
       ↓
  ViewModel holds MutableStateFlow<T>
       ↓ (collect in Activity/Fragment with repeatOnLifecycle)
  UI auto-updates
  (Works with Kotlin Flow pipeline, more flexible)
```

---

## ☕ Java vs Kotlin — What Changed?

### Old Java Way — Manual UI Updates (error-prone)

```java
// Java — have to manually refresh UI everywhere
// Network call returns → manually call updateUI()
// User updates data → manually call refreshList()
// Screen rotation → data lost, manually restore

private void fetchUser() {
    apiService.getUser(1, new Callback<User>() {
        @Override
        public void onResponse(Call<User> call, Response<User> response) {
            runOnUiThread(() -> {
                tvName.setText(response.body().getName());
                // What if Activity was destroyed by now? CRASH!
            });
        }
    });
}
```

### New Way — LiveData ✅

```kotlin
// ViewModel
class UserViewModel : ViewModel() {
    // MutableLiveData — can SET value (private to ViewModel)
    private val _user = MutableLiveData<User>()
    // LiveData — only READ (public to UI)
    val user: LiveData<User> = _user

    private val _isLoading = MutableLiveData(false)
    val isLoading: LiveData<Boolean> = _isLoading

    fun loadUser(id: Int) {
        _isLoading.value = true
        viewModelScope.launch {
            val result = withContext(Dispatchers.IO) { repository.getUser(id) }
            _user.value = result           // setValue — must be on Main thread
            _isLoading.value = false
        }
        // Or from background thread: _user.postValue(result) (thread-safe)
    }
}

// Activity/Fragment — observe LiveData
class UserFragment : Fragment() {
    private val viewModel: UserViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // observe() — ONLY called when Fragment is active (RESUMED/STARTED)
        // Safe! Won't crash if screen is in background
        viewModel.user.observe(viewLifecycleOwner) { user ->
            binding.tvName.text = user.name
            binding.tvEmail.text = user.email
        }

        viewModel.isLoading.observe(viewLifecycleOwner) { isLoading ->
            binding.progressBar.isVisible = isLoading
        }

        viewModel.loadUser(1)
    }
}
```

### New Way — StateFlow ✅ (Modern Preferred)

```kotlin
// ViewModel with StateFlow
class UserViewModel : ViewModel() {
    private val _user = MutableStateFlow<User?>(null)
    val user: StateFlow<User?> = _user.asStateFlow()

    private val _isLoading = MutableStateFlow(false)
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()
}

// Fragment — collect StateFlow
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    // IMPORTANT: Use repeatOnLifecycle to be lifecycle-safe (like LiveData's observe)
    viewLifecycleOwner.lifecycleScope.launch {
        viewLifecycleOwner.lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
            
            // Collect multiple flows in parallel
            launch {
                viewModel.user.collect { user ->
                    user?.let {
                        binding.tvName.text = it.name
                    }
                }
            }
            launch {
                viewModel.isLoading.collect { isLoading ->
                    binding.progressBar.isVisible = isLoading
                }
            }
        }
    }
}
```

---

### `setValue()` vs `postValue()`

```kotlin
// setValue — must be called from MAIN (UI) thread
_user.value = fetchedUser  // ✅ inside launch on Main thread
_user.value = fetchedUser  // ❌ inside Dispatchers.IO → crash!

// postValue — can be called from ANY thread (background safe)
_user.postValue(fetchedUser)  // ✅ safe from background thread
// Note: if called multiple times quickly, only the last value is delivered
```

---

### `MediatorLiveData` — Combine Multiple LiveData Sources

```kotlin
// Combine two LiveData sources into one:
val searchResults: MediatorLiveData<List<User>> = MediatorLiveData()

init {
    searchResults.addSource(localUsers) { local ->
        searchResults.value = mergeResults(local, searchResults.value)
    }
    searchResults.addSource(remoteUsers) { remote ->
        searchResults.value = mergeResults(searchResults.value, remote)
    }
}
```

---

### When to Use LiveData vs StateFlow

| Situation | Use | Why |
|-----------|-----|-----|
| Simple UI state with lifecycle awareness | `LiveData` | Simple, well-understood |
| Kotlin-only codebase using Flow | `StateFlow` | Integrates with Flow operators |
| Combining multiple streams | `StateFlow` + Flow operators | More powerful than MediatorLiveData |
| Room database query result | `Flow` → converted to `StateFlow` | Room natively returns Flow |
| Jetpack Compose | `StateFlow` | `collectAsStateWithLifecycle()` |
| Shared between multiple collectors | `SharedFlow` | Multiple observers / events |

---

### `SharedFlow` — For One-Time Events (Navigation, Toast, Dialog)

```kotlin
// Problem: LiveData keeps the last value — if you observe a "show dialog" event
// after rotation, the dialog shows again! That's wrong.

// Solution: SharedFlow — events are consumed, not replayed
class UserViewModel : ViewModel() {
    private val _navigationEvent = MutableSharedFlow<NavigationEvent>()
    val navigationEvent: SharedFlow<NavigationEvent> = _navigationEvent.asSharedFlow()

    fun onLoginSuccess(user: User) {
        viewModelScope.launch {
            _navigationEvent.emit(NavigationEvent.GoToHome(user.id))
            // This event fires once and is gone — no replays on rotation!
        }
    }
}

sealed class NavigationEvent {
    data class GoToHome(val userId: Int) : NavigationEvent()
    object GoToLogin : NavigationEvent()
}

// In Fragment:
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.navigationEvent.collect { event ->
            when (event) {
                is NavigationEvent.GoToHome -> findNavController().navigate(
                    LoginFragmentDirections.actionLoginToHome(event.userId)
                )
                NavigationEvent.GoToLogin -> findNavController().navigate(R.id.loginFragment)
            }
        }
    }
}
```

---

## 🔑 Key Concepts

| Concept | Type | Key Feature |
|---------|------|------------|
| `LiveData<T>` | Lifecycle-aware observable | Only delivers to active observers |
| `MutableLiveData<T>` | Settable LiveData | In ViewModel, kept private |
| `.observe(lifecycleOwner)` | Subscribe to LiveData | Auto-unsubscribes on destroy |
| `.value` / `.postValue()` | Set LiveData value | `value` on Main, `postValue` anywhere |
| `StateFlow<T>` | Hot observable with current state | Always has a value |
| `MutableStateFlow<T>` | Settable StateFlow | In ViewModel, kept private |
| `.collect {}` | Subscribe to Flow/StateFlow | Use with `repeatOnLifecycle` |
| `SharedFlow<T>` | One-time event stream | No replay — for navigation, toasts |

---

## 💡 Good Example — Complete Reactive Screen

```kotlin
// UI State sealed class
sealed class ProfileUiState {
    object Loading : ProfileUiState()
    data class Success(val profile: UserProfile) : ProfileUiState()
    data class Error(val message: String) : ProfileUiState()
}

sealed class ProfileEvent {
    data class ShowToast(val message: String) : ProfileEvent()
    object NavigateBack : ProfileEvent()
}

class ProfileViewModel(
    private val repo: UserRepository,
    private val userId: Int
) : ViewModel() {

    private val _uiState = MutableStateFlow<ProfileUiState>(ProfileUiState.Loading)
    val uiState: StateFlow<ProfileUiState> = _uiState.asStateFlow()

    private val _events = MutableSharedFlow<ProfileEvent>()
    val events: SharedFlow<ProfileEvent> = _events.asSharedFlow()

    init { loadProfile() }

    fun loadProfile() {
        viewModelScope.launch {
            _uiState.value = ProfileUiState.Loading
            try {
                val profile = withContext(Dispatchers.IO) { repo.getProfile(userId) }
                _uiState.value = ProfileUiState.Success(profile)
            } catch (e: Exception) {
                _uiState.value = ProfileUiState.Error(e.message ?: "Failed to load")
            }
        }
    }

    fun onSaveClicked(updatedProfile: UserProfile) {
        viewModelScope.launch {
            try {
                withContext(Dispatchers.IO) { repo.updateProfile(updatedProfile) }
                _events.emit(ProfileEvent.ShowToast("Profile saved!"))
                _events.emit(ProfileEvent.NavigateBack)
            } catch (e: Exception) {
                _events.emit(ProfileEvent.ShowToast("Failed: ${e.message}"))
            }
        }
    }
}
```
