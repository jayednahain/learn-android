# Step 19 — Jetpack Navigation Component

---

## 📖 What Is It? (Simple Definition)

Imagine your app's screens as cities on a map. In the old days, you had to manually draw roads between every city (manage FragmentTransactions, back stack, intents manually). With **Jetpack Navigation Component**, you draw the map once (the navigation graph), and then just say "go to this city" — Navigation handles the roads!

No more: `supportFragmentManager.beginTransaction().replace(...).commit()` everywhere.

---

## 🎯 Where Do We Use It?

- Any multi-screen app (virtually every real app)
- Replaces manual FragmentManager transactions
- Integrates with BottomNavigationView
- Handles deep links automatically
- Passes data between screens safely with type checking

---

## 🔄 Navigation Architecture Diagram

```
nav_graph.xml (the MAP)
┌─────────────────────────────────────────┐
│  [HomeFragment] ──action──► [DetailFragment] │
│       │                          │       │
│       └──action──► [ProfileFragment]     │
│  [LoginFragment] ──action──► [HomeFragment] │
└─────────────────────────────────────────┘
         ↕
   NavController (handles "go to destination")
         ↕
   NavHostFragment (the container in activity_main.xml)
```

---

## ☕ Java vs Kotlin — What Changed?

### Old Way — Manual Fragment Transactions (messy!)

```java
// Java — manual fragment navigation, scattered all over the codebase
// HomeActivity.java:
getSupportFragmentManager()
    .beginTransaction()
    .replace(R.id.container, new DetailFragment())
    .addToBackStack("detail")
    .commit();

// Pass data the old unreliable way:
Bundle bundle = new Bundle();
bundle.putInt("product_id", 42);
fragment.setArguments(bundle);
// Type safety? None. Easy to make typos in key names!
```

### New Way — Navigation Component ✅

```kotlin
// 1. Setup in build.gradle:
// implementation("androidx.navigation:navigation-fragment-ktx:2.7.5")
// implementation("androidx.navigation:navigation-ui-ktx:2.7.5")
// Safe Args plugin in build.gradle.kts (project level):
// id("androidx.navigation.safeargs.kotlin") version "2.7.5" apply false
```

```xml
<!-- 2. activity_main.xml — NavHostFragment is the screen container -->
<androidx.fragment.app.FragmentContainerView
    android:id="@+id/navHostFragment"
    android:name="androidx.navigation.fragment.NavHostFragment"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    app:navGraph="@navigation/nav_graph"
    app:defaultNavHost="true" />  <!-- handles system Back button -->
```

```xml
<!-- 3. res/navigation/nav_graph.xml — the MAP -->
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/nav_graph"
    app:startDestination="@id/homeFragment">  <!-- starting screen -->

    <fragment
        android:id="@+id/homeFragment"
        android:name="com.example.ui.HomeFragment"
        android:label="Home">

        <!-- Action: navigate from Home to Detail -->
        <action
            android:id="@+id/actionHomeToDetail"
            app:destination="@id/detailFragment"
            app:enterAnim="@anim/slide_in_right"
            app:exitAnim="@anim/slide_out_left" />
    </fragment>

    <fragment
        android:id="@+id/detailFragment"
        android:name="com.example.ui.DetailFragment"
        android:label="Detail">

        <!-- Arguments for this screen — type-safe! -->
        <argument
            android:name="productId"
            app:argType="integer" />
        <argument
            android:name="productName"
            app:argType="string"
            android:defaultValue="Unknown" />
    </fragment>

    <fragment
        android:id="@+id/profileFragment"
        android:name="com.example.ui.ProfileFragment" />

    <!-- Deep link: opening app from URL https://myapp.com/product/42 -->
    <deepLink app:uri="https://myapp.com/product/{productId}" />

</navigation>
```

```kotlin
// 4. Navigate from HomeFragment with Safe Args (type-safe!)
class HomeFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        binding.productCard.setOnClickListener {
            // Safe Args generates HomeFragmentDirections class automatically
            val action = HomeFragmentDirections.actionHomeToDetail(
                productId = 42,
                productName = "Kotlin Book"
            )
            findNavController().navigate(action)
        }
    }
}

// 5. Receive arguments in DetailFragment (type-safe!)
class DetailFragment : Fragment() {
    // Safe Args generates this delegate — no Bundle.getInt("key") needed!
    private val args: DetailFragmentArgs by navArgs()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        binding.tvTitle.text = args.productName    // String, guaranteed non-null
        loadProduct(args.productId)                // Int, guaranteed type
    }
}
```

