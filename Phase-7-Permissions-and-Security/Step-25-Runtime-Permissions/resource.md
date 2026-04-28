# Step 25 — Runtime Permissions (Updated)

---

## 📖 What Is It? (Simple Explanation)

Imagine your phone is your house 🏠. When a guest (app) wants to enter your bedroom (camera, contacts, location), they have to **knock and ask permission** first. You can say **Yes** or **No**. If you say No, they cannot enter.

That is exactly what **Runtime Permissions** are. Since Android 6.0 (API 23), apps must **ask the user at the moment they need** a sensitive permission — not quietly at install time like the old days.

---

## 🧭 Where Do We Use It?

| Situation | Permission Needed |
|---|---|
| Taking a photo / video | `CAMERA` |
| Accessing contacts | `READ_CONTACTS` |
| Getting GPS location | `ACCESS_FINE_LOCATION` |
| Reading images from gallery | `READ_MEDIA_IMAGES` (Android 13+) |
| Posting a notification | `POST_NOTIFICATIONS` (Android 13+) |
| Scanning nearby Wi-Fi devices | `NEARBY_WIFI_DEVICES` (Android 12+) |
| Setting exact alarms | `SCHEDULE_EXACT_ALARM` (Android 12+) |

---

## 🗺️ Workflow / How It Works

```
App needs Camera
       │
       ▼
Is permission already GRANTED?
       │
  ┌────┴────┐
 YES        NO
  │          │
  ▼          ▼
Go ahead   Should show Rationale?
           ┌────┴────┐
          YES        NO
           │          │
           ▼          ▼
      Show a dialog  Launch Permission
      "Why we need   Request Dialog
       this..." then
       ask again
                │
         ┌──────┴──────┐
        GRANTED       DENIED
         │              │
         ▼              ▼
     Proceed        Disable feature
                    gracefully
```

---

## ☕ Java vs Kotlin — What Changed?

### Old Way (Java — `startActivityForResult` style, messy boilerplate)

```java
// Java — old way (about 40 lines total across callbacks)
private static final int CAMERA_PERMISSION_CODE = 101;

// Step 1: request
ActivityCompat.requestPermissions(this,
    new String[]{Manifest.permission.CAMERA},
    CAMERA_PERMISSION_CODE);

// Step 2: handle result in a completely different method
@Override
public void onRequestPermissionsResult(int requestCode,
        String[] permissions, int[] grantResults) {
    super.onRequestPermissionsResult(requestCode, permissions, grantResults);
    if (requestCode == CAMERA_PERMISSION_CODE) {
        if (grantResults.length > 0 &&
                grantResults[0] == PackageManager.PERMISSION_GRANTED) {
            openCamera(); // scattered logic
        } else {
            Toast.makeText(this, "Permission denied", Toast.LENGTH_SHORT).show();
        }
    }
}
```

> Problems: you split the request and result into **two separate methods**. Hard to read, easy to mix up request codes.

### New Way (Kotlin — `ActivityResultLauncher`, ~15 lines, all in one place)

```kotlin
// Kotlin — modern way (all logic in one place)
private val requestPermissionLauncher =
    registerForActivityResult(ActivityResultContracts.RequestPermission()) { isGranted ->
        if (isGranted) {
            openCamera()
        } else {
            showPermissionDeniedMessage()
        }
    }

// Launch it anywhere
requestPermissionLauncher.launch(Manifest.permission.CAMERA)
```

> ✅ Request + result in **one block**. Clean, readable, no magic numbers.

---

## 🔑 Key Concepts

### 1. Normal vs Dangerous Permissions

| Type | Example | User Dialog? |
|---|---|---|
| **Normal** | `INTERNET`, `VIBRATE` | ❌ Granted automatically |
| **Dangerous** | `CAMERA`, `LOCATION`, `CONTACTS` | ✅ Must ask user at runtime |

### 2. `ActivityResultLauncher` Types

```kotlin
// Single permission
ActivityResultContracts.RequestPermission()

// Multiple permissions at once
ActivityResultContracts.RequestMultiplePermissions()
```

### 3. `shouldShowRequestPermissionRationale()`

When the user previously **denied** the permission, the system expects you to explain *why* before asking again. This function returns `true` when you should show that explanation.

```kotlin
if (shouldShowRequestPermissionRationale(Manifest.permission.CAMERA)) {
    // Show a dialog: "We need camera to scan QR codes"
    showRationaleDialog()
} else {
    requestPermissionLauncher.launch(Manifest.permission.CAMERA)
}
```

### 4. New Android 12+ / 13+ Permissions

