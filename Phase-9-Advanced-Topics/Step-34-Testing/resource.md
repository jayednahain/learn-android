# Step 34 — Testing

---

## 📖 What Is It? (Simple Explanation)

Imagine you build a toy car 🚗. Before giving it to a friend, you test it — does it turn left? Does it brake? Do the wheels fall off?

**Testing** in Android is the same idea. You write small programs that check your code works correctly. Every time you change something, the tests run and tell you immediately if something broke — without you having to click through the whole app manually.

---

## 🧭 Where Do We Use It?

| Layer | Test Type |
|---|---|
| Pure functions, ViewModel logic | Unit tests (JUnit) |
| ViewModel with state | ViewModel unit tests |
| Database queries | Room database tests |
| Full UI interactions | Espresso (instrumented tests) |
| Compose UI | Compose UI tests |
| Hilt dependencies in tests | Hilt test components |

---

## 🗺️ Workflow / How It Works

```
Android Test Pyramid:

         /\
        /  \   UI Tests (Espresso / Compose)
       /    \  → Slow, test real device/emulator
      /──────\
     /        \  Integration Tests (Room, Repositories)
    /          \  → Medium speed
   /────────────\
  /              \  Unit Tests (JUnit + MockK/Mockito)
 /________________\  → Fast, no device needed
```

### Test Folder Structure

```
app/src/
    main/java/...          ← your app code
    test/java/...          ← Unit tests (runs on JVM, fast)
    androidTest/java/...   ← Instrumented tests (runs on device)
```

---

## ☕ Java vs Kotlin — What Changed?

### Old Way (Java — Mockito, verbose)

```java
// Java unit test — a lot of boilerplate
@RunWith(MockitoJUnitRunner.class)
public class UserViewModelTest {

    @Mock
    UserRepository mockRepository;

    private UserViewModel viewModel;

    @Before
    public void setup() {
        viewModel = new UserViewModel(mockRepository);
    }

    @Test
    public void testLoadUser_returnsUser() {
        User fakeUser = new User(1, "John");
        when(mockRepository.getUser(1)).thenReturn(fakeUser);
        viewModel.loadUser(1);
        assertEquals("John", viewModel.user.getValue().getName());
    }
}
```

### New Way (Kotlin — MockK, cleaner)

```kotlin
// Kotlin unit test — concise and readable
@ExtendWith(MockKExtension::class)
class UserViewModelTest {

    @MockK
    lateinit var mockRepository: UserRepository

    private lateinit var viewModel: UserViewModel

    @BeforeEach
    fun setup() {
        viewModel = UserViewModel(mockRepository)
    }

    @Test
    fun `loadUser returns user`() {
        coEvery { mockRepository.getUser(1) } returns User(1, "John")
        viewModel.loadUser(1)
        assertEquals("John", viewModel.user.value?.name)
    }
}
```

> ✅ Backtick test names are human-readable: `` `loadUser returns user` `` reads like a sentence.

---

## 🔑 Key Concepts

---

### 1. Unit Tests — `JUnit` + `MockK`

Unit tests test a **single function or class** in isolation. They are fast and require no device.

**Setup dependencies:**
```groovy
// build.gradle
testImplementation("junit:junit:4.13.2")
testImplementation("io.mockk:mockk:1.13.8")
testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
testImplementation("androidx.arch.core:core-testing:2.2.0")
```

**Simple example:**
```kotlin
class MathUtilsTest {

    @Test
    fun `add returns correct sum`() {
        val result = MathUtils.add(3, 4)
        assertEquals(7, result)   // expected, actual
    }

    @Test
    fun `divide by zero throws exception`() {
        assertThrows<ArithmeticException> {
            MathUtils.divide(10, 0)
        }
    }

    @Test
    fun `email validation returns true for valid email`() {
        val valid = Validator.isValidEmail("test@example.com")
        assertTrue(valid)
    }
}
```

---

### 2. `MockK` — Creating Fake Objects

