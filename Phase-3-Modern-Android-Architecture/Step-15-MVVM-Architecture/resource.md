# Step 15 — MVVM Architecture Pattern

---

## 📖 What Is It? (Simple Definition)

MVVM stands for **Model - View - ViewModel**. It's like a well-organized kitchen:

- **Model** = The ingredients and recipes (data and how to get it)
- **ViewModel** = The chef who prepares the dish (business logic)
- **View** = The plate that presents the food (UI — Activity/Fragment)

The rule: The **chef** doesn't come to the dining room (View) directly. The waiter (data binding / observation) carries the plate. And the chef doesn't grow vegetables (Model) — the farmer (Repository) does.

This clean separation means: if you change the menu design (UI), the kitchen logic doesn't break. 

---

## 🎯 Where Do We Use It?

- Every screen in a production Android app
- Google's officially recommended architecture
- The foundation for Jetpack Compose (which follows the same idea)

---

## 🔄 MVVM Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        VIEW LAYER                           │
│          Activity / Fragment (XML Layouts)                  │
│                                                             │
│  - Shows data from ViewModel                                │
│  - Sends user events to ViewModel (clicks, typing)          │
│  - Has ZERO business logic — just UI!                       │
└────────────────────┬────────────────────────────────────────┘
         observes    │  calls functions (user events)
┌────────────────────▼────────────────────────────────────────┐
│                     VIEWMODEL LAYER                         │
│                  (survives rotation)                        │
│                                                             │
│  - Holds UiState (StateFlow/LiveData)                       │
│  - Business logic: validate form, filter data               │
│  - Calls Repository for data                                │
│  - Does NOT know about Views, Context, Activity             │
└────────────────────┬────────────────────────────────────────┘
         calls       │  returns Flow / suspend results
┌────────────────────▼────────────────────────────────────────┐
│                      MODEL LAYER                            │
│                                                             │
│  Repository ──────┬──── Remote API (Retrofit)               │
│  (data manager)   └──── Local DB (Room)                     │
│                                                             │
│  Data Classes: User, Product, etc. (plain data holders)     │
└─────────────────────────────────────────────────────────────┘
```

---

## ☕ Java vs Kotlin — What Changed?

### Old Java "MVC" Pattern — Messy!

```java
// Java Activity doing EVERYTHING — God Object!
public class UserActivity extends AppCompatActivity {
    // UI
    private TextView tvName;
    private Button btnSave;

    // Business logic directly in Activity?!
    private void validateAndSave() {
        String name = etName.getText().toString();
        if (name.isEmpty()) {        // validation in Activity — wrong!
            tvError.setText("Name required");
            return;
        }

        // Direct API call in Activity — wrong!
        apiService.saveUser(name).enqueue(new Callback<User>() {
            @Override
            public void onResponse(...) {
                // Update UI — this is okay
                tvName.setText(response.body().getName());
                // But all responsibility is here in one place!
            }
        });
    }
}
// Problems:
// - Activity has 500+ lines
// - Can't test business logic (tied to Android UI)
// - Screen rotation resets everything
// - Hard to read, hard to change
```

### New MVVM Way ✅

```kotlin
// 1. MODEL — Data class (just data, no logic)
data class User(
    val id: Int,
    val name: String,
    val email: String,
    val bio: String = ""
)

// UiState — what the UI looks like at any moment
data class EditProfileUiState(
    val name: String = "",
    val email: String = "",
    val bio: String = "",
    val isLoading: Boolean = false,
    val nameError: String? = null,
    val emailError: String? = null,
    val isSaveEnabled: Boolean = false
)

