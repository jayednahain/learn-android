# Step 18 — Retrofit + OkHttp (Networking)

---

## 📖 What Is It? (Simple Definition)

Your Android app needs to talk to the internet — like asking a server "give me the list of users" and the server replies with JSON data.

**Retrofit** is like a smart telephone operator. You define what you want to ask (API interface), and Retrofit handles all the hard work of making the actual HTTP call, converting the JSON response to Kotlin objects, and giving the result back.

**OkHttp** is the actual phone line underneath — it manages connections, timeouts, logging, and authentication headers.

---

## 🎯 Where Do We Use It?

- Fetching data from any REST API (your backend, Firebase, third-party APIs)
- Sending form data, uploading files
- Authenticating with a server
- Any data that comes from the internet

---

## 🔄 Retrofit Architecture Diagram

```
Your App
    │
    ├── Define API interface (what endpoints exist)
    │
    ▼
Retrofit Instance
    │
    ├── OkHttp Client (adds auth headers, logs calls)
    ├── Converter (Gson/Moshi — JSON ↔ Kotlin objects)
    │
    ▼
HTTP Request → [INTERNET] → Your Backend Server
HTTP Response ← [INTERNET] ← JSON data
    │
    ▼
Kotlin Data Class (auto-converted by Gson/Moshi)
    │
    ▼
Repository → ViewModel → UI
```

---

## ☕ Java vs Kotlin — What Changed?

### Old Java Way — HttpURLConnection (100+ lines per call!)

```java
// Java — painful manual HTTP call
public User fetchUser(int id) throws IOException {
    URL url = new URL("https://api.example.com/users/" + id);
    HttpURLConnection connection = (HttpURLConnection) url.openConnection();
    connection.setRequestMethod("GET");
    connection.setRequestProperty("Authorization", "Bearer " + token);

    StringBuilder result = new StringBuilder();
    BufferedReader reader = new BufferedReader(
        new InputStreamReader(connection.getInputStream())
    );
    String line;
    while ((line = reader.readLine()) != null) result.append(line);
    reader.close();
    connection.disconnect();

    // Manually parse JSON...
    JSONObject json = new JSONObject(result.toString());
    return new User(json.getInt("id"), json.getString("name"));
}
// 30+ lines per API call, error-prone, verbose...
```

### New Way — Retrofit + Coroutines ✅

```kotlin
// Step 1: Add dependencies to build.gradle
// implementation("com.squareup.retrofit2:retrofit:2.9.0")
// implementation("com.squareup.retrofit2:converter-gson:2.9.0")
// implementation("com.squareup.okhttp3:okhttp:4.12.0")
// implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")

// Step 2: Data classes matching API JSON
data class UserResponse(
    @SerializedName("id") val id: Int,
    @SerializedName("name") val name: String,
    @SerializedName("email") val email: String,
    @SerializedName("profile_url") val profileUrl: String?  // snake_case JSON → camelCase Kotlin
)

data class CreateUserRequest(
    val name: String,
    val email: String,
    val password: String
)

// Step 3: API interface — just annotations, no implementation needed!
interface UserApi {
    @GET("users")
    suspend fun getUsers(): List<UserResponse>

    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Int): UserResponse

    @GET("users")
    suspend fun searchUsers(@Query("name") name: String): List<UserResponse>

    // GET with multiple query params
    @GET("users")
    suspend fun getFilteredUsers(
        @Query("page") page: Int,
        @Query("limit") limit: Int = 20,
        @Query("sort") sort: String = "name"
    ): UsersPagedResponse

    @POST("users")
    suspend fun createUser(@Body request: CreateUserRequest): UserResponse

    @PUT("users/{id}")
    suspend fun updateUser(@Path("id") id: Int, @Body user: UserResponse): UserResponse

    @DELETE("users/{id}")
    suspend fun deleteUser(@Path("id") id: Int)

    @Multipart
    @POST("users/{id}/avatar")
    suspend fun uploadAvatar(
        @Path("id") id: Int,
        @Part avatar: MultipartBody.Part
    ): UserResponse
}
```

---

### Setting Up Retrofit Client

```kotlin
object NetworkModule {

    private const val BASE_URL = "https://api.example.com/v1/"

    // OkHttp client with interceptors
    private val okHttpClient = OkHttpClient.Builder()
        // Logging interceptor — logs all requests and responses (use only in DEBUG!)
        .addInterceptor(HttpLoggingInterceptor().apply {
            level = if (BuildConfig.DEBUG)
                HttpLoggingInterceptor.Level.BODY
            else
                HttpLoggingInterceptor.Level.NONE
        })
        // Auth interceptor — adds Bearer token to every request automatically
        .addInterceptor { chain ->
            val token = TokenManager.getToken()  // get from DataStore
            val request = chain.request().newBuilder()
                .addHeader("Authorization", "Bearer $token")
                .addHeader("Accept", "application/json")
                .build()
            chain.proceed(request)
        }
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .build()

    // Retrofit instance
    private val retrofit = Retrofit.Builder()
        .baseUrl(BASE_URL)
        .client(okHttpClient)
        .addConverterFactory(GsonConverterFactory.create())  // JSON → Kotlin class
        .build()

    // Create API service
    val userApi: UserApi = retrofit.create(UserApi::class.java)
}
```

