# Step 8 — Fragments & Fragment Lifecycle

---

## 📖 What Is It? (Simple Definition)

Imagine a TV screen split into sections — the top shows a list of emails, the bottom shows the email content. Each section is a **Fragment**. A Fragment is like a "mini-screen" that lives inside an Activity.

Think of an Activity as a house, and Fragments as rooms inside the house. You can swap rooms (navigate between Fragments) without rebuilding the whole house (Activity).

---

## 🎯 Where Do We Use It?

- Bottom navigation tabs (each tab = a Fragment)
- Split-screen / tablet layouts
- Step-by-step wizards (each step = a Fragment)
- Dialogs (`DialogFragment`)
- Reusable UI components shown in multiple places
- Navigation Component (Step 19) works entirely with Fragments

---

## 🔄 Fragment Lifecycle Diagram

```
Activity Created
       ↓
  Fragment.onAttach()       ← attached to Activity
       ↓
  Fragment.onCreate()       ← Fragment created (no UI yet)
       ↓
  Fragment.onCreateView()   ← Inflate XML layout here, return root view
       ↓
  Fragment.onViewCreated()  ← View is ready, set up click listeners, ViewModel here
       ↓
  Fragment.onStart()
       ↓
  Fragment.onResume()       ← User sees + interacts with Fragment
       ↓         (back pressed or navigated away)
  Fragment.onPause()
       ↓
  Fragment.onStop()
       ↓
  Fragment.onDestroyView()  ← Null your binding here! (prevent memory leak)
       ↓
  Fragment.onDestroy()
       ↓
  Fragment.onDetach()       ← separated from Activity
```

---

## ☕ Java vs Kotlin — What Changed?

### 1. Creating a Fragment
```java
// Java — verbose with overrides
public class ProfileFragment extends Fragment {
    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater,
                             @Nullable ViewGroup container,
                             @Nullable Bundle savedInstanceState) {
        return inflater.inflate(R.layout.fragment_profile, container, false);
    }

    @Override
    public void onViewCreated(@NonNull View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);
        TextView tv = view.findViewById(R.id.tvName);
        tv.setText("Hello");
    }
}
```
```kotlin
// Kotlin with ViewBinding — clean!
class ProfileFragment : Fragment(R.layout.fragment_profile) {
    // Pass layout ID to constructor — no need to override onCreateView!
    
    private var _binding: FragmentProfileBinding? = null
    private val binding get() = _binding!!

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View {
        _binding = FragmentProfileBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding.tvName.text = "Hello"
        binding.btnEdit.setOnClickListener { /* handle click */ }
    }

    // MUST null out binding to prevent memory leaks
    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

---

### 2. FragmentManager & FragmentTransaction
```java
// Java — fragmentManager and transaction
getSupportFragmentManager()
    .beginTransaction()
    .replace(R.id.fragmentContainer, new ProfileFragment())
    .addToBackStack(null)
    .commit();
```
```kotlin
// Kotlin — cleaner with ktx
// Add fragment-ktx to build.gradle:
// implementation("androidx.fragment:fragment-ktx:1.6.2")

supportFragmentManager.commit {
    replace(R.id.fragmentContainer, ProfileFragment())
    addToBackStack("profile")  // name the back stack entry
}

// Or navigate back programmatically
supportFragmentManager.popBackStack()
```

---

### 3. Passing Data to Fragment — Arguments (NOT constructor params!)

```kotlin
// WRONG — never pass data via constructor! Fragment loses data on recreation.
// val fragment = ProfileFragment(userId) ← BAD!

// CORRECT — use Bundle arguments
class ProfileFragment : Fragment() {
    companion object {
        private const val ARG_USER_ID = "user_id"
        private const val ARG_NAME = "name"

        // Factory method — safe way to create Fragment with data
        fun newInstance(userId: Int, name: String): ProfileFragment {
            return ProfileFragment().apply {
                arguments = Bundle().apply {
                    putInt(ARG_USER_ID, userId)
                    putString(ARG_NAME, name)
                }
            }
        }
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        // Read arguments safely
        val userId = arguments?.getInt(ARG_USER_ID) ?: -1
        val name = arguments?.getString(ARG_NAME) ?: "Unknown"

        binding.tvName.text = name
    }
}

// In Activity:
val fragment = ProfileFragment.newInstance(42, "Jayed")
supportFragmentManager.commit {
    replace(R.id.container, fragment)
    addToBackStack(null)
}
```

---

### 4. Fragment ↔ Activity Communication

**Old way (casting — fragile):**
```java
// Java — Activity implements interface
((MainActivity) getActivity()).onDataReceived(data);  // can crash!
```

**Modern way using ViewModel (Step 12 covers this fully):**
```kotlin
// Share ViewModel between Activity and Fragment
// In Fragment:
private val sharedViewModel: SharedViewModel by activityViewModels()

// In Activity:
private val sharedViewModel: SharedViewModel by viewModels()

// When fragment updates data, Activity automatically sees it!
```

---

### 5. `BottomSheetDialogFragment` — Popup Sheet from Bottom

```kotlin
class ShareBottomSheet : BottomSheetDialogFragment() {
    private var _binding: BottomSheetShareBinding? = null
    private val binding get() = _binding!!

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View {
        _binding = BottomSheetShareBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        binding.btnShareWhatsApp.setOnClickListener { shareVia("WhatsApp") }
        binding.btnShareEmail.setOnClickListener { shareVia("Email") }
    }

    private fun shareVia(platform: String) {
        dismiss()  // close the bottom sheet
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}

// Show it from Activity or Fragment:
ShareBottomSheet().show(childFragmentManager, "share_bottom_sheet")
```

---

## 🔑 Key Concepts

| Concept | What It Does | Key Note |
|---------|-------------|---------|
| `onCreateView()` | Inflate XML layout | Return `binding.root` |
| `onViewCreated()` | Setup UI, listeners, ViewModel | Views are ready here |
| `onDestroyView()` | View is being destroyed | Set `_binding = null` HERE |
| `arguments` | Pass data to fragment | Use Bundle, not constructor |
| `addToBackStack()` | Allow back navigation | Named stacks help pop to specific point |
| `childFragmentManager` | For fragment-in-fragment | Use instead of `parentFragmentManager` |
| `activityViewModels()` | Share data with Activity | Better than interface callbacks |

---

## 💡 Good Example — Bottom Navigation with 3 Fragments

```kotlin
// MainActivity.kt
class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        // Show default tab
        if (savedInstanceState == null) {
            loadFragment(HomeFragment())
        }

        binding.bottomNavigation.setOnItemSelectedListener { item ->
            when (item.itemId) {
                R.id.nav_home    -> { loadFragment(HomeFragment()); true }
                R.id.nav_search  -> { loadFragment(SearchFragment()); true }
                R.id.nav_profile -> { loadFragment(ProfileFragment.newInstance(1, "Jayed")); true }
                else -> false
            }
        }
    }

    private fun loadFragment(fragment: Fragment) {
        supportFragmentManager.commit {
            replace(R.id.fragmentContainer, fragment)
            // NOT adding to back stack for bottom nav tabs
        }
    }
}
```
