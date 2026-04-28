# Step 21 — Services (Updated)

---

## 📖 What Is It? (Simple Definition)

A **Service** is like a worker who keeps doing a job even when you're not looking at the screen.

- **Foreground Service** = A visible worker with a hardhat and a notification — "I'm downloading your file!" Users can see they're working.
- **Bound Service** = A worker you hire for a specific task — you talk to them, they respond, you dismiss them when done.

> ⚠️ Android is very strict about background work now! Most background jobs should use **WorkManager** (Step 23) instead of plain Services.

---

## 🎯 Where Do We Use It?

| Service Type | Use Case |
|-------------|---------|
| Foreground Service | Music playback, GPS tracking, file upload/download, ongoing calls |
| Bound Service | Communication between your own app's components |
| WorkManager (use instead!) | Periodic sync, off-screen data upload, background processing |

---

## 🔄 Service Lifecycle Diagram

```
Started Service (startService / startForeground):
  startService(intent)
        ↓
  onCreate()   ← called once
        ↓
  onStartCommand()  ← called every time startService is called
        ↓
  [Running…]
        ↓
  stopSelf() or stopService()
        ↓
  onDestroy()

Bound Service (bindService):
  bindService(intent, connection, flags)
        ↓
  onCreate() → onBind() returns IBinder
        ↓
  [Client communicates with service]
        ↓
  unbindService()  (no more clients)
        ↓
  onUnbind() → onDestroy()
```

---

## ☕ Java vs Kotlin — What Changed?

### `IntentService` is Deprecated → Use `CoroutineWorker` (WorkManager)

```java
// Java IntentService — deprecated!
public class MyIntentService extends IntentService {
    @Override
    protected void onHandleIntent(Intent intent) {
        // runs in background thread automatically
        doHeavyWork();
    }
}
```

```kotlin
// Kotlin replacement — use WorkManager CoroutineWorker (Step 23)
class MyWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        doHeavyWork()
        return Result.success()
    }
}
```

### Foreground Service — Music Player Example

```kotlin
// 1. Declare in AndroidManifest.xml
// <service android:name=".MusicService" android:foregroundServiceType="mediaPlayback"/>
// <uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>

class MusicService : Service() {
    private var mediaPlayer: MediaPlayer? = null
    private val binder = MusicBinder()

    inner class MusicBinder : Binder() {
        fun getService(): MusicService = this@MusicService
    }

    override fun onCreate() {
        super.onCreate()
        mediaPlayer = MediaPlayer()
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        when (intent?.action) {
            ACTION_PLAY  -> play(intent.getStringExtra("URL") ?: "")
            ACTION_PAUSE -> pause()
            ACTION_STOP  -> stopPlayback()
        }
        return START_STICKY  // restart if killed by system
    }

    private fun play(url: String) {
        // MUST call startForeground within 5 seconds for Foreground Service!
        startForeground(NOTIFICATION_ID, buildNotification("Playing..."))
        mediaPlayer?.apply {
            reset()
            setDataSource(url)
            setOnPreparedListener { start() }
            prepareAsync()
        }
    }

    private fun pause() {
        mediaPlayer?.pause()
        // Update notification
        val notificationManager = getSystemService(NotificationManager::class.java)
        notificationManager.notify(NOTIFICATION_ID, buildNotification("Paused"))
    }

    private fun stopPlayback() {
        mediaPlayer?.stop()
        stopForeground(STOP_FOREGROUND_REMOVE)  // remove notification
        stopSelf()  // stop the service
    }

    // For Bound Service clients
    override fun onBind(intent: Intent?): IBinder = binder

    private fun buildNotification(status: String): Notification {
        return NotificationCompat.Builder(this, "MUSIC_CHANNEL")
            .setSmallIcon(R.drawable.ic_music)
            .setContentTitle("Music Player")
            .setContentText(status)
            .setOngoing(true)
            .build()
    }

    override fun onDestroy() {
        super.onDestroy()
        mediaPlayer?.release()
        mediaPlayer = null
    }

    companion object {
        const val ACTION_PLAY = "ACTION_PLAY"
        const val ACTION_PAUSE = "ACTION_PAUSE"
        const val ACTION_STOP = "ACTION_STOP"
        const val NOTIFICATION_ID = 1001
    }
}
```

