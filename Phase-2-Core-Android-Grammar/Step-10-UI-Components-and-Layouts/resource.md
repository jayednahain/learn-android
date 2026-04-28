# Step 10 — UI Components & Layouts

---

## 📖 What Is It? (Simple Definition)

Layouts are like puzzle boards — they decide **where** views are placed on the screen. UI components are the puzzle pieces themselves — buttons, text fields, images.

Android gives you pre-built, beautiful components from **Material Design** — Google's design system used by apps like Gmail, YouTube, and Google Maps.

---

## 🎯 Where Do We Use It?

Every screen you build in Android uses layouts and UI components. Getting these right means:
- Your app looks professional
- Works on all screen sizes (phones, tablets, foldables)
- Is accessible (large text, screen readers)

---

## 🔄 Layout Decision Diagram

```
Do you have a complex screen with overlapping views?
   YES → ConstraintLayout

Do you have items in a simple row or column?
   YES → LinearLayout

Do you need one view on top of another?
   YES → FrameLayout

Do you need items relative to each other or parent? (old code)
   YES → RelativeLayout (prefer ConstraintLayout instead)
```

---

## ☕ Java vs Kotlin — What Changed?

### 1. ConstraintLayout — The Modern Standard

```
Old XML layouts used nested LinearLayouts:
LinearLayout (vertical)
  └── LinearLayout (horizontal)
        ├── TextView
        └── ImageView (3 levels deep = slow rendering!)

ConstraintLayout:
  All views at same level, connected by constraint lines
  (1 level = much faster rendering!)
```

```xml
<!-- ConstraintLayout — FLAT structure, no nesting needed -->
<androidx.constraintlayout.widget.ConstraintLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <ImageView
        android:id="@+id/ivAvatar"
        android:layout_width="64dp"
        android:layout_height="64dp"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        android:layout_margin="16dp" />

    <TextView
        android:id="@+id/tvName"
        android:layout_width="0dp"   <!-- 0dp = match constraints -->
        android:layout_height="wrap_content"
        android:text="Jayed"
        android:textSize="18sp"
        app:layout_constraintTop_toTopOf="@id/ivAvatar"
        app:layout_constraintStart_toEndOf="@id/ivAvatar"
        app:layout_constraintEnd_toEndOf="parent"
        android:layout_marginStart="12dp" />

    <TextView
        android:id="@+id/tvBio"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:text="Android Developer"
        app:layout_constraintTop_toBottomOf="@id/tvName"
        app:layout_constraintStart_toStartOf="@id/tvName"
        app:layout_constraintEnd_toEndOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

---

### 2. Material Design Components — Old vs New

```xml
<!-- OLD: Plain Android button — looks ugly -->
<Button
    android:text="Submit"
    android:background="#2196F3" />

<!-- NEW: MaterialButton — rounded, ripple effect, beautiful! -->
<com.google.android.material.button.MaterialButton
    android:text="Submit"
    style="@style/Widget.MaterialComponents.Button"
    app:cornerRadius="8dp"
    app:icon="@drawable/ic_check" />

<!-- MaterialButton variants -->
<com.google.android.material.button.MaterialButton
    style="@style/Widget.MaterialComponents.Button.OutlinedButton"  <!-- outlined -->
    android:text="Cancel" />

<com.google.android.material.button.MaterialButton
    style="@style/Widget.MaterialComponents.Button.TextButton"  <!-- text only -->
    android:text="Learn More" />
```

---

### 3. `TextInputLayout` — Beautiful Form Fields

```xml
<!-- OLD: Plain EditText — no floating label, no error message styling -->
<EditText android:hint="Email" />

<!-- NEW: TextInputLayout wraps EditText — floating label, error, counter, icon! -->
<com.google.android.material.textfield.TextInputLayout
    android:id="@+id/emailLayout"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:hint="Email Address"
    app:startIconDrawable="@drawable/ic_email"
    style="@style/Widget.MaterialComponents.TextInputLayout.OutlinedBox">

    <com.google.android.material.textfield.TextInputEditText
        android:id="@+id/etEmail"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:inputType="textEmailAddress" />

</com.google.android.material.textfield.TextInputLayout>
```

```kotlin
// Show/hide error in Kotlin
binding.emailLayout.error = "Invalid email!"   // show error
binding.emailLayout.error = null               // hide error

val email = binding.etEmail.text.toString()
```

---

### 4. `Snackbar` — Replaces Dialog Alerts for Short Messages

```kotlin
// OLD: AlertDialog for simple messages (too heavy)
AlertDialog.Builder(this).setMessage("Saved!").show()

// NEW: Snackbar — appears at bottom, dismisses itself, can have action
Snackbar.make(binding.root, "Profile saved!", Snackbar.LENGTH_SHORT).show()

