# Step 23 — WorkManager (Replaces AlarmManager / JobScheduler)

---

## 📖 What Is It? (Simple Definition)

Imagine you want your app to sync data with the server every hour — even when your phone restarts, or the battery dies and charges back up, or the app is closed.

**WorkManager** is the reliable janitor who always gets the job done, no matter what. You give it a task and constraints ("only when connected to WiFi"), and it guarantees it will run.

It internally uses the best available mechanism (JobScheduler, AlarmManager, or Firebase JobDispatcher) based on the device's Android version.

---

## 🎯 Where Do We Use It?

- Background data sync (every hour, every day)
- Upload logs/analytics when phone is connected to WiFi
- Database cleanup
- Image compression before upload
- Sending reports at the end of the day
- Any work that must complete eventually, even if app is closed

> 🚨 NOT for immediate tasks! Use Coroutines for that.  
> WorkManager = "guaranteed eventually" vs Coroutines = "immediately"

---

## 🔄 WorkManager Architecture Diagram

```
You define Work:
  WorkRequest (what to do)
    ├── OneTimeWorkRequest  → do it ONCE
    └── PeriodicWorkRequest → repeat every N minutes/hours

Add Constraints (conditions):
    ├── Requires network connection
    ├── Requires charging
    ├── Requires storage not low
    └── Requires device idle

WorkManager schedules it:
  ├── App open?    → Run now on background thread
  ├── App closed?  → Run when app has opportunity
  └── Rebooted?    → Rescheduled automatically!

Worker runs → returns:
  ├── Result.success()  → done!
  ├── Result.failure()  → failed (won't retry by default)
  └── Result.retry()    → try again later
```

---

## ☕ Java vs Kotlin — What Changed?

### Old Java Ways — Replaced by WorkManager

```java
// Old AlarmManager — complex, didn't survive reboots properly
AlarmManager alarmManager = (AlarmManager) getSystemService(ALARM_SERVICE);
Intent intent = new Intent(this, AlarmReceiver.class);
PendingIntent pendingIntent = PendingIntent.getBroadcast(this, 0, intent, 0);
alarmManager.setRepeating(AlarmManager.ELAPSED_REALTIME_WAKEUP,
    SystemClock.elapsedRealtime(), 60 * 60 * 1000, pendingIntent); // every hour

// Old JobScheduler — Android 5.0+ only, verbose API
// Firebase JobDispatcher — Google Play Services required
```

### New Way — WorkManager ✅

```kotlin
// Step 1: Add to build.gradle
// implementation("androidx.work:work-runtime-ktx:2.9.0")

// Step 2: Define the Worker
class SyncDataWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {  // CoroutineWorker = suspend support!

    override suspend fun doWork(): Result {
        return try {
            // Get input data (data passed when creating the request)
            val userId = inputData.getInt("USER_ID", -1)
            if (userId == -1) return Result.failure()

            // Do actual work (this runs in background automatically)
            val users = apiService.getLatestData(userId)
            database.userDao().insertAll(users)

            // Set output data (available to chained workers or observers)
            val outputData = workDataOf("SYNCED_COUNT" to users.size)
            Result.success(outputData)

        } catch (e: IOException) {
            // Retry on network errors
            Result.retry()
        } catch (e: Exception) {
            // Fail on other errors
            Result.failure()
        }
    }

    companion object {
        const val WORK_NAME = "sync_data_work"
    }
}
```

---

### OneTimeWorkRequest — Do Task Once

```kotlin
// Basic one-time work (no constraints)
val syncRequest = OneTimeWorkRequestBuilder<SyncDataWorker>()
    .setInputData(workDataOf("USER_ID" to 42))
    .build()

WorkManager.getInstance(context).enqueue(syncRequest)

// With constraints — only run when connected to internet
val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)
    .setRequiresBatteryNotLow(true)  // don't drain battery
    .build()

val syncWithConstraints = OneTimeWorkRequestBuilder<SyncDataWorker>()
    .setConstraints(constraints)
    .setInputData(workDataOf("USER_ID" to 42))
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 15, TimeUnit.MINUTES) // retry timing
    .addTag("sync_tag")   // for cancellation/observation by tag
    .build()

WorkManager.getInstance(context).enqueue(syncWithConstraints)
```

---

### PeriodicWorkRequest — Repeat at Interval

```kotlin
// Minimum interval is 15 minutes (Android OS limitation for battery)
val periodicSyncRequest = PeriodicWorkRequestBuilder<SyncDataWorker>(
    repeatInterval = 1,
    repeatIntervalTimeUnit = TimeUnit.HOURS  // every hour
)
    .setConstraints(
        Constraints.Builder()
            .setRequiredNetworkType(NetworkType.UNMETERED)  // WiFi only!
            .setRequiresCharging(true)  // only when charging (optional)
            .build()
    )
    .build()

// Enqueue uniquely — only one copy runs at a time!
WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "HourlySync",                          // unique name
    ExistingPeriodicWorkPolicy.KEEP,       // KEEP or REPLACE existing
    periodicSyncRequest
)
```

---

### Chaining Work — Run Workers in Sequence or Parallel

