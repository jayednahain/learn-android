# Step 12 — ViewModel

---

## 📖 What Is It? (Simple Definition)

Imagine you're filling out a long form on your phone, and then you accidentally rotate your screen. In the old days, everything you typed would be **gone** — the Activity is destroyed and recreated on rotation!

**ViewModel** is like a backpack that survives screen rotation. Your Activity dies and is reborn, but your ViewModel (with all the data) keeps living and hands the data right back to the new Activity.

Also: ViewModel is the "brain" of a screen — it holds ALL the business logic and data, keeping Activity/Fragment as "dumb" views that only show information.

---

## 🎯 Where Do We Use It?

- Every single screen should have a ViewModel
- Fetch and hold data that the UI needs
- Handle user actions (clicks, form submissions)
- Survive screen rotation without losing data
- Share data between Fragments via Activity ViewModel

---

## 🔄 ViewModel Lifecycle Diagram

```
Activity is created
       ↓
ViewModel is created (or retrieved if already exists)
       ↓
Activity is destroyed (e.g., screen rotated)
       ↓
       ← ViewModel SURVIVES! Data is kept! →
       ↓
NEW Activity instance is created
       ↓
Same ViewModel is handed to new Activity
       ↓
UI is restored with saved data ✅

(Only dies when user presses Back or finishes Activity)
```

---

## ☕ Java vs Kotlin — What Changed?

### Old Java Way — Manual State Management (painful!)

```java
// Java — save state manually in onSaveInstanceState (very limited, only small data!)
@Override
protected void onSaveInstanceState(Bundle outState) {
    super.onSaveInstanceState(outState);
    outState.putString("SEARCH_QUERY", searchQuery);
    outState.putInt("SCROLL_POSITION", scrollPosition);
    // Can't save complex objects or lists easily!
}

@Override
protected void onCreate(Bundle savedInstanceState) {
    if (savedInstanceState != null) {
        searchQuery = savedInstanceState.getString("SEARCH_QUERY");
    }
    // Also manually re-fetch all data — complex!
}
```

### New Kotlin Way — ViewModel handles everything!

```kotlin
// ViewModel — all data lives here, survives rotation automatically
class UserViewModel : ViewModel() {

    // MutableStateFlow — holds current data, UI observes this
    private val _users = MutableStateFlow<List<User>>(emptyList())
    val users: StateFlow<List<User>> = _users.asStateFlow()

    // This survives screen rotation!
    var searchQuery = ""

    init {
        // Auto-loads when ViewModel is first created
        loadUsers()
    }

    fun loadUsers() {
        viewModelScope.launch {
            val result = withContext(Dispatchers.IO) {
                repository.getUsers()
            }
            _users.value = result
        }
    }
}

// In Activity/Fragment — just get the ViewModel with one line!
private val viewModel: UserViewModel by viewModels()
// That's it — Android handles creation and data survival!
```

---

### Getting a ViewModel — 3 Ways

```kotlin
// 1. In Activity — basic
private val viewModel: UserViewModel by viewModels()

// 2. In Activity — with custom factory (for ViewModel that needs dependencies)
private val viewModel: UserViewModel by viewModels {
    UserViewModelFactory(repository)
}

// 3. In Fragment — scoped to Fragment
private val viewModel: UserViewModel by viewModels()

// 4. In Fragment — scoped to PARENT ACTIVITY (shared between Fragments!)
private val sharedViewModel: SharedViewModel by activityViewModels()
// Both HomeFragment and ProfileFragment can share the same ViewModel this way
```

---

### `AndroidViewModel` — When You Need Context

```kotlin
// Regular ViewModel — can't access Context (don't!), use for most cases
class UserViewModel : ViewModel() { ... }

// AndroidViewModel — when you genuinely need Application context
// (app-wide resources, SharedPreferences before DataStore, etc.)
class AppViewModel(application: Application) : AndroidViewModel(application) {
    fun getAppName(): String {
        // Access application context SAFELY
        return application.getString(R.string.app_name)
    }
}

// Usage (same as regular ViewModel!)
private val viewModel: AppViewModel by viewModels()
```

> ⚠️ Never store an `Activity` or `Fragment` reference inside ViewModel! The ViewModel outlives the screen — this causes memory leaks.

---

