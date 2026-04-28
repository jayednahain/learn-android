# Step 17 — DataStore (Replaces SharedPreferences)

---

## 📖 What Is It? (Simple Definition)

Imagine a small notebook where your app saves simple settings — like "is dark mode on?", "what was the last search term?", "is the user logged in?".

**SharedPreferences** was the old notebook — but it was slow, could crash your app (blocking the main thread), and didn't work well with modern Kotlin.

**DataStore** is the new upgraded smart notebook — async (doesn't block UI), safe, and works perfectly with Kotlin coroutines and Flow.

---

## 🎯 Where Do We Use It?

- Save user preferences (dark mode, language, notification settings)
- Remember if the user completed onboarding
- Store auth token / login state
- Save last scroll position or search query
- Any small key-value data that needs to persist (not a full database)

> 💡 Rule: Small, simple, non-relational data → DataStore. Structured/complex data → Room.

---

## 🔄 DataStore vs SharedPreferences

```
SharedPreferences (Old):
  Context.getSharedPreferences(...)
  ↓
  .edit().putString("key", value).apply()   ← async but no callback
  .edit().putString("key", value).commit()  ← BLOCKS main thread!
  ↓
  Crashes possible, not coroutine-friendly

DataStore (New):
  Context.dataStore  (created once with preferencesDataStore delegate)
  ↓
  dataStore.edit { prefs -> prefs[KEY] = value }    ← suspend, safe
  dataStore.data.map { prefs -> prefs[KEY] ?: DEFAULT }  ← Flow, reactive
  ↓
  Never blocks, coroutine-native, crash-safe
```

---

## ☕ Java vs Kotlin — What Changed?

### Old Java Way — SharedPreferences

```java
// Java SharedPreferences — old and risky
SharedPreferences prefs = getSharedPreferences("my_prefs", Context.MODE_PRIVATE);

// Write (apply = async but fire-and-forget, commit = blocks main thread!)
SharedPreferences.Editor editor = prefs.edit();
editor.putString("user_name", "Jayed");
editor.putBoolean("dark_mode", true);
editor.putInt("theme_color", 0xFF2196F3);
editor.apply(); // or .commit() which blocks UI thread!

// Read (synchronous — potential ANR on slow storage!)
String name = prefs.getString("user_name", "Unknown");
boolean darkMode = prefs.getBoolean("dark_mode", false);

// Problems:
// - Not type-safe (easy to use wrong type)
// - Main thread can block on slow storage
// - No clear way to observe changes with LiveData/Flow
// - Concurrent writes can corrupt data
```

### New Way — Preferences DataStore ✅

```kotlin
// Step 1: Add to build.gradle
// implementation("androidx.datastore:datastore-preferences:1.0.0")

// Step 2: Create DataStore — define once at file level (top of file, not in class!)
private val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "user_settings")

// Step 3: Define keys with type safety
object SettingsKeys {
    val USER_NAME = stringPreferencesKey("user_name")
    val DARK_MODE = booleanPreferencesKey("dark_mode")
    val ONBOARDING_DONE = booleanPreferencesKey("onboarding_done")
    val SELECTED_LANGUAGE = stringPreferencesKey("selected_language")
    val FONT_SIZE = intPreferencesKey("font_size")
    val LAST_SEARCH = stringPreferencesKey("last_search")
}

// Step 4: Repository wrapping DataStore
class SettingsRepository(private val dataStore: DataStore<Preferences>) {

    // READ — returns Flow (reactive! UI updates automatically on change)
    val isDarkMode: Flow<Boolean> = dataStore.data
        .map { prefs -> prefs[SettingsKeys.DARK_MODE] ?: false }

    val userName: Flow<String> = dataStore.data
        .map { prefs -> prefs[SettingsKeys.USER_NAME] ?: "" }

    val isOnboardingDone: Flow<Boolean> = dataStore.data
        .map { prefs -> prefs[SettingsKeys.ONBOARDING_DONE] ?: false }

    // READ multiple values at once
    val appSettings: Flow<AppSettings> = dataStore.data.map { prefs ->
        AppSettings(
            isDarkMode = prefs[SettingsKeys.DARK_MODE] ?: false,
            language = prefs[SettingsKeys.SELECTED_LANGUAGE] ?: "en",
            fontSize = prefs[SettingsKeys.FONT_SIZE] ?: 14
        )
    }

    // WRITE — suspend function (safe, never blocks main thread)
    suspend fun setDarkMode(enabled: Boolean) {
        dataStore.edit { prefs -> prefs[SettingsKeys.DARK_MODE] = enabled }
    }

    suspend fun setUserName(name: String) {
        dataStore.edit { prefs -> prefs[SettingsKeys.USER_NAME] = name }
    }

    suspend fun completeOnboarding() {
        dataStore.edit { prefs -> prefs[SettingsKeys.ONBOARDING_DONE] = true }
    }

    // CLEAR ALL settings
    suspend fun clearAll() {
        dataStore.edit { it.clear() }
    }

    // CLEAR specific key
    suspend fun clearUserName() {
        dataStore.edit { prefs -> prefs.remove(SettingsKeys.USER_NAME) }
    }
}

data class AppSettings(
    val isDarkMode: Boolean,
    val language: String,
    val fontSize: Int
)
```

---

### Using DataStore in ViewModel

```kotlin
class SettingsViewModel(private val settingsRepo: SettingsRepository) : ViewModel() {

    val isDarkMode: StateFlow<Boolean> = settingsRepo.isDarkMode
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), false)

    val appSettings: StateFlow<AppSettings> = settingsRepo.appSettings
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), AppSettings(false, "en", 14))

    fun toggleDarkMode() {
        viewModelScope.launch {
            settingsRepo.setDarkMode(!isDarkMode.value)
        }
    }

    fun setLanguage(lang: String) {
        viewModelScope.launch {
            settingsRepo.setSelectedLanguage(lang)
        }
    }
}

// In Fragment:
class SettingsFragment : Fragment() {
    private val viewModel: SettingsViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // Toggle switch
        binding.switchDarkMode.setOnCheckedChangeListener { _, isChecked ->
            viewModel.toggleDarkMode()
        }

        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.isDarkMode.collect { isDark ->
                    binding.switchDarkMode.isChecked = isDark
                    // Apply theme
                    AppCompatDelegate.setDefaultNightMode(
                        if (isDark) AppCompatDelegate.MODE_NIGHT_YES
                        else AppCompatDelegate.MODE_NIGHT_NO
                    )
                }
            }
        }
    }
}
```

---

### Proto DataStore — Typed Object Storage

Proto DataStore uses Protocol Buffers (protobuf) for strongly-typed, structured data. Use it when your settings are complex objects.

```protobuf
// userprefs.proto
syntax = "proto3";
option java_package = "com.example.app";
option java_multiple_files = true;

message UserPrefsProto {
    string user_name = 1;
    bool dark_mode = 2;
    string language = 3;
    int32 font_size = 4;
}
```

```kotlin
// Serializer for UserPrefsProto
object UserPrefsSerializer : Serializer<UserPrefsProto> {
    override val defaultValue: UserPrefsProto = UserPrefsProto.getDefaultInstance()
    override suspend fun readFrom(input: InputStream): UserPrefsProto =
        UserPrefsProto.parseFrom(input)
    override suspend fun writeTo(t: UserPrefsProto, output: OutputStream) = t.writeTo(output)
}

// Use it
private val Context.protoDataStore by dataStore("user_prefs.pb", UserPrefsSerializer)
```

---

## 🔑 Key Concepts

| Concept | SharedPreferences | DataStore |
|---------|-----------------|---------|
| Thread safety | ❌ can block UI | ✅ always async |
| Kotlin coroutines | ❌ not native | ✅ native suspend/Flow |
| Type safety | ❌ easy to use wrong type | ✅ typed keys (`stringPreferencesKey`) |
| Reactive updates | ❌ manual listeners | ✅ Flow (auto-updates) |
| Error handling | ❌ silent failures | ✅ exceptions caught via Flow |
| Concurrent writes | ❌ can corrupt | ✅ atomic transactions |

---

## 💡 Good Example — Onboarding Completed Check

```kotlin
class SplashViewModel(private val settings: SettingsRepository) : ViewModel() {
    
    private val _navigationTarget = MutableSharedFlow<String>()
    val navigationTarget: SharedFlow<String> = _navigationTarget.asSharedFlow()

    init {
        viewModelScope.launch {
            // Check if user has completed onboarding — read ONCE
            val isOnboardingDone = settings.isOnboardingDone.first()
            val isLoggedIn = settings.isLoggedIn.first()

            when {
                !isOnboardingDone -> _navigationTarget.emit("onboarding")
                !isLoggedIn       -> _navigationTarget.emit("login")
                else              -> _navigationTarget.emit("home")
            }
        }
    }
}

// SplashActivity:
class SplashActivity : AppCompatActivity() {
    private val viewModel: SplashViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleScope.launch {
            viewModel.navigationTarget.collect { destination ->
                val intent = when (destination) {
                    "onboarding" -> Intent(this@SplashActivity, OnboardingActivity::class.java)
                    "login" -> Intent(this@SplashActivity, LoginActivity::class.java)
                    else -> Intent(this@SplashActivity, HomeActivity::class.java)
                }
                startActivity(intent)
                finish()
            }
        }
    }
}
```
