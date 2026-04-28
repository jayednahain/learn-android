# Step 22 — BroadcastReceiver (Updated)

---

## 📖 What Is It? (Simple Definition)

Think of a **BroadcastReceiver** as a radio receiver. The Android system (and other apps) constantly "broadcast" system announcements like:
- 📶 "Internet just connected!"  
- 🔋 "Battery is low!"
- 📱 "Phone just booted!"
- ✈️ "Airplane mode turned on!"

Your app can have a receiver that **listens** to these announcements and reacts to them.

---

## 🎯 Where Do We Use It?

- React to system events (network change, device boot, battery low)
- Receive intents from other apps or your own services
- Handle notification action buttons
- Scheduled alarms (from AlarmManager)

> ⚠️ Many implicit broadcasts are now **restricted on Android 8+** — apps can no longer freely listen to system-wide broadcasts in the background. Use `JobScheduler` / `WorkManager` instead.

---

## 🔄 Broadcast Flow Diagram

```
Android System or Your App
          ↓ sends broadcast
   Intent("android.net.conn.CONNECTIVITY_CHANGE")
          ↓
   Android delivers to all registered receivers
          ↓
   Your BroadcastReceiver.onReceive() is called
          ↓
   Your app reacts (show notification, sync data, etc.)

Types:
  Static (Manifest)  → Works even when app is not running  (limited in API 26+)
  Dynamic (code)     → Works while app/component is alive
  Ordered            → Multiple receivers in priority order
```

---

## ☕ Java vs Kotlin — What Changed?

### Old Java Way

```java
// Java BroadcastReceiver
public class BootReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        if (Intent.ACTION_BOOT_COMPLETED.equals(intent.getAction())) {
            // App booted — reschedule alarms
        }
    }
}
// Register in Manifest...
```

### Kotlin — Static Receiver (Manifest-declared)

```kotlin
// BroadcastReceiver in Kotlin — compact!
class BootReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        when (intent.action) {
            Intent.ACTION_BOOT_COMPLETED -> {
                // Phone rebooted — restart any needed services or reschedule work
                WorkManager.getInstance(context).enqueue(
                    OneTimeWorkRequestBuilder<SyncWorker>().build()
                )
            }
        }
    }
}
```

```xml
<!-- AndroidManifest.xml — static receiver -->
<receiver
    android:name=".BootReceiver"
    android:exported="false"> <!-- false = only receive from system, not other apps -->
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

---

### Dynamic Receiver — Register in Code (Modern, Preferred)

```kotlin
class NetworkMonitorFragment : Fragment() {
    private var networkReceiver: BroadcastReceiver? = null

    override fun onStart() {
        super.onStart()
        // Register receiver when fragment becomes visible
        networkReceiver = object : BroadcastReceiver() {
            override fun onReceive(context: Context, intent: Intent) {
                val isConnected = isNetworkConnected(context)
                if (isConnected) {
                    binding.networkBanner.isVisible = false
                    viewModel.retryFetch()
                } else {
                    binding.networkBanner.isVisible = true
                    binding.networkBanner.text = "No internet connection"
                }
            }
        }

        val filter = IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION)
        // Android 13+ requires specifying receiver exported status
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            requireContext().registerReceiver(networkReceiver, filter, Context.RECEIVER_NOT_EXPORTED)
        } else {
            requireContext().registerReceiver(networkReceiver, filter)
        }
    }

    override fun onStop() {
        super.onStop()
        // ALWAYS unregister to avoid memory leaks!
        networkReceiver?.let {
            requireContext().unregisterReceiver(it)
        }
        networkReceiver = null
    }

    private fun isNetworkConnected(context: Context): Boolean {
        val cm = context.getSystemService(ConnectivityManager::class.java)
        val net = cm.activeNetwork ?: return false
        val caps = cm.getNetworkCapabilities(net) ?: return false
        return caps.hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
    }
}
```

---

### Modern Network Monitoring — Use `ConnectivityManager.NetworkCallback` Instead of Broadcast!

```kotlin
// Since Android 8+, the CONNECTIVITY_CHANGE broadcast is restricted in background.
// Use NetworkCallback instead — more reliable, works in background too.

class NetworkMonitor(private val context: Context) {

    private val _isConnected = MutableStateFlow(true)
    val isConnected: StateFlow<Boolean> = _isConnected.asStateFlow()

    private val connectivityManager =
        context.getSystemService(ConnectivityManager::class.java)

