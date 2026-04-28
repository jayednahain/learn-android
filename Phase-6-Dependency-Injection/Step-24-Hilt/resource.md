# Step 24 — Hilt (Replaces Dagger — Dependency Injection)

---

## 📖 What Is It? (Simple Definition)

Imagine you own a restaurant. Every morning, the chef needs: a knife, cutting board, pans, ingredients. In the old system, the chef goes and gets every item themselves — very inefficient!

**Dependency Injection (DI)** is like a supply manager who prepares everything the chef needs and places it on the counter before the chef starts. The chef just starts cooking!

- Without DI: `val repo = UserRepository(UserApi(), UserDao(), Database)`  ← you build everything manually
- With Hilt DI: `@Inject lateinit var repo: UserRepository` ← Hilt builds it and gives it to you!

**Hilt** is Google's simplified DI library for Android. Before Hilt, we used raw **Dagger** which was complex. Hilt makes DI simple.

---

## 🎯 Where Do We Use It?

- Provide `Repository` to `ViewModel`
- Provide `Retrofit ApiService` to `Repository`
- Provide `Room Database` to `Repository`
- Avoid manually creating long chains of objects
- Makes testing easy (swap real API with fake one!)

---

## 🔄 Hilt Dependency Flow Diagram

```
@HiltAndroidApp        → Application class
       ↓ provides dependencies to
@AndroidEntryPoint     → Activity, Fragment, Service, Worker
       ↓ uses
@HiltViewModel         → ViewModel (via @Inject constructor)
       ↓ gets its deps from
@Module / @Provides    → Where objects are created (Retrofit, Room, etc.)
       ↓ creates
Repository → ApiService, Dao (the actual dependencies)
```

---

## ☕ Java vs Kotlin — What Changed?

### Old Java + Manual DI — Painful!

```java
// Java — manually creating every dependency (DI without a framework)
ApiService apiService = RetrofitClient.getApiService();
UserDao userDao = AppDatabase.getInstance(context).userDao();
UserRepository repository = new UserRepository(apiService, userDao);
UserViewModel viewModel = new UserViewModel(repository);
// Problem: 10+ objects, all manually created, impossible to test
```

### Old Dagger (Pre-Hilt) — Complicated!

```kotlin
// Dagger — too many boilerplate files:
// Components, Subcomponents, Modules, Provides, Inject, Scope annotations...
// Very powerful but steep learning curve
@Component(modules = [AppModule::class, NetworkModule::class])
interface AppComponent {
    fun inject(activity: MainActivity)
    fun inject(fragment: HomeFragment)
    // 20 more inject() methods...
}
```

### New Hilt — Simple! ✅

```kotlin
// Step 1: Add dependencies to build.gradle
// implementation("com.google.dagger:hilt-android:2.48")
// kapt("com.google.dagger:hilt-android-compiler:2.48")
// (or ksp for KSP users)
```

```kotlin
// Step 2: Annotate your Application class
@HiltAndroidApp
class MyApp : Application() {
    // Nothing else needed! Hilt sets up everything.
}
```

```xml
<!-- AndroidManifest.xml — register your Application class -->
<application android:name=".MyApp" ...>
```

```kotlin
// Step 3: Mark Activities/Fragments as Hilt entry points
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    // Hilt can now inject into this Activity
}

@AndroidEntryPoint
class HomeFragment : Fragment() {
    // Hilt can now inject into this Fragment
}
```

---

### Creating Injected Classes with `@Inject constructor`

```kotlin
// Hilt learns how to create UserRepository from the @Inject constructor
class UserRepository @Inject constructor(
    private val apiService: UserApi,   // Hilt provides this
    private val userDao: UserDao        // Hilt provides this
) {
    suspend fun getUsers() = apiService.getUsers()
    fun getUsersAsFlow() = userDao.getAllUsers()
}
```

---

### Providing Third-Party Objects with `@Module` + `@Provides`

You can't add `@Inject constructor` to classes you don't own (Retrofit, OkHttp, Room). Use `@Module` instead:

