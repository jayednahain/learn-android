# Step 6 — ViewBinding & DataBinding

---

## 📖 What Is It? (Simple Definition)

Imagine you have a remote control (your Kotlin code) and a TV (your XML layout). **ViewBinding** gives you a smart remote where every button is already labeled and typed — no guessing which button does what.

**DataBinding** goes further: it connects your data (like a user name) directly to the TV screen, so when the data changes, the screen automatically updates — without you pressing any button.

---

## 🎯 Where Do We Use It?

- **ViewBinding** → Every Activity, Fragment, Adapter (replacing `findViewById`)
- **DataBinding** → When you want live data to automatically update UI (used with ViewModel)

---

## 🔄 Workflow Diagram

```
ViewBinding:
  activity_main.xml → generates ActivityMainBinding
  binding.myButton.setOnClickListener { ... }   // type safe!

DataBinding:
  ViewModel holds data (LiveData)
        ↓
  XML layout observes it via @{viewModel.userName}
        ↓
  TextView updates automatically when data changes
  (no code needed in Activity/Fragment!)
```

---

## ☕ Java vs Kotlin — What Changed?

### Before (Java + `findViewById`) — The Dark Ages 😅

```java
// Java Activity — messy and error-prone
public class MainActivity extends AppCompatActivity {
    private TextView tvName;
    private Button btnSave;
    private EditText etEmail;
    private ImageView ivProfile;
    // ... 10 more views

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Every. Single. View. with ugly cast
        tvName    = (TextView)   findViewById(R.id.tvName);
        btnSave   = (Button)     findViewById(R.id.btnSave);
        etEmail   = (EditText)   findViewById(R.id.etEmail);
        ivProfile = (ImageView)  findViewById(R.id.ivProfile);

        btnSave.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                tvName.setText("Saved!");
            }
        });
    }
}
```

### Now — ViewBinding ✅

```kotlin
// Enable in build.gradle
android {
    buildFeatures { viewBinding = true }
}
```

```kotlin
// Activity — clean and safe!
class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding  // auto-generated class

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        // Direct, type-safe access — no casting, no null crashes!
        binding.tvName.text = "Jayed"
        binding.btnSave.setOnClickListener {
            binding.tvName.text = "Saved!"
        }
    }
}
```

### ViewBinding in Fragments (slightly different pattern)

```kotlin
class ProfileFragment : Fragment() {
    // Need to nullify binding in onDestroyView to avoid memory leaks!
    private var _binding: FragmentProfileBinding? = null
    private val binding get() = _binding!!  // safe non-null accessor

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View {
        _binding = FragmentProfileBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding.userName.text = "Jayed"
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null  // IMPORTANT: prevent memory leak!
    }
}
```

---

### DataBinding — The Next Level

```kotlin
// Enable in build.gradle
android {
    buildFeatures {
        viewBinding = true
        dataBinding = true  // add this
    }
}
```

**XML with DataBinding — wrap with `<layout>` tag:**
```xml
<!-- activity_main.xml -->
<layout xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- Declare variables used in this layout -->
    <data>
        <variable
            name="viewModel"
            type="com.example.myapp.MainViewModel" />
    </data>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <!-- @{} binds directly to ViewModel data! -->
        <TextView
            android:id="@+id/tvName"
            android:text="@{viewModel.userName}"  />

        <!-- Two-way binding: @={} — EditText changes update ViewModel too -->
        <EditText
            android:text="@={viewModel.inputEmail}" />

        <!-- Bind a click listener -->
        <Button
            android:onClick="@{() -> viewModel.onSaveClicked()}" />

    </LinearLayout>
</layout>
```

**Kotlin code with DataBinding:**
```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding
    private val viewModel: MainViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = DataBindingUtil.setContentView(this, R.layout.activity_main)

        // Connect ViewModel to the layout
        binding.viewModel = viewModel
        binding.lifecycleOwner = this  // needed for LiveData to auto-update UI

        // That's IT! The UI now auto-updates when ViewModel data changes.
    }
}
```

---

## 🔑 Key Concepts

| Feature | ViewBinding | DataBinding |
|---------|------------|-------------|
| Purpose | Safe view access | Auto-connect data ↔ UI |
| Complexity | Simple | Medium |
| XML change | None needed | Wrap with `<layout>` tag |
| Auto UI update | No (manual) | Yes (with LiveData) |
| Build time | Faster | Slightly slower |
| When to use | Always (default choice) | When using MVVM + LiveData |

**ViewBinding naming rule:**
```
activity_main.xml       → ActivityMainBinding
fragment_profile.xml    → FragmentProfileBinding
item_product_card.xml   → ItemProductCardBinding
```
> 💡 Rule: Convert snake_case to PascalCase, add "Binding" at end.

---

## 💡 Good Example — RecyclerView Adapter with ViewBinding

```kotlin
// item_user_card.xml has: tvName, tvEmail, ivAvatar, btnFollow

class UserAdapter(
    private val users: List<User>,
    private val onFollowClick: (User) -> Unit
) : RecyclerView.Adapter<UserAdapter.UserViewHolder>() {

    inner class UserViewHolder(
        private val binding: ItemUserCardBinding  // ViewBinding for item layout
    ) : RecyclerView.ViewHolder(binding.root) {

        fun bind(user: User) {
            binding.tvName.text = user.name
            binding.tvEmail.text = user.email
            // Glide to load image
            Glide.with(binding.root).load(user.avatarUrl).into(binding.ivAvatar)
            binding.btnFollow.setOnClickListener { onFollowClick(user) }
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): UserViewHolder {
        val binding = ItemUserCardBinding.inflate(
            LayoutInflater.from(parent.context), parent, false
        )
        return UserViewHolder(binding)
    }

    override fun onBindViewHolder(holder: UserViewHolder, position: Int) {
        holder.bind(users[position])
    }

    override fun getItemCount() = users.size
}
```