// Snackbar with action button
Snackbar.make(binding.root, "Item deleted", Snackbar.LENGTH_LONG)
    .setAction("UNDO") { viewModel.undoDelete() }
    .show()
```

---

### 5. `BottomNavigationView` — Tab Navigation

```xml
<!-- In layout -->
<com.google.android.material.bottomnavigation.BottomNavigationView
    android:id="@+id/bottomNavigation"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    app:menu="@menu/bottom_nav_menu" />
```

```xml
<!-- res/menu/bottom_nav_menu.xml -->
<menu xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:id="@+id/nav_home"    android:title="Home"    android:icon="@drawable/ic_home" />
    <item android:id="@+id/nav_search"  android:title="Search"  android:icon="@drawable/ic_search" />
    <item android:id="@+id/nav_profile" android:title="Profile" android:icon="@drawable/ic_person" />
</menu>
```

---

### 6. `TabLayout` + `ViewPager2` — Swipeable Tabs

```xml
<LinearLayout
    android:orientation="vertical">
    <com.google.android.material.tabs.TabLayout
        android:id="@+id/tabLayout"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />
    <androidx.viewpager2.widget.ViewPager2
        android:id="@+id/viewPager"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1" />
</LinearLayout>
```

```kotlin
// FragmentStateAdapter for ViewPager2
class TabAdapter(fragment: Fragment) : FragmentStateAdapter(fragment) {
    private val tabs = listOf("Posts", "Followers", "Following")
    override fun getItemCount() = tabs.size
    override fun createFragment(position: Int) = when (position) {
        0 -> PostsFragment()
        1 -> FollowersFragment()
        else -> FollowingFragment()
    }
}

// Connect TabLayout with ViewPager2
val adapter = TabAdapter(this)
binding.viewPager.adapter = adapter
TabLayoutMediator(binding.tabLayout, binding.viewPager) { tab, position ->
    tab.text = listOf("Posts", "Followers", "Following")[position]
}.attach()
```

---

### 7. `CoordinatorLayout` + `AppBarLayout` — Collapsing Toolbar

```xml
<androidx.coordinatorlayout.widget.CoordinatorLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <com.google.android.material.appbar.AppBarLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content">

        <com.google.android.material.appbar.CollapsingToolbarLayout
            android:layout_width="match_parent"
            android:layout_height="200dp"
            app:layout_scrollFlags="scroll|exitUntilCollapsed"
            app:toolbarId="@id/toolbar">

            <ImageView
                android:src="@drawable/header_image"
                app:layout_collapseMode="parallax" />

            <androidx.appcompat.widget.Toolbar
                android:id="@+id/toolbar"
                app:layout_collapseMode="pin" /> <!-- stays visible when collapsed -->

        </com.google.android.material.appbar.CollapsingToolbarLayout>
    </com.google.android.material.appbar.AppBarLayout>

    <androidx.recyclerview.widget.RecyclerView
        app:layout_behavior="@string/appbar_scrolling_view_behavior" />  <!-- IMPORTANT! -->

</androidx.coordinatorlayout.widget.CoordinatorLayout>
```

---

## 🔑 Key Concepts

| Component | Replaces | Use For |
|-----------|---------|--------|
| `ConstraintLayout` | Nested LinearLayouts | Most screens |
| `MaterialButton` | Plain `Button` | All buttons |
| `TextInputLayout` | Plain `EditText` | All form inputs |
| `Snackbar` | Toast + AlertDialog | Brief feedback |
| `BottomNavigationView` | Custom tab bar | Bottom tabs |
| `ViewPager2` | ViewPager (old) | Swipeable screens/tabs |
| `CollapsingToolbarLayout` | Custom scroll logic | Profile/detail pages |
| `SwipeRefreshLayout` | Custom pull-to-refresh | Lists with refresh |
| `Chip` | RadioButton groups | Tags, filters, categories |

---

## 💡 Good Example — Profile Screen Layout

```xml
<!-- fragment_profile.xml -->
<androidx.coordinatorlayout.widget.CoordinatorLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <com.google.android.material.appbar.AppBarLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content">

        <androidx.appcompat.widget.Toolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            app:title="Profile" />

        <com.google.android.material.tabs.TabLayout
            android:id="@+id/tabLayout"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />

    </com.google.android.material.appbar.AppBarLayout>

    <androidx.viewpager2.widget.ViewPager2
        android:id="@+id/viewPager"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="@string/appbar_scrolling_view_behavior" />

    <!-- Floating Action Button -->
    <com.google.android.material.floatingactionbutton.FloatingActionButton
        android:id="@+id/fabEdit"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="bottom|end"
        android:layout_margin="16dp"
        android:src="@drawable/ic_edit"
        app:tint="@android:color/white" />

</androidx.coordinatorlayout.widget.CoordinatorLayout>
```