```kotlin
@Module
@InstallIn(SingletonComponent::class)  // live for entire app lifetime
object NetworkModule {

    @Provides
    @Singleton  // create only ONE instance
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = HttpLoggingInterceptor.Level.BODY
            })
            .build()
    }

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(okHttpClient)  // Hilt automatically injects the OkHttpClient above!
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    @Provides
    @Singleton
    fun provideUserApi(retrofit: Retrofit): UserApi {
        return retrofit.create(UserApi::class.java)  // Hilt injects Retrofit above
    }
}

@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(context, AppDatabase::class.java, "app.db").build()
    }

    @Provides
    fun provideUserDao(database: AppDatabase): UserDao {
        return database.userDao()
        // No @Singleton — DAO is tied to database which is @Singleton
    }
}
```

---

### `@HiltViewModel` — Inject ViewModel

```kotlin
// ViewModel with Hilt — just add @HiltViewModel and @Inject constructor!
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository  // Hilt injects UserRepository
) : ViewModel() {

    private val _users = MutableStateFlow<List<User>>(emptyList())
    val users: StateFlow<List<User>> = _users.asStateFlow()

    init { loadUsers() }

    fun loadUsers() {
        viewModelScope.launch {
            _users.value = withContext(Dispatchers.IO) { repository.getUsers() }
        }
    }
}

// In Fragment — no factory needed! "by viewModels()" just works with Hilt!
@AndroidEntryPoint
class HomeFragment : Fragment() {
    private val viewModel: UserViewModel by viewModels()  // Hilt magic ✨
    // Before Hilt, you needed a ViewModelFactory with 20 lines...
}
```

---

### `@Binds` — Interface Bindings

```kotlin
// When you have an interface and a concrete implementation:
interface UserRepository {
    suspend fun getUsers(): List<User>
}

class UserRepositoryImpl @Inject constructor(
    private val api: UserApi,
    private val dao: UserDao
) : UserRepository {
    override suspend fun getUsers() = api.getUsers().map { it.toUser() }
}

// Tell Hilt: "When someone asks for UserRepository, give them UserRepositoryImpl"
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    @Singleton
    abstract fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
    //                                     ↑ concrete class         ↑ interface (what to inject as)
}
```

---

### Hilt Scopes — How Long Does an Object Live?

```kotlin
@Singleton          // 1 instance for entire app lifetime (Application scope)
@ActivityScoped     // 1 instance per Activity (destroyed when activity dies)
@FragmentScoped     // 1 instance per Fragment
@ViewModelScoped    // 1 instance per ViewModel (use with @HiltViewModel)
```

---

## 🔑 Key Concepts

| Annotation | On What? | Purpose |
|-----------|---------|---------|
| `@HiltAndroidApp` | Application class | Enable Hilt in the app |
| `@AndroidEntryPoint` | Activity, Fragment, Service | Allow injection here |
| `@Inject constructor` | Your own class | Hilt learns to create this class |
| `@Module` | Object/Abstract class | Group of `@Provides` or `@Binds` |
| `@InstallIn(...)` | Module | Scope of the module |
| `@Provides` | Function in module | Tell Hilt HOW to create the object |
| `@Binds` | Abstract function | Bind interface → implementation |
| `@Singleton` | Provided object | One instance per app |
| `@HiltViewModel` | ViewModel class | Allow injection into ViewModel |
| `by viewModels()` | Fragment/Activity | Get ViewModel with Hilt |

---

## 💡 Good Example — Full Hilt-Wired App

```
Complete dependency chain built automatically by Hilt:

Fragment
  by viewModels() → UserViewModel (@HiltViewModel)
                          ↓ @Inject constructor
                    UserRepository (@Inject constructor)
                       ↙              ↘
                  UserApi              UserDao
               (from @Provides       (from @Provides
                in NetworkModule)     in DatabaseModule)
```

```kotlin
// That's all the code you need in each class:

@HiltAndroidApp
class App : Application()

@AndroidEntryPoint
class HomeFragment : Fragment() {
    private val viewModel: UserViewModel by viewModels()
    // Entire chain above is set up by Hilt automatically!
}

@HiltViewModel
class UserViewModel @Inject constructor(val repo: UserRepository) : ViewModel()

class UserRepository @Inject constructor(val api: UserApi, val dao: UserDao)

// Only NetworkModule and DatabaseModule need @Module boilerplate
// for third-party classes (Retrofit, Room)
```
