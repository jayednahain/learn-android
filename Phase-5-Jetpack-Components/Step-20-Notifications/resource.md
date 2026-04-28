# Step 20 — Notifications (Major Updates)

---

## 📖 What Is It? (Simple Definition)

Notifications are the little alerts that appear in the status bar at the top of your phone — like "You have a new message!", "Your order is shipped!", "🏃 3km run complete!"

In modern Android, notifications have strict rules:
- You MUST create a **Notification Channel** first (like a category/folder for notifications)
- On Android 13+, you must ask for **POST_NOTIFICATIONS** permission

---

## 🎯 Where Do We Use It?

- Chat messages
- Order/delivery updates
- Download progress
- Reminders and alarms
- Background service status (music player, GPS tracking)

---

## 🔄 Notification Architecture Diagram

```
Create Notification Channel (do once on app start)
              ↓
Build Notification with NotificationCompat.Builder
              ↓
Add: Title, Text, Icon, Style, Actions (buttons)
              ↓
Add: PendingIntent (what happens when tapped)
              ↓
NotificationManagerCompat.notify(id, notification)
              ↓
Appears in Status Bar / Notification Drawer

Android 13+:
User must also GRANT permission first!
```

---

## ☕ Java vs Kotlin — What Changed?

### The Big Change — Notification Channels (since Android 8.0 / API 26)

```java
// Old Java — just build and show, no channels needed (for old Android)
// NotificationCompat.Builder builder = new NotificationCompat.Builder(this)
//     .setSmallIcon(R.drawable.icon)
//     .setContentTitle("Hello")
//     .setContentText("World");
// NotificationManagerCompat.from(this).notify(1, builder.build());
```

```kotlin
// Modern Kotlin — MUST create channel first!

// 1. Create Notification Channel (only needs to run ONCE, safe to call repeatedly)
fun createNotificationChannels(context: Context) {
    // Only needed on Android 8.0+ (API 26+)
    val channels = listOf(
        NotificationChannel(
            "CHAT_CHANNEL",                          // unique ID
            "Chat Messages",                          // user-visible name
            NotificationManager.IMPORTANCE_HIGH       // shows as popup
        ).apply {
            description = "Notifications for new chat messages"
            enableLights(true)
            lightColor = Color.BLUE
            enableVibration(true)
        },
        NotificationChannel(
            "ORDER_CHANNEL",
            "Order Updates",
            NotificationManager.IMPORTANCE_DEFAULT    // no popup, just shows in drawer
        ).apply {
            description = "Notifications for order status updates"
        },
        NotificationChannel(
            "PROMO_CHANNEL",
            "Promotions",
            NotificationManager.IMPORTANCE_LOW        // silent
        ).apply {
            description = "Promotional offers"
        }
    )

    val manager = context.getSystemService(NotificationManager::class.java)
    channels.forEach { manager.createNotificationChannel(it) }
}

// Call in Application class or MainActivity.onCreate():
// createNotificationChannels(this)
```

---

### Building a Basic Notification

```kotlin
fun showChatNotification(context: Context, senderName: String, message: String) {
    // PendingIntent — what opens when user taps the notification
    val openChatIntent = Intent(context, ChatActivity::class.java).apply {
        putExtra("SENDER_NAME", senderName)
        flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP
    }
    val pendingIntent = PendingIntent.getActivity(
        context,
        0,
        openChatIntent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE  // IMMUTABLE required on API 31+
    )

    // Build the notification
    val notification = NotificationCompat.Builder(context, "CHAT_CHANNEL")  // use channel ID!
        .setSmallIcon(R.drawable.ic_chat)                    // required!
        .setContentTitle(senderName)                         // bold title
        .setContentText(message)                             // body text
        .setLargeIcon(BitmapFactory.decodeResource(context.resources, R.drawable.user_avatar))
        .setStyle(NotificationCompat.BigTextStyle().bigText(message))  // show full text when expanded
        .setContentIntent(pendingIntent)                     // open chat when tapped
        .setAutoCancel(true)                                 // dismiss after tap
        .setPriority(NotificationCompat.PRIORITY_HIGH)       // needed for pre-API26 devices
        .build()

    // Show the notification
    NotificationManagerCompat.from(context).notify(1001, notification) // unique ID per notification
}
```

---

### Notification Styles

```kotlin
// 1. BigTextStyle — show long text when expanded
.setStyle(NotificationCompat.BigTextStyle()
    .setBigContentTitle("New Message from Jayed")
    .bigText("Hey! Are you coming to the meeting tomorrow at 10am? Let me know!"))

// 2. InboxStyle — list of messages (e.g., multiple emails)
.setStyle(NotificationCompat.InboxStyle()
    .setBigContentTitle("5 new messages")
    .addLine("Jayed: When's the meeting?")
    .addLine("Fahim: Sent you the designs")
    .addLine("Boss: Call me today")
    .setSummaryText("+2 more"))

// 3. BigPictureStyle — show image when expanded
.setStyle(NotificationCompat.BigPictureStyle()
    .bigPicture(orderImageBitmap)
    .bigLargeIcon(null as Bitmap?))  // hide small icon when expanded

// 4. MessagingStyle — perfect for chat (shows conversation thread!)
val person = Person.Builder().setName("Jayed").build()
.setStyle(NotificationCompat.MessagingStyle(person)
    .addMessage("Ready for the meeting?", System.currentTimeMillis() - 2000, person)
    .addMessage("Yes, on my way!", System.currentTimeMillis(), null)  // null = you (the user)
)
```

