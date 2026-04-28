# Step 14 — Repository Pattern

---

## 📖 What Is It? (Simple Definition)

Imagine you work at a company and need a document. You don't go to every room yourself — you ask the **librarian (Repository)**, who knows where everything is stored (network, database, cache).

The **Repository** is the single place that manages all your data. The ViewModel only talks to the Repository and doesn't care if data came from the internet, local database, or cache.

---

## 🎯 Where Do We Use It?

- Any app that gets data from multiple sources (API + database)
- When you want offline support (cache data locally)
- When you want to swap data sources easily (e.g., switch API providers)
- Any production Android app!

---

## 🔄 Architecture Diagram

```
┌─────────────────────────────────────┐
│              UI Layer               │
│   Activity / Fragment (dumb views)  │
└──────────────┬──────────────────────┘
               │ observes StateFlow / LiveData
┌──────────────▼──────────────────────┐
│           ViewModel Layer           │
│  Holds UI state, calls Repository   │
└──────────────┬──────────────────────┘
               │ calls repository functions
┌──────────────▼──────────────────────┐
│          Repository Layer           │  ← YOU ARE HERE
│  Single source of truth for data    │
│  Decides: network? cache? database? │
└───────┬──────────────┬──────────────┘
        │              │
┌───────▼───────┐ ┌────▼──────────────┐
│  Remote API   │ │  Local Database   │
│  (Retrofit)   │ │     (Room)        │
└───────────────┘ └───────────────────┘
```

---

## ☕ Java vs Kotlin — What Changed?

### Old Java Way — ViewModel directly calling API (bad practice!)

```java
// Java ViewModel calling API directly — wrong architecture
public class UserViewModel extends ViewModel {
    private ApiService apiService = RetrofitClient.getService(); // direct dependency!
    private AppDatabase db = AppDatabase.getInstance();          // direct dependency!

    public void loadUser(int id) {
        // ViewModel knows about both API and Database — too much responsibility!
        apiService.getUser(id).enqueue(new Callback<User>() {
            @Override
            public void onResponse(...) {
                db.userDao().insert(response.body());
                // update LiveData...
            }
        });
    }
}
```

### New Way — Repository Pattern ✅

```kotlin
// Repository — the only class that knows about data sources
class UserRepository(
    private val apiService: ApiService,      // Retrofit (network)
    private val userDao: UserDao             // Room (local database)
) {
    // Single source of truth: always return Room data.
    // Room is updated from network in the background.
    fun getUsers(): Flow<List<User>> = userDao.getAllUsers()  // always from DB

    suspend fun refreshUsers() {
        // Fetch from network, save to DB. Room Flow auto-emits new data!
        val users = apiService.getUsers()
        userDao.insertAll(users)
    }

    suspend fun getUser(id: Int): User {
        // Try cache first, then network
        return userDao.getUser(id) ?: run {
            val user = apiService.getUser(id)
            userDao.insert(user)
            user
        }
    }

    suspend fun deleteUser(id: Int) {
        userDao.delete(id)
        apiService.deleteUser(id)  // sync with server
    }
}

// ViewModel — only knows about Repository, nothing else!
class UserViewModel(private val repository: UserRepository) : ViewModel() {

    val users: StateFlow<List<User>> = repository.getUsers()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun refresh() {
        viewModelScope.launch {
            try {
                repository.refreshUsers()
            } catch (e: Exception) {
                // Handle error
            }
        }
    }
}
```

---

### Offline-First Architecture — The Gold Standard

