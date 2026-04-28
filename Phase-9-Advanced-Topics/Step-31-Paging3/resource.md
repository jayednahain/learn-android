# Step 31 — Paging 3 (Large Lists)

---

## 📖 What Is It? (Simple Explanation)

Imagine a **huge library** 📚 with 10,000 books. You don't carry all 10,000 books home at once — you ask the librarian for **10 books at a time**. When you finish reading those, you ask for the **next 10**.

**Paging 3** does exactly this for your app. Instead of loading 10,000 items from a server/database all at once (which would crash the app or make it very slow), Paging 3 loads small **pages** of data as the user scrolls. When they reach the bottom, it automatically loads the next page.

---

## 🧭 Where Do We Use It?

- Social media feeds (Instagram, Twitter-style lists)
- Product catalog (e.g., 500+ products from an API)
- Search results with many pages
- Chat message history
- Any list where you don't know the total size upfront

---

## 🗺️ Workflow / How It Works

```
User scrolls down
      │
      ▼
PagingDataAdapter detects reaching near the end
      │
      ▼
Calls PagingSource.load(page = 3)
      │
      ▼
PagingSource fetches items 21-30 from API/Database
      │
      ▼
Returns new items → adapter appends them to the list
      │
      ▼
User sees more items appear — no manual "load more" button needed


With RemoteMediator (offline-first):
API call → save to Room  → display from Room
   ↑                              │
   └──────── refresh ─────────────┘
```

---

## ☕ Java vs Kotlin (Paging 3) — What Changed?

### Old Way (Java — manual pagination)

```java
// Java — completely manual, fragile
int currentPage = 1;
boolean isLoading = false;

recyclerView.addOnScrollListener(new RecyclerView.OnScrollListener() {
    @Override
    public void onScrolled(@NonNull RecyclerView rv, int dx, int dy) {
        if (!isLoading && !layoutManager.canScrollVertically(1)) {
            isLoading = true;
            currentPage++;
            loadPage(currentPage);  // custom function
        }
    }
});

void loadPage(int page) {
    apiService.getItems(page).enqueue(new Callback<List<Item>>() {
        @Override
        public void onResponse(Call<List<Item>> call, Response<List<Item>> response) {
            adapter.addItems(response.body());  // manual add
            isLoading = false;
        }
        @Override public void onFailure(Call<List<Item>> call, Throwable t) { isLoading = false; }
    });
}
```

> Problems: scroll detection is fragile, error handling is manual, loading state logic is scattered everywhere.

### New Way (Kotlin + Paging 3)

```kotlin
// PagingSource — one class, handles everything
class ItemsPagingSource(private val api: ApiService) : PagingSource<Int, Item>() {
    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Item> {
        return try {
            val page = params.key ?: 1
            val response = api.getItems(page = page, pageSize = params.loadSize)
            LoadResult.Page(
                data = response.items,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.items.isEmpty()) null else page + 1
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }
    override fun getRefreshKey(state: PagingState<Int, Item>) = state.anchorPosition
}
```

> ✅ Paging 3 handles: auto-loading next page, retry on error, empty states, DiffUtil — all built in.

---

## 🔑 Key Concepts

---

### 1. Three Core Components

```
PagingSource   → knows HOW to fetch one page of data
Pager          → orchestrates paging config and creates PagingData flow
PagingDataAdapter → RecyclerView adapter that receives PagingData
```

---

### 2. `PagingSource` — The Data Fetcher

```kotlin
class NewsPagingSource(private val api: NewsApi) : PagingSource<Int, Article>() {

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Article> {
        val page = params.key ?: 1  // start from page 1
        return try {
            val response = api.getArticles(page = page, pageSize = params.loadSize)
            LoadResult.Page(
                data = response.articles,
                prevKey = if (page == 1) null else page - 1,  // null = no previous page
                nextKey = if (response.articles.isEmpty()) null else page + 1 // null = last page
            )
        } catch (e: IOException) {
            LoadResult.Error(e)   // network down
        } catch (e: HttpException) {
            LoadResult.Error(e)   // server error
        }
    }

    override fun getRefreshKey(state: PagingState<Int, Article>): Int? {
        return state.anchorPosition  // where to reload after invalidation
    }
}
```

---

### 3. `Pager` — Setting Up the Config

```kotlin
// In Repository
fun getNewsPagingFlow(): Flow<PagingData<Article>> {
    return Pager(
        config = PagingConfig(
            pageSize = 20,              // load 20 items per page
            prefetchDistance = 5,       // load next page when 5 items from end
            enablePlaceholders = false
        ),
        pagingSourceFactory = { NewsPagingSource(api) }
    ).flow
}
```

---

### 4. `PagingDataAdapter` — RecyclerView Adapter