---

### Notification Actions (Buttons)

```kotlin
// Quick Reply action — reply without opening app
val replyAction = NotificationCompat.Action.Builder(
    R.drawable.ic_reply,
    "Reply",
    remoteInputPendingIntent  // handle the reply
).addRemoteInput(
    RemoteInput.Builder("REMOTE_INPUT_KEY")
        .setLabel("Write reply...")
        .build()
).build()

// Dismiss action
val dismissPendingIntent = PendingIntent.getBroadcast(
    context, 0,
    Intent(context, NotificationDismissReceiver::class.java),
    PendingIntent.FLAG_IMMUTABLE
)

notification = NotificationCompat.Builder(context, "CHAT_CHANNEL")
    ...
    .addAction(R.drawable.ic_reply, "Reply", replyPendingIntent)
    .addAction(R.drawable.ic_dismiss, "Dismiss", dismissPendingIntent)
    .build()
```

---

### Android 13+ — Request POST_NOTIFICATIONS Permission

```kotlin
// Android 13 (API 33) adds runtime permission for notifications!
private val requestNotificationPermission =
    registerForActivityResult(ActivityResultContracts.RequestPermission()) { isGranted ->
        if (isGranted) {
            showNotification()
        } else {
            // Show explanation to user
            Snackbar.make(binding.root, "Notifications disabled", Snackbar.LENGTH_LONG)
                .setAction("Settings") { openAppSettings() }
                .show()
        }
    }

// In Activity onCreate or when about to show first notification:
fun checkNotificationPermission() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        when {
            ContextCompat.checkSelfPermission(this, Manifest.permission.POST_NOTIFICATIONS)
                == PackageManager.PERMISSION_GRANTED -> showNotification()

            shouldShowRequestPermissionRationale(Manifest.permission.POST_NOTIFICATIONS) ->
                showPermissionRationaleDialog()

            else -> requestNotificationPermission.launch(Manifest.permission.POST_NOTIFICATIONS)
        }
    } else {
        showNotification()  // Permission not needed before API 33
    }
}
```

---

### Foreground Service Notification (Must-Have for Long Running Tasks)

```kotlin
// Music player, GPS tracking, file download — must show persistent notification
class MusicPlayerService : Service() {

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val notification = NotificationCompat.Builder(this, "MUSIC_CHANNEL")
            .setSmallIcon(R.drawable.ic_music)
            .setContentTitle("Playing: Favorite Song")
            .setContentText("Artist Name")
            .addAction(R.drawable.ic_pause, "Pause", pausePendingIntent)
            .addAction(R.drawable.ic_next, "Next", nextPendingIntent)
            .setStyle(androidx.media.app.NotificationCompat.MediaStyle()
                .setShowActionsInCompactView(0, 1))
            .setOngoing(true)  // can't be dismissed by user
            .build()

        // REQUIRED for Foreground Service — must call within 5 seconds!
        startForeground(1, notification)

        return START_STICKY
    }
}
```

---

## 🔑 Key Concepts

| Concept | Required Since | Purpose |
|---------|--------------|---------|
| NotificationChannel | Android 8.0 (API 26) | Category for notifications |
| IMPORTANCE_HIGH | — | Shows as popup heads-up |
| PendingIntent | Always | What happens on tap |
| FLAG_IMMUTABLE | Android 12 (API 31) | Security requirement |
| POST_NOTIFICATIONS | Android 13 (API 33) | Runtime permission for notifications |
| startForeground() | Always for services | Required for long-running services |
| setAutoCancel(true) | — | Dismiss on tap |
| notify(id, notif) | Always | Show the notification |

---

## 💡 Good Example — Order Status Notification

```kotlin
object OrderNotificationHelper {

    fun showOrderUpdate(context: Context, orderId: String, status: String) {
        val (title, icon) = when (status) {
            "confirmed" -> "Order Confirmed! 🎉" to R.drawable.ic_check
            "shipped"   -> "Your order is on the way! 📦" to R.drawable.ic_truck
            "delivered" -> "Order Delivered! ✅" to R.drawable.ic_delivered
            else        -> "Order Update" to R.drawable.ic_order
        }

        val intent = Intent(context, OrderDetailActivity::class.java).apply {
            putExtra("ORDER_ID", orderId)
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP
        }
        val pi = PendingIntent.getActivity(context, orderId.hashCode(), intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE)

        val notification = NotificationCompat.Builder(context, "ORDER_CHANNEL")
            .setSmallIcon(icon)
            .setContentTitle(title)
            .setContentText("Order #$orderId — $status")
            .setStyle(NotificationCompat.BigTextStyle()
                .bigText("Your order #$orderId status has been updated to: $status"))
            .setContentIntent(pi)
            .setAutoCancel(true)
            .setPriority(NotificationCompat.PRIORITY_DEFAULT)
            .build()

        NotificationManagerCompat.from(context).notify(orderId.hashCode(), notification)
    }
}
```