```kotlin
// Offline-first: Database is the ONLY source for UI.
// Network just syncs data to the database.
// Result: App works perfectly even without internet!

class NewsRepository(
    private val newsApi: NewsApi,
    private val newsDao: NewsDao
) {
    // UI always observes database Flow (works offline!)
    fun getTopNews(): Flow<List<Article>> = newsDao.getTopArticles()

    // Call this to sync latest news from server
    suspend fun syncNews() {
        val articles = newsApi.getTopHeadlines()  // fetch from network
        newsDao.insertAll(articles)               // save to DB
        // Flow above auto-emits the new articles to UI!
    }

    // Called when app starts or user pulls to refresh
    suspend fun refreshIfNeeded() {
        val lastUpdate = newsDao.getLastUpdateTime()
        val isStale = System.currentTimeMillis() - lastUpdate > 30 * 60 * 1000 // 30 min
        if (isStale) syncNews()
    }
}
```

---

### Repository with Multiple Data Sources (Cache Strategy)

```kotlin
class WeatherRepository(
    private val weatherApi: WeatherApi,
    private val weatherDao: WeatherDao,
    private val prefs: DataStore<Preferences>
) {
    private val CACHE_DURATION = 10 * 60 * 1000L // 10 minutes

    suspend fun getWeather(city: String): Result<Weather> {
        // 1. Check memory/DB cache
        val cached = weatherDao.getWeather(city)
        val cacheAge = System.currentTimeMillis() - (cached?.timestamp ?: 0)

        return if (cached != null && cacheAge < CACHE_DURATION) {
            // 2. Cache is fresh — return immediately
            Result.success(cached.toDomainModel())
        } else {
            // 3. Cache is stale or missing — fetch from network
            try {
                val fresh = weatherApi.getWeather(city)
                weatherDao.insert(fresh.toEntity())  // update cache
                Result.success(fresh.toDomainModel())
            } catch (e: Exception) {
                if (cached != null) {
                    // 4. Network failed but we have stale cache — use it
                    Result.success(cached.toDomainModel())
                } else {
                    Result.failure(e)
                }
            }
        }
    }
}
```

---

## 🔑 Key Concepts

| Concept | What It Means | Key Note |
|---------|--------------|---------|
| Repository | Single manager for data | ViewModel never touches API/DB directly |
| Single source of truth | DB is the master copy | UI reads from DB, network syncs to DB |
| Offline-first | App works without internet | DB → UI, network → DB |
| Cache | Save network data locally | Faster, works offline |
| `Result<T>` | Wrapper for success or failure | Better than try/catch everywhere |
| Data source | Where data comes from | Remote API, Local DB, DataStore |

---

## 💡 Good Example — Product Repository

```kotlin
interface ProductRepository {
    fun getProducts(): Flow<List<Product>>
    suspend fun refreshProducts()
    suspend fun getProductDetail(id: Int): Product
    suspend fun addToFavorites(productId: Int)
    suspend fun removeFromFavorites(productId: Int)
    fun getFavorites(): Flow<List<Product>>
}

class ProductRepositoryImpl(
    private val api: ProductApi,
    private val productDao: ProductDao,
    private val favoriteDao: FavoriteDao
) : ProductRepository {

    // Always serve from database (works offline!)
    override fun getProducts(): Flow<List<Product>> =
        productDao.getAllProducts().map { entities -> entities.map { it.toProduct() } }

    // Sync with network (call on pull-to-refresh or app start)
    override suspend fun refreshProducts() {
        val remoteProducts = api.getProducts()
        productDao.deleteAll()
        productDao.insertAll(remoteProducts.map { it.toEntity() })
    }

    override suspend fun getProductDetail(id: Int): Product {
        // Check local DB first
        productDao.getProduct(id)?.let { return it.toProduct() }
        // Not in DB — fetch from API and cache it
        val remote = api.getProduct(id)
        productDao.insert(remote.toEntity())
        return remote.toProduct()
    }

    override suspend fun addToFavorites(productId: Int) {
        favoriteDao.insert(FavoriteEntity(productId))
    }

    override suspend fun removeFromFavorites(productId: Int) {
        favoriteDao.delete(productId)
    }

    override fun getFavorites(): Flow<List<Product>> =
        favoriteDao.getAllFavorites()
            .map { favorites -> favorites.mapNotNull { productDao.getProduct(it.productId)?.toProduct() } }
}
```
