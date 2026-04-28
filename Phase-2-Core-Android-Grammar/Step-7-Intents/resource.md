# Step 7 — Intents (Refresher + Updates)

---

## 📖 What Is It? (Simple Definition)

An **Intent** is like a message you send inside (or outside) your app. It says: *"Hey, go open this screen!"* or *"Hey, take this photo for me!"* or *"Hey, share this text through WhatsApp!"*

- **Explicit Intent** = You know exactly who to send the message to (a specific screen in your app)
- **Implicit Intent** = You say what you need done, and Android decides who can help (share, camera, browser)

---

## 🎯 Where Do We Use It?

- Navigate from one screen to another
- Pass data between screens
- Open camera, gallery, maps, browser, dialer
- Share content via WhatsApp, Gmail, etc.
- Handle incoming deep links (open app from a URL)

---

## 🔄 Intent Flow Diagram

```
Explicit Intent:
  MainActivity ──────────────────→ ProfileActivity
   (I know who I want)

Implicit Intent:
  Your App ──→ Android System ──→ Resolves → Camera App
              (Who can take a photo?)         or
                                              Gallery App
                                              or lets user choose

ActivityResultLauncher (modern):
  Your Activity ──→ Another Activity/System App
                 ←── Returns result back safely
```

---

## ☕ Java vs Kotlin — What Changed?

### 1. Explicit Intent — Start Another Screen
```java
// Java
Intent intent = new Intent(this, ProfileActivity.class);
intent.putExtra("USER_ID", 42);
intent.putExtra("USER_NAME", "Jayed");
startActivity(intent);
```
```kotlin
// Kotlin — shorter with apply
val intent = Intent(this, ProfileActivity::class.java).apply {
    putExtra("USER_ID", 42)
    putExtra("USER_NAME", "Jayed")
}
startActivity(intent)
```

**Receiving data in ProfileActivity:**
```kotlin
// Java
String name = getIntent().getStringExtra("USER_NAME");

// Kotlin — safe null handling
val userId = intent.getIntExtra("USER_ID", -1)
val userName = intent.getStringExtra("USER_NAME") ?: "Unknown"
```

---

### 2. Implicit Intent — Open System Features
```kotlin
// Open a URL in browser
val browserIntent = Intent(Intent.ACTION_VIEW, Uri.parse("https://google.com"))
startActivity(browserIntent)

// Open phone dialer
val dialIntent = Intent(Intent.ACTION_DIAL, Uri.parse("tel:+8801234567890"))
startActivity(dialIntent)

// Share text
val shareIntent = Intent(Intent.ACTION_SEND).apply {
    type = "text/plain"
    putExtra(Intent.EXTRA_TEXT, "Check out this awesome app!")
    putExtra(Intent.EXTRA_SUBJECT, "App Recommendation")
}
startActivity(Intent.createChooser(shareIntent, "Share via"))

// Open email
val emailIntent = Intent(Intent.ACTION_SENDTO).apply {
    data = Uri.parse("mailto:jayed@gmail.com")
    putExtra(Intent.EXTRA_SUBJECT, "Hello!")
}
if (emailIntent.resolveActivity(packageManager) != null) {
    startActivity(emailIntent)
}
```

---

### 3. `startActivityForResult` → `ActivityResultLauncher` (BIG CHANGE)