```kotlin
// Android 13 — reading images only (no need to read ALL files)
Manifest.permission.READ_MEDIA_IMAGES
Manifest.permission.READ_MEDIA_VIDEO
Manifest.permission.READ_MEDIA_AUDIO

// Android 13 — sending notifications
Manifest.permission.POST_NOTIFICATIONS

// Android 12 — nearby WiFi scanning
Manifest.permission.NEARBY_WIFI_DEVICES

// Android 12 — exact alarms
Manifest.permission.SCHEDULE_EXACT_ALARM
```

### 5. Photo Picker API (Android 13+)

Instead of asking `READ_MEDIA_IMAGES` permission, use the **system Photo Picker**. It lets users pick photos **without giving your app full gallery access**.

```kotlin
val pickMedia = registerForActivityResult(
    ActivityResultContracts.PickVisualMedia()
) { uri ->
    if (uri != null) {
        imageView.setImageURI(uri) // user picked a photo
    }
}

// Launch it — NO permission needed!
pickMedia.launch(PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly))
```

---

## 💡 Full Practical Example

### Goal: Ask for Camera Permission, then open the camera

#### `AndroidManifest.xml`
```xml
<uses-permission android:name="android.permission.CAMERA" />
```

#### `MainActivity.kt`
```kotlin
class MainActivity : AppCompatActivity() {

    // Step 1: Register the launcher (like setting up a mailbox)
    private val cameraPermissionLauncher =
        registerForActivityResult(ActivityResultContracts.RequestPermission()) { isGranted ->
            if (isGranted) {
                Toast.makeText(this, "Camera opened!", Toast.LENGTH_SHORT).show()
                openCamera()
            } else {
                Toast.makeText(this, "Camera permission denied.", Toast.LENGTH_SHORT).show()
            }
        }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        findViewById<Button>(R.id.btnCamera).setOnClickListener {
            checkAndRequestCameraPermission()
        }
    }

    // Step 2: Check before asking
    private fun checkAndRequestCameraPermission() {
        when {
            // Already allowed? Just open.
            ContextCompat.checkSelfPermission(
                this, Manifest.permission.CAMERA
            ) == PackageManager.PERMISSION_GRANTED -> {
                openCamera()
            }

            // Denied before? Explain first.
            shouldShowRequestPermissionRationale(Manifest.permission.CAMERA) -> {
                AlertDialog.Builder(this)
                    .setTitle("Camera Permission Needed")
                    .setMessage("We need the camera to let you take profile photos.")
                    .setPositiveButton("OK") { _, _ ->
                        cameraPermissionLauncher.launch(Manifest.permission.CAMERA)
                    }
                    .setNegativeButton("Cancel", null)
                    .show()
            }

            // First time asking
            else -> {
                cameraPermissionLauncher.launch(Manifest.permission.CAMERA)
            }
        }
    }

    private fun openCamera() {
        // Open camera logic here
    }
}
```

---

## 🔄 Multiple Permissions at Once

```kotlin
private val multiplePermissionsLauncher =
    registerForActivityResult(ActivityResultContracts.RequestMultiplePermissions()) { permissions ->
        val cameraGranted = permissions[Manifest.permission.CAMERA] ?: false
        val locationGranted = permissions[Manifest.permission.ACCESS_FINE_LOCATION] ?: false

        if (cameraGranted && locationGranted) {
            startFeature()
        } else {
            Toast.makeText(this, "Both permissions are needed.", Toast.LENGTH_SHORT).show()
        }
    }

// Launch
multiplePermissionsLauncher.launch(
    arrayOf(
        Manifest.permission.CAMERA,
        Manifest.permission.ACCESS_FINE_LOCATION
    )
)
```

---

## 🚫 Best Practices

- ✅ Only ask permissions when the user **performs an action** that needs it
- ✅ Explain **why** before asking (rationale dialog)
- ✅ Gracefully disable the feature if denied — don't crash
- ✅ Use **Photo Picker** instead of storage permissions when possible
- ❌ Never ask all permissions on app start
- ❌ Never show the permission dialog in a loop after denial

---

## 📊 Quick Comparison Summary

| Old (Java) | New (Kotlin) |
|---|---|
| `requestPermissions()` + `onRequestPermissionsResult()` | `ActivityResultLauncher` (all in one) |
| Magic integer request codes | No request codes needed |
| Logic split across methods | Logic in one callback |
| `READ_EXTERNAL_STORAGE` for gallery | `READ_MEDIA_IMAGES` or Photo Picker |
| No notification permission needed | `POST_NOTIFICATIONS` required (API 33+) |
