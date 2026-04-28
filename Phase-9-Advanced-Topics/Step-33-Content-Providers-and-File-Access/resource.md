# Step 33 — Content Providers & File Access

---

## 📖 What Is It? (Simple Explanation)

### Content Provider
Imagine a **government ID office** 🏛️. Two different apps are like two different people. Normally, they can't touch each other's files. But if App A opens a "window" (Content Provider), App B can come to that window and ask for data — and App A decides what to share.

**Content Providers** are that window between apps for sharing data (contacts, media, custom data).

### File Access
Think of your phone's storage like a **filing cabinet** 🗂️. You have your own drawer (app's private storage). You also have shared drawers (external storage, documents). The rules about who can open which drawer have **changed a lot** since Android 10+.

---

## 🧭 Where Do We Use It?

| Feature | Use Case |
|---|---|
| `ContentProvider` | Share your app's database with other apps |
| `FileProvider` | Share a file (like a photo) with another app safely |
| `MediaStore` | Access device photos, videos, music |
| `SAF` (Storage Access Framework) | Let user pick any document / directory |
| `BlobStoreManager` | Share large binary data between apps efficiently |

---

## 🗺️ Workflow / How It Works

### FileProvider (Sharing a File)

```
Your App wants to share an image with a Camera/Share app
            │
            ▼
FileProvider converts internal file path
to a secure content:// URI
            │
            ▼
Shared app receives safe URI
→ reads file without knowing your private path
```

### MediaStore (Accessing Photos)

```
App wants to read photos
            │
            ▼
Query MediaStore.Images.Media.EXTERNAL_CONTENT_URI
            │
            ▼
System returns cursor with photo URIs
            │
            ▼
Load photo using URI (never raw file path)
```

### SAF (Storage Access Framework)

```
User taps "Open File"
            │
            ▼
System file picker opens (any app can handle it)
            │
            ▼
User picks a file
            │
            ▼
App receives a content:// URI with permission
```

---

## ☕ Java vs Kotlin — What Changed?

### Old Way — File Access (Java, direct file paths, insecure)

```java
// Java — old insecure way (crashes on Android 7+ without FileProvider)
File file = new File(Environment.getExternalStorageDirectory(), "photo.jpg");
Uri uri = Uri.fromFile(file);  // ❌ FileUriExposedException on API 24+
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.setDataAndType(uri, "image/jpeg");
startActivity(intent);
```

### New Way — FileProvider (Kotlin, secure content URI)

```kotlin
// Kotlin — modern secure way
val file = File(cacheDir, "photo.jpg")
val uri = FileProvider.getUriForFile(
    this,
    "${packageName}.fileprovider",  // authority from manifest
    file
)
val intent = Intent(Intent.ACTION_VIEW).apply {
    setDataAndType(uri, "image/jpeg")
    addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)  // grant temporary read access
}
startActivity(intent)
```

---

## 🔑 Key Concepts

---

### 1. `ContentProvider` — Share Your App's Data

```kotlin
class MyNotesProvider : ContentProvider() {

    private lateinit var db: NotesDatabase

    override fun onCreate(): Boolean {
        db = NotesDatabase.getInstance(context!!)
        return true
    }

    override fun query(
        uri: Uri, projection: Array<out String>?,
        selection: String?, selectionArgs: Array<out String>?,
        sortOrder: String?
    ): Cursor? {
        return db.noteDao().getAllAsCursor()
    }

    override fun insert(uri: Uri, values: ContentValues?): Uri? { TODO() }
    override fun update(uri: Uri, values: ContentValues?, selection: String?, selectionArgs: Array<out String>?) = 0
    override fun delete(uri: Uri, selection: String?, selectionArgs: Array<out String>?) = 0
    override fun getType(uri: Uri) = "vnd.android.cursor.dir/vnd.com.example.notes"
}
```

**`AndroidManifest.xml`**
```xml
<provider
    android:name=".MyNotesProvider"
    android:authorities="com.example.notesprovider"
    android:exported="true"/>
```

**Another app reads from it:**
```kotlin
val cursor = contentResolver.query(
    Uri.parse("content://com.example.notesprovider/notes"),
    null, null, null, null
)
cursor?.use { c ->
    while (c.moveToNext()) {
        val title = c.getString(c.getColumnIndexOrThrow("title"))
        println(title)
    }
}
```

---

### 2. `FileProvider` — Share Files Safely

**Step 1: Declare in manifest**
```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:grantUriPermissions="true"
    android:exported="false">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths"/>
</provider>
```

**Step 2: Define paths (`res/xml/file_paths.xml`)**
```xml
<paths>
    <cache-path name="camera_photos" path="photos/"/>
    <files-path name="shared_files" path="share/"/>
    <external-files-path name="external" path="."/>
</paths>
```

**Step 3: Use in code**
```kotlin
// Share a file via intent
fun shareFile(file: File) {
    val uri = FileProvider.getUriForFile(this, "$packageName.fileprovider", file)
    val shareIntent = Intent(Intent.ACTION_SEND).apply {
        type = "image/jpeg"
        putExtra(Intent.EXTRA_STREAM, uri)
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
    }
    startActivity(Intent.createChooser(shareIntent, "Share image via"))
}

// Take a photo and save to cache
fun takePhoto() {
    val photoFile = File(cacheDir, "photos/captured.jpg").also { it.parentFile?.mkdirs() }
    cameraUri = FileProvider.getUriForFile(this, "$packageName.fileprovider", photoFile)
    cameraLauncher.launch(cameraUri)
}
```