```kotlin
class ArticleAdapter : PagingDataAdapter<Article, ArticleViewHolder>(DIFF_CALLBACK) {

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ArticleViewHolder {
        val binding = ItemArticleBinding.inflate(LayoutInflater.from(parent.context), parent, false)
        return ArticleViewHolder(binding)
    }

    override fun onBindViewHolder(holder: ArticleViewHolder, position: Int) {
        val article = getItem(position) // can be null while loading
        if (article != null) holder.bind(article)
    }

    companion object {
        val DIFF_CALLBACK = object : DiffUtil.ItemCallback<Article>() {
            override fun areItemsTheSame(old: Article, new: Article) = old.id == new.id
            override fun areContentsTheSame(old: Article, new: Article) = old == new
        }
    }
}
```

---

### 5. In ViewModel & Fragment

```kotlin
// ViewModel
@HiltViewModel
class NewsViewModel @Inject constructor(private val repository: NewsRepository) : ViewModel() {
    val articles = repository.getNewsPagingFlow().cachedIn(viewModelScope)
    // cachedIn — keeps data alive across screen rotations
}

// Fragment
class NewsFragment : Fragment() {
    private val viewModel: NewsViewModel by viewModels()
    private val adapter = ArticleAdapter()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        binding.recyclerView.adapter = adapter.withLoadStateFooter(
            footer = LoadingStateAdapter { adapter.retry() }  // shows spinner at bottom
        )

        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.articles.collectLatest { pagingData ->
                adapter.submitData(pagingData)
            }
        }
    }
}
```

---

### 6. `LoadState` — Handle Loading, Error, Empty

```kotlin
// Show progress bar when first page is loading
viewLifecycleOwner.lifecycleScope.launch {
    adapter.loadStateFlow.collectLatest { loadStates ->
        binding.progressBar.isVisible = loadStates.refresh is LoadState.Loading
        binding.errorView.isVisible = loadStates.refresh is LoadState.Error
        binding.emptyView.isVisible = loadStates.refresh is LoadState.NotLoading
                                      && adapter.itemCount == 0
    }
}
```

---

### 7. `RemoteMediator` — Network + Database (Offline-First)

```kotlin
@OptIn(ExperimentalPagingApi::class)
class ArticleRemoteMediator(
    private val db: AppDatabase,
    private val api: NewsApi
) : RemoteMediator<Int, Article>() {

    override suspend fun load(loadType: LoadType, state: PagingState<Int, Article>): MediatorResult {
        return try {
            val page = when (loadType) {
                LoadType.REFRESH -> 1
                LoadType.PREPEND -> return MediatorResult.Success(endOfPaginationReached = true)
                LoadType.APPEND -> getNextPageFromDb() ?: return MediatorResult.Success(true)
            }
            val articles = api.getArticles(page = page, pageSize = state.config.pageSize)
            db.withTransaction {
                if (loadType == LoadType.REFRESH) db.articleDao().clearAll()
                db.articleDao().insertAll(articles)
            }
            MediatorResult.Success(endOfPaginationReached = articles.isEmpty())
        } catch (e: Exception) {
            MediatorResult.Error(e)
        }
    }
}
```

---

## 💡 Full Setup — Compose + Paging 3

```kotlin
// Build dependency
// implementation("androidx.paging:paging-compose:3.2.1")

@Composable
fun NewsListScreen(viewModel: NewsViewModel = hiltViewModel()) {
    val lazyPagingItems = viewModel.articles.collectAsLazyPagingItems()

    LazyColumn {
        items(
            count = lazyPagingItems.itemCount,
            key = lazyPagingItems.itemKey { it.id }
        ) { index ->
            val article = lazyPagingItems[index]
            if (article != null) {
                ArticleCard(article = article)
            }
        }

        // Footer loading / error state
        when (lazyPagingItems.loadState.append) {
            is LoadState.Loading -> item { CircularProgressIndicator(modifier = Modifier.fillMaxWidth().wrapContentWidth()) }
            is LoadState.Error -> item {
                TextButton(onClick = { lazyPagingItems.retry() }) { Text("Retry") }
            }
            else -> Unit
        }
    }

    // Full-screen loading for first page
    if (lazyPagingItems.loadState.refresh is LoadState.Loading) {
        Box(modifier = Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
            CircularProgressIndicator()
        }
    }
}
```

---

## 📊 Quick Comparison Summary

| Old (Manual Pagination) | Paging 3 |
|---|---|
| `OnScrollListener` + flags | Automatic — no listener needed |
| Manual page counter | `PagingSource` key system |
| Custom "isLoading" boolean | `LoadState.Loading` built-in |
| Manual error retry | `adapter.retry()` one call |
| Manual `DiffUtil` | Built into `PagingDataAdapter` |
| `notifyDataSetChanged()` | `submitData(pagingData)` |
| Pages lost on rotation | `cachedIn(viewModelScope)` |
| Network OR database | `RemoteMediator` handles both |