---

### Navigation with BottomNavigationView

```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        // Connect NavController to BottomNavigationView — ONE LINE!
        val navController = findNavController(R.id.navHostFragment)
        binding.bottomNavigation.setupWithNavController(navController)

        // Connect to ActionBar/Toolbar if needed
        setupActionBarWithNavController(navController)
    }

    // Handle toolbar back button
    override fun onSupportNavigateUp(): Boolean {
        return findNavController(R.id.navHostFragment).navigateUp() || super.onSupportNavigateUp()
    }
}
// That's it! Back stack, selected tab state, everything handled automatically!
```

---

### Popping Back Stack

```kotlin
// Go back (same as pressing Back button)
findNavController().navigateUp()
findNavController().popBackStack()

// Go back to a specific destination
findNavController().popBackStack(R.id.homeFragment, inclusive = false)

// Go back and pass result to previous screen
// (In DetailFragment — set result before navigating back)
val navController = findNavController()
navController.previousBackStackEntry?.savedStateHandle?.set("result", "selected item")
navController.popBackStack()

// In HomeFragment — observe the result
navController.currentBackStackEntry?.savedStateHandle?.getLiveData<String>("result")
    ?.observe(viewLifecycleOwner) { result ->
        binding.tvResult.text = result
    }
```

---

### Nested Navigation Graphs

```xml
<!-- Each tab in BottomNavigationView can have its own navigation graph -->
<navigation android:id="@+id/nav_home_graph" app:startDestination="@id/homeFragment">
    <fragment android:id="@+id/homeFragment" ... />
    <fragment android:id="@+id/productDetailFragment" ... />
</navigation>
```

---

## 🔑 Key Concepts

| Concept | What It Does | Java Equivalent |
|---------|-------------|----------------|
| `NavHostFragment` | Container for all Fragment screens | `FrameLayout` + FragmentTransaction |
| `NavController` | Controls navigation (go to, back) | `FragmentManager` |
| `nav_graph.xml` | Map of all screens and connections | Scattered code everywhere |
| `Safe Args` | Type-safe argument passing | Raw `Bundle.putInt("key", value)` |
| `findNavController()` | Get NavController in Fragment | `getSupportFragmentManager()` |
| `navigate(action)` | Go to next screen | `beginTransaction().replace()` |
| `navigateUp()` | Go back | `popBackStack()` |
| Deep Link | Open screen from URL | Manual intent-filter parsing |

---

## 💡 Good Example — E-Commerce App Navigation

```xml
<!-- nav_graph.xml for a shopping app -->
<navigation app:startDestination="@id/homeFragment">

    <fragment android:id="@+id/homeFragment" android:name="...HomeFragment">
        <action android:id="@+id/toSearch" app:destination="@id/searchFragment" />
        <action android:id="@+id/toProduct" app:destination="@id/productFragment" />
    </fragment>

    <fragment android:id="@+id/searchFragment" android:name="...SearchFragment">
        <argument android:name="initialQuery" app:argType="string" android:defaultValue="" />
        <action android:id="@+id/toProduct" app:destination="@id/productFragment" />
    </fragment>

    <fragment android:id="@+id/productFragment" android:name="...ProductFragment">
        <argument android:name="productId" app:argType="integer" />
        <action android:id="@+id/toCart" app:destination="@id/cartFragment"
            app:popUpTo="@id/homeFragment" /> <!-- clear product stack when going to cart -->
    </fragment>

    <fragment android:id="@+id/cartFragment" android:name="...CartFragment">
        <action android:id="@+id/toCheckout" app:destination="@id/checkoutFragment" />
    </fragment>

    <fragment android:id="@+id/checkoutFragment" android:name="...CheckoutFragment">
        <!-- After checkout, pop everything back to home -->
        <action android:id="@+id/toOrderSuccess" app:destination="@id/homeFragment"
            app:popUpTo="@id/homeFragment"
            app:launchSingleTop="true" />
    </fragment>
</navigation>
```

```kotlin
// In HomeFragment
binding.searchBar.setOnClickListener {
    findNavController().navigate(
        HomeFragmentDirections.toSearch(initialQuery = "")
    )
}
binding.rvProducts.setOnItemClickListener { productId ->
    findNavController().navigate(
        HomeFragmentDirections.toProduct(productId = productId)
    )
}
```