**Old way (DEPRECATED — don't use!):**
```java
// Java old way — confusing with request codes
startActivityForResult(intent, REQUEST_CODE_PICK_IMAGE);

@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == REQUEST_CODE_PICK_IMAGE && resultCode == RESULT_OK) {
        Uri imageUri = data.getData();
        // use imageUri
    }
}
```

**New modern way — `ActivityResultLauncher` ✅:**
```kotlin
// Step 1: Register the launcher — do this at class level, NOT inside a method!
private val pickImageLauncher =
    registerForActivityResult(ActivityResultContracts.GetContent()) { uri: Uri? ->
        // This runs when the image picker returns
        uri?.let {
            binding.profileImage.setImageURI(it)
        }
    }

// Step 2: Launch it when needed
binding.pickPhotoButton.setOnClickListener {
    pickImageLauncher.launch("image/*")  // filter: show only images
}
```

**More ActivityResultContracts examples:**
```kotlin
// Request a single permission
private val requestPermission =
    registerForActivityResult(ActivityResultContracts.RequestPermission()) { isGranted ->
        if (isGranted) startCamera()
        else showPermissionDeniedMessage()
    }
binding.cameraButton.setOnClickListener {
    requestPermission.launch(Manifest.permission.CAMERA)
}

// Take a photo
private val takePicture =
    registerForActivityResult(ActivityResultContracts.TakePicturePreview()) { bitmap ->
        bitmap?.let { binding.imageView.setImageBitmap(it) }
    }

// Start another activity and get result
private val openSettings =
    registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { result ->
        if (result.resultCode == Activity.RESULT_OK) {
            val data = result.data?.getStringExtra("SETTING_VALUE")
        }
    }
```

---

### 4. Intent Flags — Control Back Stack

```kotlin
// Clear all previous activities and start fresh (used in login → home flow)
val intent = Intent(this, HomeActivity::class.java).apply {
    flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
}
startActivity(intent)
// User can no longer press Back to go back to Login!

// Bring existing activity to top if it exists (don't create new instance)
Intent(this, MainActivity::class.java).also {
    it.flags = Intent.FLAG_ACTIVITY_CLEAR_TOP or Intent.FLAG_ACTIVITY_SINGLE_TOP
    startActivity(it)
}
```

---

### 5. PendingIntent — Intent for Later Use

```kotlin
// PendingIntent is an intent that runs LATER (for Notifications, Widgets, Alarms)
val openAppIntent = Intent(this, MainActivity::class.java)
val pendingIntent = PendingIntent.getActivity(
    this,
    0,                      // request code
    openAppIntent,
    PendingIntent.FLAG_IMMUTABLE  // required on Android 12+
)
// Now give pendingIntent to a Notification — when user taps notification, app opens
```

---

### 6. Deep Links — Open Your App from a URL

```xml
<!-- In AndroidManifest.xml — add to your Activity -->
<intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="https"
          android:host="myapp.com"
          android:pathPrefix="/product" />
</intent-filter>
```

```kotlin
// In Activity — handle the deep link
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    // Check if opened via deep link
    val uri = intent.data
    if (uri != null) {
        val productId = uri.getQueryParameter("id")
        // https://myapp.com/product?id=42 → productId = "42"
        loadProduct(productId)
    }
}
```

---

## 🔑 Key Concepts

| Concept | Use Case | Modern Way |
|---------|---------|-----------|
| Explicit Intent | Open your own screen | `Intent(this, TargetActivity::class.java)` |
| Implicit Intent | Open system apps | `Intent(Intent.ACTION_VIEW, ...)` |
| Pass data | Send values to next screen | `putExtra()` / `getStringExtra()` |
| Get result back | Photo picker, file picker, permissions | `ActivityResultLauncher` |
| Back stack control | Login→Home (no back) | `FLAG_ACTIVITY_CLEAR_TASK` |
| PendingIntent | Notifications, Alarms | `PendingIntent.getActivity()` |
| Deep Link | URL → opens app screen | `intent-filter` in Manifest |

---

## 💡 Good Example — Complete Login → Home Flow

```kotlin
class LoginActivity : AppCompatActivity() {
    private lateinit var binding: ActivityLoginBinding

    // Launcher to open Google Sign-In and get result
    private val signInLauncher =
        registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { result ->
            if (result.resultCode == RESULT_OK) {
                val account = GoogleSignIn.getSignedInAccountFromIntent(result.data)
                navigateToHome(account.result.displayName ?: "User")
            }
        }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityLoginBinding.inflate(layoutInflater)
        setContentView(binding.root)

        binding.btnLogin.setOnClickListener {
            val email = binding.etEmail.text.toString()
            val pass = binding.etPassword.text.toString()
            if (email.isNotEmpty() && pass.isNotEmpty()) {
                navigateToHome(email)
            }
        }

        binding.btnGoogleSignIn.setOnClickListener {
            val signInIntent = GoogleSignIn.getClient(this, GoogleSignInOptions.DEFAULT_SIGN_IN).signInIntent
            signInLauncher.launch(signInIntent)
        }
    }

    private fun navigateToHome(userName: String) {
        val intent = Intent(this, HomeActivity::class.java).apply {
            putExtra("USER_NAME", userName)
            // Clear back stack — user can't press back to go to Login
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
        }
        startActivity(intent)
    }
}
```
