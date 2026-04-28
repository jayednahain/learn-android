# Step 9 — RecyclerView (Replaces ListView)

---

## 📖 What Is It? (Simple Definition)

Imagine you have 1000 photos to show in a scrollable list. If you loaded all 1000 at once, your phone would freeze!

**RecyclerView** is smart — it only keeps the visible items in memory. When you scroll down and an item goes off-screen, RecyclerView **recycles** that item's view and reuses it for the new item coming in. That's why it's called "Recycler" View!

Think of it like a conveyor belt at a sushi restaurant — the plates keep coming around, but only the ones in front of you are filled with fresh sushi.

---

## 🎯 Where Do We Use It?

- Lists of products, messages, users, posts
- Chat screens (messages scrolling)
- Photo grids (Instagram-like)
- News feeds
- Any list with **more than a few items**

> 💡 Rule: If you have more than 4-5 items that scroll → use RecyclerView.

---

## 🔄 RecyclerView Architecture Diagram

```
Data List (from ViewModel/Repository)
         ↓
    RecyclerView.Adapter
    ┌─────────────────────────────┐
    │  onCreateViewHolder()       │ ← inflate item XML (create the plate)
    │  onBindViewHolder()         │ ← fill with data (put sushi on plate)
    │  getItemCount()             │ ← how many items total
    └─────────────────────────────┘
         ↓
    ViewHolder (holds item views)
         ↓
    LayoutManager (decides arrangement)
    ├── LinearLayoutManager   → vertical/horizontal list
    ├── GridLayoutManager     → grid (2, 3 columns)
    └── StaggeredGridLayout   → Pinterest style
         ↓
    RecyclerView (the scrollable container on screen)
```

---

## ☕ Java vs Kotlin — What Changed?

### RecyclerView vs ListView

```
ListView (Old Era):
- No ViewHolder = slow, freezes on scroll
- No DiffUtil = entire list redraws on any change
- Limited animations
- Min code: ~50 lines

RecyclerView (Modern):
- ViewHolder is mandatory = always smooth
- DiffUtil = only changed items redraw
- Built-in animations
- ListAdapter makes it even easier
```

### 1. Basic Adapter (Java → Kotlin)

```java
// Java Adapter — ~60 lines
public class UserAdapter extends RecyclerView.Adapter<UserAdapter.ViewHolder> {
    private List<User> users;

    public UserAdapter(List<User> users) {
        this.users = users;
    }

    @NonNull
    @Override
    public ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(parent.getContext())
                .inflate(R.layout.item_user, parent, false);
        return new ViewHolder(view);
    }

    @Override
    public void onBindViewHolder(@NonNull ViewHolder holder, int position) {
        User user = users.get(position);
        holder.tvName.setText(user.getName());
    }

    @Override
    public int getItemCount() { return users.size(); }

    public static class ViewHolder extends RecyclerView.ViewHolder {
        TextView tvName;
        public ViewHolder(@NonNull View itemView) {
            super(itemView);
            tvName = itemView.findViewById(R.id.tvName);
        }
    }
}
```

```kotlin
// Kotlin Adapter with ViewBinding — ~30 lines!
class UserAdapter(
    private val users: List<User>,
    private val onUserClick: (User) -> Unit
) : RecyclerView.Adapter<UserAdapter.UserViewHolder>() {

    inner class UserViewHolder(private val binding: ItemUserBinding)
        : RecyclerView.ViewHolder(binding.root) {
        fun bind(user: User) {
            binding.tvName.text = user.name
            binding.tvEmail.text = user.email
            binding.root.setOnClickListener { onUserClick(user) }
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int) =
        UserViewHolder(ItemUserBinding.inflate(LayoutInflater.from(parent.context), parent, false))

    override fun onBindViewHolder(holder: UserViewHolder, position: Int) =
        holder.bind(users[position])

    override fun getItemCount() = users.size
}
```

---

### 2. `ListAdapter` + `DiffUtil` — The Modern Standard ✅

`ListAdapter` is smarter than basic `RecyclerView.Adapter`. It uses `DiffUtil` to figure out **what changed** and only animates those items:

```kotlin
// Define how to compare two items
class UserDiffCallback : DiffUtil.ItemCallback<User>() {
    // Are these the same user? (compare unique ID)
    override fun areItemsTheSame(oldItem: User, newItem: User) =
        oldItem.id == newItem.id

    // Do they have the same content? (compare all fields)
    override fun areContentsTheSame(oldItem: User, newItem: User) =
        oldItem == newItem  // data class equals() compares all fields automatically!
}

// ListAdapter — just pass DiffCallback, no need to manage list yourself
class UserListAdapter(
    private val onUserClick: (User) -> Unit
) : ListAdapter<User, UserListAdapter.UserViewHolder>(UserDiffCallback()) {

    inner class UserViewHolder(private val binding: ItemUserBinding)
        : RecyclerView.ViewHolder(binding.root) {
        fun bind(user: User) {
            binding.tvName.text = user.name
            binding.tvEmail.text = user.email
            binding.ivAvatar.load(user.avatarUrl) // Coil image loading
            binding.root.setOnClickListener { onUserClick(user) }
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int) =
        UserViewHolder(ItemUserBinding.inflate(LayoutInflater.from(parent.context), parent, false))

    override fun onBindViewHolder(holder: UserViewHolder, position: Int) =
        holder.bind(getItem(position)) // getItem() instead of list[position]
}

// In Fragment/Activity:
val adapter = UserListAdapter { user -> navigateToProfile(user.id) }
binding.recyclerView.adapter = adapter

// Update the list — DiffUtil handles the diff and animations automatically!
adapter.submitList(newUserList)
```