// 2. VIEWMODEL — Business logic
class EditProfileViewModel(
    private val repository: UserRepository,
    private val userId: Int
) : ViewModel() {

    private val _uiState = MutableStateFlow(EditProfileUiState())
    val uiState: StateFlow<EditProfileUiState> = _uiState.asStateFlow()

    private val _events = MutableSharedFlow<String>()
    val events: SharedFlow<String> = _events.asSharedFlow()

    init { loadProfile() }

    private fun loadProfile() {
        viewModelScope.launch {
            _uiState.value = _uiState.value.copy(isLoading = true)
            try {
                val user = repository.getUser(userId)
                _uiState.value = EditProfileUiState(
                    name = user.name,
                    email = user.email,
                    bio = user.bio,
                    isSaveEnabled = true
                )
            } catch (e: Exception) {
                _events.emit("Failed to load profile")
            } finally {
                _uiState.value = _uiState.value.copy(isLoading = false)
            }
        }
    }

    // Handle UI events (user typing, clicking)
    fun onNameChanged(name: String) {
        val error = if (name.isBlank()) "Name cannot be empty" else null
        _uiState.value = _uiState.value.copy(
            name = name,
            nameError = error,
            isSaveEnabled = error == null
        )
    }

    fun onEmailChanged(email: String) {
        val error = if (!android.util.Patterns.EMAIL_ADDRESS.matcher(email).matches())
            "Invalid email" else null
        _uiState.value = _uiState.value.copy(
            email = email,
            emailError = error
        )
    }

    fun onSaveClicked() {
        val state = _uiState.value
        if (state.nameError != null || state.emailError != null) return

        viewModelScope.launch {
            _uiState.value = state.copy(isLoading = true, isSaveEnabled = false)
            try {
                repository.updateUser(User(userId, state.name, state.email, state.bio))
                _events.emit("Profile saved successfully!")
            } catch (e: Exception) {
                _events.emit("Failed to save: ${e.message}")
                _uiState.value = _uiState.value.copy(isSaveEnabled = true)
            } finally {
                _uiState.value = _uiState.value.copy(isLoading = false)
            }
        }
    }
}

// 3. VIEW — Activity/Fragment (just UI, no logic!)
class EditProfileFragment : Fragment() {
    private val viewModel: EditProfileViewModel by viewModels()
    private var _binding: FragmentEditProfileBinding? = null
    private val binding get() = _binding!!

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // Wire up user inputs to ViewModel
        binding.etName.addTextChangedListener { viewModel.onNameChanged(it.toString()) }
        binding.etEmail.addTextChangedListener { viewModel.onEmailChanged(it.toString()) }
        binding.btnSave.setOnClickListener { viewModel.onSaveClicked() }

        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {

                // Observe UI state
                launch {
                    viewModel.uiState.collect { state ->
                        // Populate fields (only if not same to avoid cursor jump)
                        if (binding.etName.text.toString() != state.name)
                            binding.etName.setText(state.name)
                        if (binding.etEmail.text.toString() != state.email)
                            binding.etEmail.setText(state.email)

                        // Show/hide errors
                        binding.nameLayout.error = state.nameError
                        binding.emailLayout.error = state.emailError

                        // Loading state
                        binding.progressBar.isVisible = state.isLoading
                        binding.btnSave.isEnabled = state.isSaveEnabled && !state.isLoading
                    }
                }

                // Observe one-time events
                launch {
                    viewModel.events.collect { message ->
                        Snackbar.make(binding.root, message, Snackbar.LENGTH_SHORT).show()
                    }
                }
            }
        }
    }

    override fun onDestroyView() { super.onDestroyView(); _binding = null }
}
```

---

## 🔑 Key Concepts

| Layer | Class Types | Responsibility |
|-------|------------|---------------|
| **View** | Activity, Fragment | Show UI, forward user events to ViewModel |
| **ViewModel** | ViewModel | Hold UiState, business logic, call Repository |
| **Model** | Repository, data classes, Room, Retrofit | Data access and storage |

### Rules of MVVM

| DO ✅ | DON'T ❌ |
|-------|---------|
| ViewModel calls Repository | ViewModel imports Android UI classes |
| Fragment observes StateFlow | Fragment has if/else business logic |
| UiState is a data class | ViewModel stores Activity/Fragment ref |
| Repository returns Flow/suspend | Repository updates UI directly |
| View Layer is dumb | Activity has 500+ lines of logic |

---

## 💡 Good Example — File/Folder Structure for MVVM

```
app/
└── src/main/java/com/example/app/
    ├── data/
    │   ├── model/          ← Data classes (User, Product, etc.)
    │   │   └── User.kt
    │   ├── remote/         ← API interfaces (Retrofit)
    │   │   └── UserApi.kt
    │   ├── local/          ← Room DAO and entities
    │   │   ├── UserDao.kt
    │   │   └── UserEntity.kt
    │   └── repository/     ← Repository implementations
    │       └── UserRepositoryImpl.kt
    │
    ├── domain/
    │   └── repository/     ← Repository interfaces (for DI)
    │       └── UserRepository.kt
    │
    └── ui/
        └── profile/        ← One folder per screen
            ├── ProfileFragment.kt      ← VIEW
            ├── ProfileViewModel.kt     ← VIEWMODEL
            └── ProfileUiState.kt      ← UI State data class
```
