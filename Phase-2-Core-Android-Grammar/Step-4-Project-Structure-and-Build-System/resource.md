# Step 4 — Project Structure & Build System

---

## 📖 What Is It? (Simple Definition)

When you open an Android Studio project, you see lots of folders and files. Think of it like a school building:
- The **main office** (AndroidManifest.xml) keeps all the records — who works here, what rooms exist
- The **classrooms** (`res/layout/`) are your screen designs
- The **supply room** (`res/values/`) holds shared text, colors, and sizes
- The **blueprint files** (`build.gradle`) say what materials are needed to build the school

Understanding where everything lives saves you hours of confusion.

---

## 🎯 Where Do We Use It?

Every time you:
- Add a new Activity or Service (→ register in `AndroidManifest.xml`)
- Add a new library (→ edit `build.gradle`)
- Create a new screen layout (→ `res/layout/`)
- Add app colors/strings (→ `res/values/`)

---

## 🔄 Project Structure Diagram

```
MyApp/
├── app/
│   ├── src/
│   │   └── main/
│   │       ├── AndroidManifest.xml      ← App's ID card
│   │       ├── java/com/example/myapp/  ← Kotlin/Java source code
│   │       │   ├── MainActivity.kt
│   │       │   └── ...
│   │       └── res/
│   │           ├── layout/              ← XML screen designs
│   │           │   └── activity_main.xml
│   │           ├── values/
│   │           │   ├── strings.xml      ← text labels
│   │           │   ├── colors.xml       ← app colors
│   │           │   └── themes.xml       ← app theme/style
│   │           ├── drawable/            ← images, icons, shapes
│   │           └── mipmap/              ← launcher icons
│   └── build.gradle (app level)         ← app dependencies + settings
├── build.gradle (project level)         ← project-wide settings
└── gradle.properties                    ← JVM / Gradle flags
```

---

## ☕ Java vs Kotlin — What Changed?

### 1. `build.gradle` — Moving to Kotlin DSL
```groovy
// Old Java era — build.gradle (Groovy DSL)
android {
    compileSdkVersion 33
    defaultConfig {
        applicationId "com.example.myapp"
        minSdkVersion 21
        targetSdkVersion 33
        versionCode 1
        versionName "1.0"
    }
}
dependencies {
    implementation 'androidx.appcompat:appcompat:1.6.1'
}
```
```kotlin
// Modern Kotlin DSL — build.gradle.kts (same config, Kotlin style)
android {
    compileSdk = 34
    defaultConfig {
        applicationId = "com.example.myapp"
        minSdk = 21
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"
    }
}
dependencies {
    implementation("androidx.appcompat:appcompat:1.6.1")
    implementation("androidx.core:core-ktx:1.12.0") // Kotlin extensions
}
```
> 💡 New projects use `.kts` files (Kotlin Script). Old projects use `.gradle` (Groovy). Both work the same way.

---

### 2. `AndroidManifest.xml` — Still the Same but More Important Now
```xml
<!-- AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- Permissions you need -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.CAMERA" />

    <application
        android:name=".MyApp"            <!-- Your Application class (for Hilt) -->
        android:label="@string/app_name" <!-- App name from strings.xml -->
        android:icon="@mipmap/ic_launcher"
        android:theme="@style/Theme.MyApp">

        <!-- Every Activity MUST be registered here -->
        <activity android:name=".MainActivity"
            android:exported="true">
            <!-- This marks it as the LAUNCH screen -->
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <activity android:name=".ProfileActivity" />
        <!-- No intent-filter = NOT a launcher screen -->

    </application>
</manifest>
```

---

### 3. `R` Class — How Resources Are Referenced
The `R` class is **automatically generated** by Android. You never write it — Android Studio creates it for you based on your `res/` folder.

```kotlin
// In your Kotlin code, refer to resources with R.
setContentView(R.layout.activity_main)   // the layout XML file
binding.image.setImageResource(R.drawable.logo)
val color = ContextCompat.getColor(this, R.color.primary_blue)
val text = getString(R.string.welcome_message)
```

```xml
<!-- res/values/strings.xml -->
<resources>
    <string name="app_name">My App</string>
    <string name="welcome_message">Welcome back!</string>
</resources>

<!-- res/values/colors.xml -->
<resources>
    <color name="primary_blue">#2196F3</color>
    <color name="background">#FFFFFF</color>
</resources>
```

---

## 🔑 Key Concepts

| File / Folder | Purpose | Must Know |
|--------------|---------|-----------|
| `AndroidManifest.xml` | App's ID card — declares all components | Every Activity/Service MUST be here |
| `build.gradle (app)` | App's shopping list: libraries, min SDK | Add dependencies here |
| `build.gradle (project)` | Workspace-wide settings | Rarely edited |
| `gradle.properties` | JVM/Gradle performance flags | Set `JvmArgs` here |
| `res/layout/` | XML files for screen designs | One xml per screen |
| `res/values/strings.xml` | All text labels | Never hardcode strings in code! |
| `res/values/colors.xml` | All app colors | Use color names, not hex codes in code |
| `res/drawable/` | Images, vector icons, shape XMLs | Use vector drawables for scalability |
| `res/mipmap/` | App launcher icons only | Different sizes for different screens |
| `R` class | Auto-generated resource IDs | Never edit manually |

---

## 💡 Good Example — Adding a New Library and Using It

**Scenario:** Add the Glide image loading library.

**Step 1 — Add dependency in `build.gradle (app)`:**
```kotlin
dependencies {
    implementation("com.github.bumptech.glide:glide:4.16.0")
}
```

**Step 2 — Add internet permission in `AndroidManifest.xml`:**
```xml
<uses-permission android:name="android.permission.INTERNET" />
```

**Step 3 — Add an image resource string in `res/values/strings.xml`:**
```xml
<string name="profile_pic_url">https://example.com/photo.jpg</string>
```

**Step 4 — Use in Kotlin code:**
```kotlin
val url = getString(R.string.profile_pic_url)
Glide.with(this)
    .load(url)
    .into(binding.profileImageView)
```

> 💡 **Important tip:** After adding a new dependency in `build.gradle`, always do **Sync Project with Gradle Files** (the elephant icon in Android Studio toolbar).