---

### 3. `MediaStore` — Access Device Photos/Videos (Android 10+)

```kotlin
// Query all images
fun getDevicePhotos(): List<Uri> {
    val photos = mutableListOf<Uri>()
    val projection = arrayOf(
        MediaStore.Images.Media._ID,
        MediaStore.Images.Media.DISPLAY_NAME,
        MediaStore.Images.Media.DATE_ADDED
    )
    val sortOrder = "${MediaStore.Images.Media.DATE_ADDED} DESC"

    contentResolver.query(
        MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
        projection, null, null, sortOrder
    )?.use { cursor ->
        val idColumn = cursor.getColumnIndexOrThrow(MediaStore.Images.Media._ID)
        while (cursor.moveToNext()) {
            val id = cursor.getLong(idColumn)
            val uri = ContentUris.withAppendedId(
                MediaStore.Images.Media.EXTERNAL_CONTENT_URI, id
            )
            photos.add(uri)
        }
    }
    return photos
}

// Save a photo to gallery (Android 10+, no permission needed with MediaStore)
fun savePhotoToGallery(bitmap: Bitmap) {
    val values = ContentValues().apply {
        put(MediaStore.Images.Media.DISPLAY_NAME, "photo_${System.currentTimeMillis()}.jpg")
        put(MediaStore.Images.Media.MIME_TYPE, "image/jpeg")
        put(MediaStore.Images.Media.RELATIVE_PATH, Environment.DIRECTORY_PICTURES)
        put(MediaStore.Images.Media.IS_PENDING, 1)
    }
    val uri = contentResolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values)!!
    contentResolver.openOutputStream(uri)?.use { stream ->
        bitmap.compress(Bitmap.CompressFormat.JPEG, 90, stream)
    }
    values.clear()
    values.put(MediaStore.Images.Media.IS_PENDING, 0)
    contentResolver.update(uri, values, null, null)
}
```

---

### 4. SAF — Storage Access Framework (Let User Pick Any File)

```kotlin
// Open file picker
private val openFileLauncher =
    registerForActivityResult(ActivityResultContracts.OpenDocument()) { uri ->
        if (uri != null) {
            // Read the file (URI stays valid during this session)
            val content = contentResolver.openInputStream(uri)?.bufferedReader()?.readText()
            textView.text = content
        }
    }

// Launch — let user pick a PDF or TXT
openFileLauncher.launch(arrayOf("application/pdf", "text/plain"))

// Pick a directory
private val openDirLauncher =
    registerForActivityResult(ActivityResultContracts.OpenDocumentTree()) { uri ->
        uri?.let {
            // persist permission for future access
            contentResolver.takePersistableUriPermission(
                it, Intent.FLAG_GRANT_READ_URI_PERMISSION
            )
            // list files in directory...
        }
    }
openDirLauncher.launch(null)
```

---

### 5. Photo Picker API (Android 13+ — Simpler Gallery Access)

```kotlin
// No permission required for photos/videos on Android 13+
val pickMedia = registerForActivityResult(ActivityResultContracts.PickVisualMedia()) { uri ->
    if (uri != null) {
        imageView.setImageURI(uri)
    }
}

// Pick a single image
pickMedia.launch(PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly))

// Pick image or video
pickMedia.launch(PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageAndVideo))

// Pick multiple images
val pickMultipleMedia = registerForActivityResult(
    ActivityResultContracts.PickMultipleVisualMedia(5) // max 5
) { uris -> /* use uris list */ }
```

---

## 💡 Scoped Storage — The Big Change (Android 10+)

```
Pre-Android 10:         Android 10+ (Scoped Storage):
──────────────          ──────────────────────────────
/sdcard/Pictures/  ←    App can only see its own files in
  freely readable        /sdcard/Android/data/com.example/
  by all apps
                         To share: use MediaStore or SAF
                         To share files with apps: use FileProvider
```

| Storage Type | How to Access |
|---|---|
| App private files | `filesDir`, `cacheDir` — no permission needed |
| App external files | `getExternalFilesDir()` — no permission needed |
| Shared media (photos) | `MediaStore` API — `READ_MEDIA_IMAGES` (Android 13+) or Photo Picker |
| User documents | SAF (`OpenDocument`, `OpenDocumentTree`) |

---

## 📊 Quick Comparison Summary

| Old (Java) | New (Kotlin / Modern Android) |
|---|---|
| `Uri.fromFile(file)` — crashed on API 24+ | `FileProvider.getUriForFile()` |
| `REQUEST_EXTERNAL_STORAGE` permission | Scoped storage — no permission for own directory |
| `File(Environment.getExternalStorageDirectory(), "x")` | `MediaStore` API or `getExternalFilesDir()` |
| `startActivityForResult(pickIntent)` + `onActivityResult` | `ActivityResultContracts.OpenDocument()` launcher |
| Manual cursor parsing everywhere | `use { cursor -> }` Kotlin extension |
| `READ_EXTERNAL_STORAGE` for gallery | `READ_MEDIA_IMAGES` (API 33+) or Photo Picker |