```kotlin
// Sequential: Step 1 → Step 2 → Step 3
WorkManager.getInstance(context)
    .beginWith(OneTimeWorkRequestBuilder<CompressImagesWorker>().build())
    .then(OneTimeWorkRequestBuilder<UploadImagesWorker>().build())
    .then(OneTimeWorkRequestBuilder<NotifyServerWorker>().build())
    .enqueue()

// Parallel then sequential:
// Fetch Users  ─┐
//               ├─→ CombineAndSave
// Fetch Products─┘

val fetchUsersWork = OneTimeWorkRequestBuilder<FetchUsersWorker>().build()
val fetchProductsWork = OneTimeWorkRequestBuilder<FetchProductsWorker>().build()
val combineWork = OneTimeWorkRequestBuilder<CombineDataWorker>().build()

WorkManager.getInstance(context)
    .beginWith(listOf(fetchUsersWork, fetchProductsWork))  // parallel!
    .then(combineWork)                                     // sequential after both
    .enqueue()
```

---

### Observing Work Status

```kotlin
// Observe by ID
WorkManager.getInstance(context)
    .getWorkInfoByIdLiveData(syncRequest.id)
    .observe(viewLifecycleOwner) { workInfo ->
        when (workInfo?.state) {
            WorkInfo.State.ENQUEUED  -> binding.status.text = "Waiting..."
            WorkInfo.State.RUNNING   -> binding.status.text = "Syncing..."
            WorkInfo.State.SUCCEEDED -> {
                val count = workInfo.outputData.getInt("SYNCED_COUNT", 0)
                binding.status.text = "Synced $count items!"
            }
            WorkInfo.State.FAILED    -> binding.status.text = "Sync failed"
            WorkInfo.State.CANCELLED -> binding.status.text = "Cancelled"
            else -> {}
        }
    }

// Observe by tag
WorkManager.getInstance(context)
    .getWorkInfosByTagLiveData("sync_tag")
    .observe(viewLifecycleOwner) { workInfos ->
        val isRunning = workInfos.any { it.state == WorkInfo.State.RUNNING }
        binding.progressBar.isVisible = isRunning
    }

// Cancel work
WorkManager.getInstance(context).cancelWorkById(syncRequest.id)
WorkManager.getInstance(context).cancelUniqueWork("HourlySync")
WorkManager.getInstance(context).cancelAllWorkByTag("sync_tag")
```

---

## 🔑 Key Concepts

| Concept | What It Does | Key Note |
|---------|-------------|---------|
| `Worker` | Synchronous worker | Runs on background thread |
| `CoroutineWorker` | Async worker with suspend | Preferred for Kotlin |
| `OneTimeWorkRequest` | Run task once | With + without constraints |
| `PeriodicWorkRequest` | Run task repeatedly | Min 15 min interval |
| `Constraints` | Conditions that must be met | Network, charging, battery, storage |
| `Result.success()` | Work completed | Carries output data |
| `Result.failure()` | Work failed, don't retry | Unrecoverable error |
| `Result.retry()` | Try again later | Network timeout, temporary error |
| `enqueueUniqueWork` | Ensure only one instance runs | Use KEEP or REPLACE policy |
| `WorkInfo` | Current work state | Observe via LiveData/Flow |

---

## 💡 Good Example — Photo Upload Worker

```kotlin
class PhotoUploadWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val photoPath = inputData.getString("PHOTO_PATH")
            ?: return Result.failure(workDataOf("ERROR" to "No photo path"))

        val photoFile = File(photoPath)
        if (!photoFile.exists()) return Result.failure(workDataOf("ERROR" to "File not found"))

        return try {
            // Update progress notification
            setForeground(createForegroundInfo("Uploading photo..."))

            val photoBytes = photoFile.readBytes()
            val body = photoBytes.toRequestBody("image/jpeg".toMediaTypeOrNull())
            val part = MultipartBody.Part.createFormData("photo", photoFile.name, body)

            val response = apiService.uploadPhoto(part)

            if (response.isSuccessful) {
                val photoUrl = response.body()?.url ?: ""
                // Save URL to database
                database.photoDao().updateUrl(photoPath, photoUrl)
                Result.success(workDataOf("PHOTO_URL" to photoUrl))
            } else {
                if (response.code() in 500..599) Result.retry()  // server error — retry
                else Result.failure(workDataOf("ERROR" to "HTTP ${response.code()}"))
            }

        } catch (e: IOException) {
            Result.retry()  // network error — retry
        }
    }

    private fun createForegroundInfo(message: String): ForegroundInfo {
        val notification = NotificationCompat.Builder(applicationContext, "UPLOAD_CHANNEL")
            .setSmallIcon(R.drawable.ic_upload)
            .setContentTitle(message)
            .setOngoing(true)
            .build()
        return ForegroundInfo(NOTIFICATION_ID, notification)
    }

    companion object {
        const val NOTIFICATION_ID = 2001

        fun buildRequest(photoPath: String): OneTimeWorkRequest {
            return OneTimeWorkRequestBuilder<PhotoUploadWorker>()
                .setInputData(workDataOf("PHOTO_PATH" to photoPath))
                .setConstraints(
                    Constraints.Builder()
                        .setRequiredNetworkType(NetworkType.CONNECTED)
                        .build()
                )
                .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 10, TimeUnit.MINUTES)
                .addTag("photo_upload")
                .build()
        }
    }
}

// In ViewModel:
fun uploadPhoto(photoPath: String) {
    val request = PhotoUploadWorker.buildRequest(photoPath)
    WorkManager.getInstance(context).enqueue(request)
}
```