### `viewModelScope` — Coroutines Tied to ViewModel

```kotlin
class ProductViewModel(private val repo: ProductRepository) : ViewModel() {

    private val _products = MutableStateFlow<List<Product>>(emptyList())
    val products: StateFlow<List<Product>> = _products.asStateFlow()

    private val _isLoading = MutableStateFlow(false)
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()

    // viewModelScope — coroutine is AUTOMATICALLY cancelled when ViewModel dies
    // No need to manually cancel! No memory leaks!
    fun loadProducts() {
        viewModelScope.launch {
            _isLoading.value = true
            try {
                _products.value = repo.getProducts()
            } finally {
                _isLoading.value = false
            }
        }
    }

    fun deleteProduct(id: Int) {
        viewModelScope.launch {
            repo.delete(id)
            loadProducts() // refresh
        }
    }

    // Called when user leaves screen (Back pressed / Activity finished)
    override fun onCleared() {
        super.onCleared()
        // viewModelScope coroutines are already cancelled automatically
        // Only use this for other manual cleanup
    }
}
```

---

### Sharing ViewModel Between Fragments

```kotlin
// SharedViewModel — lives in Activity scope
class CartViewModel : ViewModel() {
    private val _cartItems = MutableStateFlow<List<CartItem>>(emptyList())
    val cartItems: StateFlow<List<CartItem>> = _cartItems.asStateFlow()
    val cartCount: StateFlow<Int> = cartItems.map { it.size }.stateIn(
        viewModelScope, SharingStarted.WhileSubscribed(), 0
    )

    fun addToCart(item: CartItem) {
        _cartItems.value = _cartItems.value + item
    }
}

// ProductListFragment — adds items to cart
class ProductListFragment : Fragment() {
    private val cartViewModel: CartViewModel by activityViewModels()
    // ... onAddToCartClicked:
    fun onAddToCartClicked(product: Product) {
        cartViewModel.addToCart(CartItem(product.id, product.name, product.price))
    }
}

// CartFragment — shows cart items
class CartFragment : Fragment() {
    private val cartViewModel: CartViewModel by activityViewModels()
    // Both fragments share the SAME ViewModel instance — changes in one
    // are immediately visible in the other!
}
```

---

## 🔑 Key Concepts

| Concept | What It Does | Key Note |
|---------|-------------|---------|
| `ViewModel` | Holds UI data, survives rotation | One ViewModel per screen |
| `viewModels()` | Create/retrieve ViewModel | In Activity or Fragment |
| `activityViewModels()` | Shared ViewModel via Activity | For inter-Fragment communication |
| `viewModelScope` | Coroutine tied to ViewModel | Auto-cancelled on ViewModel clear |
| `AndroidViewModel` | ViewModel with Application context | Only when you need Context |
| `onCleared()` | Called when ViewModel dies | Clean up non-coroutine resources |

---

## 💡 Good Example — Search Screen with ViewModel

```kotlin
class SearchViewModel(private val repo: UserRepository) : ViewModel() {

    private val _query = MutableStateFlow("")
    val query: StateFlow<String> = _query.asStateFlow()

    // When query changes, automatically search!
    val searchResults: StateFlow<List<User>> = _query
        .debounce(400)  // wait 400ms after last keystroke (don't spam the API!)
        .filter { it.length >= 2 }  // only if at least 2 chars typed
        .flatMapLatest { query ->
            flow {
                emit(repo.searchUsers(query))  // fetch from repo
            }.flowOn(Dispatchers.IO)
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun onQueryChanged(newQuery: String) {
        _query.value = newQuery
    }
}

class SearchFragment : Fragment() {
    private val viewModel: SearchViewModel by viewModels()
    private var _binding: FragmentSearchBinding? = null
    private val binding get() = _binding!!

    private val adapter = UserListAdapter { user -> navigateToProfile(user.id) }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding.recyclerView.adapter = adapter

        // Connect search input to ViewModel
        binding.searchInput.addTextChangedListener { text ->
            viewModel.onQueryChanged(text.toString())
        }

        // Observe results
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.searchResults.collect { users ->
                    adapter.submitList(users)
                    binding.emptyState.isVisible = users.isEmpty()
                }
            }
        }
    }

    override fun onDestroyView() { super.onDestroyView(); _binding = null }
}
```
