# Step 5 — Activity & Lifecycle (Refresher)

---

## 📖 What Is It? (Simple Definition)

An **Activity** is one screen in your app. Think of it like a TV channel — each channel is one view. When you change channels (navigate between screens), you're switching Activities (or Fragments).

The **lifecycle** is the story of a screen's life — it's born (`onCreate`), starts playing (`onStart`), gets your full attention (`onResume`), is pushed to background (`onPause`), leaves the screen (`onStop`), and finally dies (`onDestroy`).

---

## 🎯 Where Do We Use It?

Every single screen in your app is an Activity (or hosted in one). The lifecycle tells you **when** to:
- Load data (→ `onCreate`)
- Start animations or sensors (→ `onResume`)
- Save data before leaving (→ `onPause`)
- Release heavy resources (→ `onStop` / `onDestroy`)

---

## 🔄 Lifecycle Diagram

```
App launches screen
       ↓
   onCreate()    ← Set up UI, ViewBinding, ViewModel
       ↓
   onStart()     ← Screen is visible (but not interactive)
       ↓
   onResume()    ← User is actively using the screen
       ↓         ← ← ← ← ← ← ← ← ←
       |  [Another app / dialog comes]  |
       ↓                               |
   onPause()     ← Screen partially hidden
       ↓                               |
   onStop()      ← Screen fully hidden ↑ (back to screen)
       ↓
   onDestroy()   ← Screen removed from memory
```

```
Screen Rotated / Memory pressure:
  onStop → onDestroy → onCreate (fresh start!)
  ← ViewModel saves data across this! →
```

---

## ☕ Java vs Kotlin — What Changed?

### 1. Activity Declaration
```java
// Java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        TextView tv = findViewById(R.id.myTextView); // old way
        tv.setText("Hello");
    }
}
```
```kotlin
// Kotlin with ViewBinding (modern way)
class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        binding.myTextView.text = "Hello"  // direct access, no casting!
    }
}
```
> 💡 No more `(TextView) findViewById(R.id.myTextView)` with the ugly casting! ViewBinding gives you **type-safe** access to every view.

---

### 2. All Lifecycle Methods in Kotlin
```kotlin
class MainActivity : AppCompatActivity() {

    // Called ONCE when activity is created. Do: setup UI, ViewBinding, ViewModel, click listeners
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)
        Log.d("Lifecycle", "onCreate")
    }

    // Called every time screen becomes visible
    override fun onStart() {
        super.onStart()
        Log.d("Lifecycle", "onStart")
    }

    // Called when user can interact with screen
    override fun onResume() {
        super.onResume()
        Log.d("Lifecycle", "onResume — user is here!")
        // Good place: start camera, start sensor, resume video
    }

    // Called when screen loses focus (another app/dialog on top)
    override fun onPause() {
        super.onPause()
        Log.d("Lifecycle", "onPause — save lightweight data here")
        // Good place: pause video, stop sensor, save draft text
    }

    // Called when screen is no longer visible
    override fun onStop() {
        super.onStop()
        Log.d("Lifecycle", "onStop — release heavy resources here")
        // Good place: cancel network requests, unregister listeners
    }

    // Called when activity is destroyed (back pressed, screen rotated, low memory)
    override fun onDestroy() {
        super.onDestroy()
        Log.d("Lifecycle", "onDestroy — final cleanup")
    }

    // Called when recreated after rotation (if you used onSaveInstanceState)
    override fun onRestoreInstanceState(savedInstanceState: Bundle) {
        super.onRestoreInstanceState(savedInstanceState)
        val myText = savedInstanceState.getString("MY_TEXT") ?: ""
        binding.myTextView.text = myText
    }

    // Called before destroy if system needs to kill app — save critical state
    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        outState.putString("MY_TEXT", binding.myTextView.text.toString())
    }
}
```

---

### 3. `ViewBinding` — The Modern `findViewById` Replacement

**Enable in `build.gradle`:**
```kotlin
android {
    buildFeatures {
        viewBinding = true
    }
}
```

**How it works:**
```
activity_main.xml exists
         ↓
Android generates: ActivityMainBinding.kt (automatically)
         ↓
You use: binding.anyViewId  (type-safe, no casting!)
```

```kotlin
// Old Java way — 100 lines of findViewByIds
TextView tv = (TextView) findViewById(R.id.myText);   // ugly cast
Button btn = (Button) findViewById(R.id.myButton);
ImageView img = (ImageView) findViewById(R.id.myImage);
// × 20 views = 20 lines of this mess

// New Kotlin + ViewBinding way
binding.myText.text = "Hello"    // auto-typed as TextView
binding.myButton.text = "Click"  // auto-typed as Button
binding.myImage.setImageResource(R.drawable.logo) // auto-typed as ImageView
```

---

## 🔑 Key Concepts

| Lifecycle Method | When Called | What to Do Here |
|-----------------|------------|-----------------|
| `onCreate()` | First creation | Setup UI, binding, ViewModel, click listeners |
| `onStart()` | Becoming visible | Start low-priority tasks |
| `onResume()` | User interaction begins | Resume video, sensors, animations |
| `onPause()` | Losing focus | Pause video, save lightweight state |
| `onStop()` | Fully hidden | Cancel requests, release heavy resources |
| `onDestroy()` | Being removed | Final cleanup (rare need) |
| `onSaveInstanceState()` | Before potential destroy | Save critical UI state |

> 🚨 **Important:** After Step 12 (ViewModel), you'll learn a better way to save data across rotation — using ViewModel instead of `onSaveInstanceState`.

---

## 💡 Good Example — Complete Activity with All Best Practices

```kotlin
class ProductDetailActivity : AppCompatActivity() {

    private lateinit var binding: ActivityProductDetailBinding
    // ViewModel will be added in Step 12 — for now, just plain data
    private var productId: Int = -1

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // ViewBinding setup
        binding = ActivityProductDetailBinding.inflate(layoutInflater)
        setContentView(binding.root)

        // Get data passed from previous screen
        productId = intent.getIntExtra("PRODUCT_ID", -1)

        if (productId == -1) {
            Toast.makeText(this, "Invalid product!", Toast.LENGTH_SHORT).show()
            finish() // close the screen
            return
        }

        // Setup toolbar
        setSupportActionBar(binding.toolbar)
        supportActionBar?.setDisplayHomeAsUpEnabled(true) // back arrow

        // Click listeners
        binding.buyButton.setOnClickListener {
            Toast.makeText(this, "Buying product $productId!", Toast.LENGTH_SHORT).show()
        }

        binding.shareButton.setOnClickListener {
            val shareIntent = Intent(Intent.ACTION_SEND).apply {
                type = "text/plain"
                putExtra(Intent.EXTRA_TEXT, "Check out product $productId!")
            }
            startActivity(Intent.createChooser(shareIntent, "Share via"))
        }

        loadProduct(productId)
    }

    private fun loadProduct(id: Int) {
        // Simulate loading — in real app, use ViewModel + Repository
        binding.productTitle.text = "Product #$id"
        binding.productPrice.text = "$99.99"
    }

    override fun onResume() {
        super.onResume()
        // Refresh stock availability when returning to this screen
        binding.stockStatus.text = "In Stock"
    }

    override fun onPause() {
        super.onPause()
        // Save the current scroll position
        // (will use ViewModel for this properly in Step 12)
    }

    // Handle toolbar back button
    override fun onSupportNavigateUp(): Boolean {
        onBackPressedDispatcher.onBackPressed()
        return true
    }
}
```