    private val networkCallback = object : ConnectivityManager.NetworkCallback() {
        override fun onAvailable(network: Network) {
            _isConnected.value = true
        }
        override fun onLost(network: Network) {
            _isConnected.value = false
        }
    }

    fun startMonitoring() {
        val request = NetworkRequest.Builder()
            .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
            .build()
        connectivityManager.registerNetworkCallback(request, networkCallback)
    }

    fun stopMonitoring() {
        connectivityManager.unregisterNetworkCallback(networkCallback)
    }
}

// In ViewModel:
class MainViewModel(private val networkMonitor: NetworkMonitor) : ViewModel() {
    val isOnline: StateFlow<Boolean> = networkMonitor.isConnected

    init { networkMonitor.startMonitoring() }
    override fun onCleared() { networkMonitor.stopMonitoring() }
}
```

---

### `LocalBroadcastManager` — DEPRECATED → Use Flow Instead

```kotlin
// Old way: LocalBroadcastManager.getInstance(context).sendBroadcast(intent)
// DEPRECATED — replaced by SharedFlow!

// OLD (don't use):
// LocalBroadcastManager.getInstance(context).sendBroadcast(
//     Intent("ACTION_DATA_DOWNLOADED")
// )

// NEW — use SharedFlow in a singleton or a ViewModel scope:
object AppEventBus {
    private val _events = MutableSharedFlow<AppEvent>(extraBufferCapacity = 10)
    val events: SharedFlow<AppEvent> = _events.asSharedFlow()

    suspend fun emit(event: AppEvent) = _events.emit(event)
}

sealed class AppEvent {
    object DataSynced : AppEvent()
    data class DownloadComplete(val fileName: String) : AppEvent()
    data class ErrorOccurred(val message: String) : AppEvent()
}

// Emit from anywhere (e.g., from a Service):
GlobalScope.launch { AppEventBus.emit(AppEvent.DataSynced) }

// Listen in ViewModel:
class HomeViewModel : ViewModel() {
    init {
        viewModelScope.launch {
            AppEventBus.events.collect { event ->
                when (event) {
                    AppEvent.DataSynced -> refreshData()
                    is AppEvent.DownloadComplete -> showDownloadSuccess(event.fileName)
                    is AppEvent.ErrorOccurred -> showError(event.message)
                }
            }
        }
    }
}
```

---

### Ordered Broadcasts — Message Passing in Priority Order

```kotlin
// Send ordered broadcast — receivers get it in priority order, can abort or modify
val intent = Intent("com.example.ORDERED_ACTION")
sendOrderedBroadcast(intent, null)

// High priority receiver can abort delivery to lower priority receivers
class HighPriorityReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        // Do work...
        // Optionally abort so lower-priority receivers don't get it:
        abortBroadcast()
    }
}
```

---

## 🔑 Key Concepts

| Concept | Static vs Dynamic | Modern Alternative |
|---------|-----------------|-------------------|
| System broadcasts | Static (Manifest) | WorkManager constraints |
| Network change | Dynamic (limited in background) | `NetworkCallback` |
| App-internal events | `LocalBroadcastManager` (deprecated) | `SharedFlow` |
| Notification actions | PendingIntent → BroadcastReceiver | Still valid! |
| BOOT_COMPLETED | Static (Manifest) | Still valid for rescheduling |
| Ordered broadcasts | `sendOrderedBroadcast()` | Still valid |

---

## 💡 Good Example — Notification Action Handler

```kotlin
// Handle "Mark as Read" button in notification without opening the app
class MarkReadReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val messageId = intent.getIntExtra("MESSAGE_ID", -1)
        if (messageId == -1) return

        // Use coroutines in a goAsync() block for DB operation
        val pendingResult = goAsync()  // tells Android to wait a bit more
        CoroutineScope(Dispatchers.IO).launch {
            try {
                // Mark message as read in database
                AppDatabase.getInstance(context).messageDao().markAsRead(messageId)
                // Cancel the notification
                NotificationManagerCompat.from(context).cancel(messageId)
            } finally {
                pendingResult.finish()  // tell Android we're done
            }
        }
    }
}

// Create PendingIntent for the action button in notification:
val markReadIntent = Intent(context, MarkReadReceiver::class.java).apply {
    putExtra("MESSAGE_ID", message.id)
}
val markReadPI = PendingIntent.getBroadcast(
    context, message.id,
    markReadIntent,
    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
)

NotificationCompat.Builder(context, "CHAT_CHANNEL")
    .addAction(R.drawable.ic_done, "Mark as Read", markReadPI)
    ...
```