---

### Using in Repository (with Error Handling)

```kotlin
class UserRepositoryImpl(private val api: UserApi) : UserRepository {

    override suspend fun getUsers(): Result<List<User>> {
        return try {
            val response = api.getUsers()
            Result.success(response.map { it.toUser() })
        } catch (e: HttpException) {
            // 4xx, 5xx HTTP errors
            Result.failure(Exception("Server error: ${e.code()} ${e.message()}"))
        } catch (e: IOException) {
            // Network errors (no internet, timeout)
            Result.failure(Exception("No internet connection"))
        }
    }

    override suspend fun createUser(name: String, email: String, password: String): Result<User> {
        return try {
            val request = CreateUserRequest(name, email, password)
            val response = api.createUser(request)
            Result.success(response.toUser())
        } catch (e: HttpException) {
            when (e.code()) {
                409 -> Result.failure(Exception("Email already registered"))
                422 -> Result.failure(Exception("Invalid data provided"))
                else -> Result.failure(Exception("Server error: ${e.code()}"))
            }
        } catch (e: IOException) {
            Result.failure(Exception("No internet connection"))
        }
    }
}
```

---

### Handling Paginated API Responses

```kotlin
data class UsersPagedResponse(
    @SerializedName("data") val users: List<UserResponse>,
    @SerializedName("total") val total: Int,
    @SerializedName("page") val page: Int,
    @SerializedName("has_more") val hasMore: Boolean
)

// Using Paging 3 with Retrofit (covered in Step 31)
class UserPagingSource(private val api: UserApi) : PagingSource<Int, User>() {
    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, User> {
        val page = params.key ?: 1
        return try {
            val response = api.getFilteredUsers(page = page, limit = params.loadSize)
            LoadResult.Page(
                data = response.users.map { it.toUser() },
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.hasMore) page + 1 else null
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }
    override fun getRefreshKey(state: PagingState<Int, User>) = state.anchorPosition?.let {
        state.closestPageToPosition(it)?.prevKey?.plus(1) ?: state.closestPageToPosition(it)?.nextKey?.minus(1)
    }
}
```

---

## 🔑 Key Concepts

| Annotation/Class | Purpose | Example |
|-----------------|---------|---------|
| `@GET("path")` | HTTP GET request | `@GET("users")` |
| `@POST("path")` | HTTP POST request | `@POST("users")` |
| `@PUT("path")` | HTTP PUT request | `@PUT("users/{id}")` |
| `@DELETE("path")` | HTTP DELETE request | `@DELETE("users/{id}")` |
| `@Path` | URL segment variable | `@Path("id") id: Int` |
| `@Query` | URL query parameter | `@Query("page") page: Int` |
| `@Body` | Request body | `@Body user: UserRequest` |
| `@Header` | Single header | `@Header("Auth") token: String` |
| `OkHttp Interceptor` | Modify all requests | Add token, log, retry |
| `Gson/Moshi` | JSON ↔ Kotlin conversion | `@SerializedName("user_id")` |
| `HttpException` | 4xx/5xx errors | `e.code() == 401` |
| `IOException` | Network error (no internet) | `e.message == "timeout"` |

---

## 💡 Good Example — Full Network Layer

```kotlin
// 1. API Response models
data class LoginResponse(
    @SerializedName("access_token") val token: String,
    @SerializedName("user") val user: UserResponse
)

// 2. API interface
interface AuthApi {
    @POST("auth/login")
    suspend fun login(@Body request: LoginRequest): LoginResponse

    @POST("auth/refresh")
    suspend fun refreshToken(@Body token: RefreshTokenRequest): LoginResponse

    @DELETE("auth/logout")
    suspend fun logout()
}

// 3. Auth interceptor with token refresh
class AuthInterceptor(private val tokenProvider: TokenProvider) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val token = tokenProvider.getAccessToken()
        val request = chain.request().newBuilder()
            .header("Authorization", "Bearer $token")
            .build()
        
        val response = chain.proceed(request)
        
        // If 401 (token expired), try refreshing
        if (response.code == 401) {
            val newToken = tokenProvider.refreshToken()  // synchronous refresh
            if (newToken != null) {
                val retried = chain.request().newBuilder()
                    .header("Authorization", "Bearer $newToken")
                    .build()
                return chain.proceed(retried)
            }
        }
        return response
    }
}

// 4. ViewModel using auth repository
class LoginViewModel(private val authRepo: AuthRepository) : ViewModel() {
    private val _loginState = MutableStateFlow<LoginUiState>(LoginUiState.Idle)
    val loginState: StateFlow<LoginUiState> = _loginState.asStateFlow()

    fun login(email: String, password: String) {
        viewModelScope.launch {
            _loginState.value = LoginUiState.Loading
            val result = authRepo.login(email, password)
            _loginState.value = when {
                result.isSuccess -> LoginUiState.Success(result.getOrNull()!!)
                else -> LoginUiState.Error(result.exceptionOrNull()?.message ?: "Login failed")
            }
        }
    }
}

sealed class LoginUiState {
    object Idle : LoginUiState()
    object Loading : LoginUiState()
    data class Success(val user: User) : LoginUiState()
    data class Error(val message: String) : LoginUiState()
}
```