When your class depends on a `Repository` or `API`, you don't want to make real network calls in tests. Use **mocks** — fake stand-ins that return whatever you tell them.

```kotlin
// Mock a repository
val mockRepo = mockk<UserRepository>()

// Tell the mock what to return (for suspend functions — coEvery)
coEvery { mockRepo.getUser(1) } returns User(id = 1, name = "Alice")

// For normal functions
every { mockRepo.getCachedUser() } returns User(id = 2, name = "Bob")

// Verify a function was called
verify { mockRepo.saveUser(any()) }
coVerify { mockRepo.getUser(1) }
```

---

### 3. ViewModel Unit Tests

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class LoginViewModelTest {

    // Replaces Main dispatcher (no Android looper in unit tests)
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    // Replaces LiveData background threading
    @get:Rule
    val instantTaskRule = InstantTaskExecutorRule()

    private val mockRepo = mockk<AuthRepository>()
    private lateinit var viewModel: LoginViewModel

    @BeforeEach
    fun setup() {
        viewModel = LoginViewModel(mockRepo)
    }

    @Test
    fun `login with valid credentials sets isLoggedIn to true`() = runTest {
        coEvery { mockRepo.login("user@test.com", "pass") } returns Result.success(Unit)

        viewModel.onEmailChange("user@test.com")
        viewModel.onPasswordChange("pass")
        viewModel.login()

        // Advance coroutines
        advanceUntilIdle()

        assertEquals(true, viewModel.uiState.value.isLoggedIn)
    }

    @Test
    fun `login with empty email sets error message`() = runTest {
        viewModel.onEmailChange("")
        viewModel.login()
        advanceUntilIdle()

        assertNotNull(viewModel.uiState.value.error)
    }
}

// Helper: MainDispatcherRule
class MainDispatcherRule : TestWatcher() {
    val dispatcher = StandardTestDispatcher()
    override fun starting(desc: Description) = Dispatchers.setMain(dispatcher)
    override fun finished(desc: Description) = Dispatchers.resetMain()
}
```

---

### 4. Room Database Tests

```kotlin
@RunWith(AndroidJUnit4::class)
class UserDaoTest {

    private lateinit var db: AppDatabase
    private lateinit var dao: UserDao

    @Before
    fun setup() {
        // In-memory database — no real file created, wiped after test
        db = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            AppDatabase::class.java
        ).allowMainThreadQueries().build()
        dao = db.userDao()
    }

    @After
    fun teardown() {
        db.close()
    }

    @Test
    fun insertUser_and_getUser_returnsCorrectUser() = runTest {
        val user = User(id = 1, name = "Charlie", email = "charlie@test.com")
        dao.insert(user)

        val retrieved = dao.getUser(1)
        assertEquals("Charlie", retrieved?.name)
    }

    @Test
    fun deleteUser_removesUser() = runTest {
        val user = User(id = 1, name = "Dana", email = "dana@test.com")
        dao.insert(user)
        dao.delete(user)

        val result = dao.getUser(1)
        assertNull(result)
    }

    @Test
    fun getAllUsers_returnsAllInserted() = runTest {
        dao.insert(User(1, "Eve", "eve@test.com"))
        dao.insert(User(2, "Frank", "frank@test.com"))

        val users = dao.getAllUsers().first() // Flow — take first emission
        assertEquals(2, users.size)
    }
}
```

---

### 5. Espresso — UI Tests (XML-based)

```kotlin
@RunWith(AndroidJUnit4::class)
class LoginActivityTest {

    @get:Rule
    val activityRule = ActivityScenarioRule(LoginActivity::class.java)

    @Test
    fun enterCredentials_clickLogin_showsHomeScreen() {
        // Type into email field
        onView(withId(R.id.etEmail))
            .perform(typeText("user@test.com"), closeSoftKeyboard())

        // Type password
        onView(withId(R.id.etPassword))
            .perform(typeText("password123"), closeSoftKeyboard())

        // Click login button
        onView(withId(R.id.btnLogin)).perform(click())

        // Check that home screen is shown
        onView(withId(R.id.tvWelcome)).check(matches(isDisplayed()))
    }