---

### 3. Setting Up RecyclerView in XML + Kotlin

```xml
<!-- In layout XML -->
<androidx.recyclerview.widget.RecyclerView
    android:id="@+id/recyclerView"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:clipToPadding="false"
    android:paddingBottom="16dp"
    tools:listitem="@layout/item_user" /> <!-- Preview in Android Studio -->
```

```kotlin
// In Fragment
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    super.onViewCreated(view, savedInstanceState)

    val adapter = UserListAdapter { user ->
        // Navigate to user detail (Step 19 — Navigation Component)
        findNavController().navigate(
            HomeFragmentDirections.actionHomeToProfile(user.id)
        )
    }

    binding.recyclerView.apply {
        this.adapter = adapter
        layoutManager = LinearLayoutManager(requireContext())
        // Add spacing between items
        addItemDecoration(DividerItemDecoration(requireContext(), DividerItemDecoration.VERTICAL))
    }

    // Observe data from ViewModel (Step 12-13)
    viewModel.users.observe(viewLifecycleOwner) { users ->
        adapter.submitList(users)
    }
}
```

---

### 4. Grid Layout

```kotlin
// 2-column grid
binding.recyclerView.layoutManager = GridLayoutManager(requireContext(), 2)

// Staggered grid (Pinterest style — items of different heights)
binding.recyclerView.layoutManager =
    StaggeredGridLayoutManager(2, StaggeredGridLayoutManager.VERTICAL)
```

---

## 🔑 Key Concepts

| Concept | Role | Key Note |
|---------|------|---------|
| `Adapter` | Bridge between data and views | Creates and binds ViewHolders |
| `ViewHolder` | Holds references to item views | One per visible item |
| `onCreateViewHolder()` | Inflate item layout | Called rarely (only for new slots) |
| `onBindViewHolder()` | Fill item with data | Called for each visible item |
| `LayoutManager` | Controls arrangement & scroll | Always required |
| `ListAdapter` | Smart adapter with DiffUtil | Preferred over basic Adapter |
| `DiffUtil` | Finds what changed in list | Enables smooth animations |
| `submitList()` | Update list data | ListAdapter handles diff automatically |
| `getItem(position)` | Get item in ListAdapter | Use instead of `list[position]` |

---

## 💡 Good Example — Product Grid with Filter

```kotlin
data class Product(val id: Int, val name: String, val price: Double, val imageUrl: String)

class ProductDiffCallback : DiffUtil.ItemCallback<Product>() {
    override fun areItemsTheSame(old: Product, new: Product) = old.id == new.id
    override fun areContentsTheSame(old: Product, new: Product) = old == new
}

class ProductGridAdapter(
    private val onAddToCart: (Product) -> Unit
) : ListAdapter<Product, ProductGridAdapter.ProductViewHolder>(ProductDiffCallback()) {

    inner class ProductViewHolder(private val binding: ItemProductBinding)
        : RecyclerView.ViewHolder(binding.root) {
        fun bind(product: Product) {
            binding.tvProductName.text = product.name
            binding.tvPrice.text = "$${"%.2f".format(product.price)}"
            Glide.with(binding.root).load(product.imageUrl).into(binding.ivProduct)
            binding.btnAddToCart.setOnClickListener { onAddToCart(product) }
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int) =
        ProductViewHolder(ItemProductBinding.inflate(LayoutInflater.from(parent.context), parent, false))

    override fun onBindViewHolder(holder: ProductViewHolder, position: Int) =
        holder.bind(getItem(position))
}

// In Fragment
class ShopFragment : Fragment() {
    private var _binding: FragmentShopBinding? = null
    private val binding get() = _binding!!

    private val adapter = ProductGridAdapter { product ->
        Toast.makeText(requireContext(), "${product.name} added to cart!", Toast.LENGTH_SHORT).show()
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        binding.recyclerView.apply {
            this.adapter = this@ShopFragment.adapter
            layoutManager = GridLayoutManager(requireContext(), 2)
        }

        // Simulate data — replace with ViewModel in real app
        val products = (1..20).map {
            Product(it, "Product $it", it * 9.99, "https://picsum.photos/200/200?random=$it")
        }
        adapter.submitList(products)
    }

    override fun onDestroyView() { super.onDestroyView(); _binding = null }
}
```