### Starting and Controlling the Foreground Service

```kotlin
// In Activity/Fragment:

// Start service
val intent = Intent(context, MusicService::class.java).apply {
    action = MusicService.ACTION_PLAY
    putExtra("URL", "https://example.com/music.mp3")
}
// For Foreground Services on Android 8+, use startForegroundService
ContextCompat.startForegroundService(context, intent)

// Stop/Pause via intents
val pauseIntent = Intent(context, MusicService::class.java).apply {
    action = MusicService.ACTION_PAUSE
}
startService(pauseIntent)

// Stop service completely
stopService(Intent(context, MusicService::class.java))
```

### Bound Service — Talk Directly to Service

```kotlin
// When you need to call service methods directly (2-way communication)
class ActivityWithBoundService : AppCompatActivity() {
    private var musicService: MusicService? = null
    private var isBound = false

    private val serviceConnection = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName, binder: IBinder) {
            musicService = (binder as MusicService.MusicBinder).getService()
            isBound = true
            // Now you can call service methods directly!
            musicService?.play("url")
        }
        override fun onServiceDisconnected(name: ComponentName) {
            isBound = false
            musicService = null
        }
    }

    override fun onStart() {
        super.onStart()
        val intent = Intent(this, MusicService::class.java)
        bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE)
    }

    override fun onStop() {
        super.onStop()
        if (isBound) {
            unbindService(serviceConnection)
            isBound = false
        }
    }
}
```

---

## 🔑 Key Concepts

| Concept | What It Does | When to Use |
|---------|-------------|------------|
| `Service` | Base class for all services | Long-running tasks |
| `startService()` | Start standalone service | Fire and forget |
| `startForegroundService()` | Required for Foreground on API 26+ | UI-visible long tasks |
| `startForeground()` | Show notification, become foreground | Must call within 5 seconds! |
| `stopSelf()` | Service stops itself | When work is done |
| `onBind()` | Returns IBinder for clients | Bound service communication |
| `bindService()` | Connect to a bound service | Get direct service reference |
| `SERVICE_STICKY` | Restart if killed | Ongoing tasks like music |
| `WorkManager` | Guaranteed background work | Prefer over plain Services |

---

## 💡 Good Example — File Download Service

```kotlin
class FileDownloadService : Service() {
    private val serviceScope = CoroutineScope(Dispatchers.IO + SupervisorJob())

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val fileUrl = intent?.getStringExtra("FILE_URL") ?: return START_NOT_STICKY
        val fileName = intent.getStringExtra("FILE_NAME") ?: "download"
        val notifId = intent.getIntExtra("NOTIF_ID", 1)

        // Show initial notification
        startForeground(notifId, buildProgressNotification(fileName, 0))

        serviceScope.launch {
            try {
                downloadFile(fileUrl, fileName) { progress ->
                    // Update notification progress on IO thread
                    val nm = getSystemService(NotificationManager::class.java)
                    nm.notify(notifId, buildProgressNotification(fileName, progress))
                }
                // Show completion notification
                val nm = getSystemService(NotificationManager::class.java)
                nm.notify(notifId, buildCompleteNotification(fileName))
            } catch (e: Exception) {
                val nm = getSystemService(NotificationManager::class.java)
                nm.notify(notifId, buildErrorNotification(fileName))
            } finally {
                stopForeground(STOP_FOREGROUND_DETACH)
                stopSelf(startId)
            }
        }

        return START_NOT_STICKY  // don't restart if killed
    }

    private fun buildProgressNotification(fileName: String, progress: Int) =
        NotificationCompat.Builder(this, "DOWNLOAD_CHANNEL")
            .setSmallIcon(R.drawable.ic_download)
            .setContentTitle("Downloading $fileName")
            .setProgress(100, progress, progress == 0)
            .setOngoing(true)
            .build()

    private fun buildCompleteNotification(fileName: String) =
        NotificationCompat.Builder(this, "DOWNLOAD_CHANNEL")
            .setSmallIcon(R.drawable.ic_done)
            .setContentTitle("Download Complete!")
            .setContentText(fileName)
            .setAutoCancel(true)
            .build()

    override fun onBind(intent: Intent?) = null

    override fun onDestroy() {
        super.onDestroy()
        serviceScope.cancel()
    }
}
```