    @Test
    fun emptyEmail_showsError() {
        onView(withId(R.id.btnLogin)).perform(click())
        onView(withText("Email required")).check(matches(isDisplayed()))
    }
}
```

---

### 6. Compose UI Tests

```kotlin
@RunWith(AndroidJUnit4::class)
class CounterScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Before
    fun setup() {
        composeTestRule.setContent {
            MaterialTheme {
                CounterScreen()
            }
        }
    }

    @Test
    fun initialCountIsZero() {
        composeTestRule.onNodeWithText("Count: 0").assertIsDisplayed()
    }

    @Test
    fun clickIncrementButton_incrementsCount() {
        composeTestRule.onNodeWithText("Press Me").performClick()
        composeTestRule.onNodeWithText("Count: 1").assertIsDisplayed()
    }

    @Test
    fun clickResetButton_resetsCount() {
        // Increment first
        composeTestRule.onNodeWithText("Press Me").performClick()
        composeTestRule.onNodeWithText("Press Me").performClick()

        // Reset
        composeTestRule.onNodeWithText("Reset").performClick()
        composeTestRule.onNodeWithText("Count: 0").assertIsDisplayed()
    }
}
```

---

### 7. Hilt in Tests

```kotlin
// Replace real bindings with test fakes
@Module
@TestInstallIn(components = [SingletonComponent::class], replaces = [AppModule::class])
object TestAppModule {
    @Provides
    @Singleton
    fun provideUserRepository(): UserRepository = FakeUserRepository()
}

@HiltAndroidTest
@RunWith(AndroidJUnit4::class)
class MainActivityTest {

    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    val composeTestRule = createAndroidComposeRule<MainActivity>()

    @Inject lateinit var repository: UserRepository

    @Before
    fun inject() {
        hiltRule.inject()
    }

    @Test
    fun homeScreen_showsUserName() {
        composeTestRule.onNodeWithTag("user_name").assertIsDisplayed()
    }
}
```

---

## 💡 Test Naming Convention

```kotlin
// Format: `action_condition_expectedResult`
@Test fun `loadUser with valid id returns user`() { }
@Test fun `submit with empty form shows error`() { }
@Test fun `delete user removes user from database`() { }
```

> ✅ Backtick names are human sentences — anyone can understand what the test checks.

---

## 📊 Quick Comparison Summary

| Old (Java) | New (Kotlin) |
|---|---|
| `Mockito.mock(Repo.class)` | `mockk<Repo>()` |
| `when(mock.fn()).thenReturn(x)` | `every { mock.fn() } returns x` |
| `Mockito.verify(mock).fn()` | `verify { mock.fn() }` |
| Test names: `testLoadUser_returnsUser` | `` `loadUser returns user` `` (backtick, readable) |
| `@Before` | `@BeforeEach` (JUnit 5) |
| `assertEquals(expected, actual)` | Same — but also `assertThat(actual).isEqualTo(expected)` |
| No Compose test support | `createComposeRule()` for full Compose UI testing |
| Hilt testing was complex | `@HiltAndroidTest` + `HiltAndroidRule` |
| Room tests needed real device | `Room.inMemoryDatabaseBuilder()` runs in JVM tests |

---

## 🏁 Full Test Matrix for a Feature

| What to Test | Tool | Location |
|---|---|---|
| Pure logic (validators, mappers) | JUnit | `test/` |
| ViewModel state changes | JUnit + MockK + `runTest` | `test/` |
| Repository (network mocking) | JUnit + MockK | `test/` |
| Room DAO queries | Room in-memory | `androidTest/` |
| Compose UI interactions | Compose UI test rule | `androidTest/` |
| Full activity flows | Espresso | `androidTest/` |
