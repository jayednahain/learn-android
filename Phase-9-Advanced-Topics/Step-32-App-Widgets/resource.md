# Step 32 — App Widgets

---

## 📖 What Is It? (Simple Explanation)

An **App Widget** is a small view of your app that lives on the **home screen** of the phone 📱. You've seen them — the weather widget that shows temperature without opening the app, the clock widget, the music widget with play/pause buttons.

Widgets let users see information or do small actions **without opening your app**. They are like a tiny window into your app sitting on the home screen.

---

## 🧭 Where Do We Use It?

- Weather apps (show temperature on home screen)
- Music apps (show now-playing + controls)
- Fitness apps (show step count)
- Calendar apps (show upcoming events)
- Notes apps (quick note widget)
- Any app that has data users want to glance at quickly

---

## 🗺️ Workflow / How It Works

```
User adds widget to home screen
            │
            ▼
Android calls AppWidgetProvider.onUpdate()
            │
            ▼
Your code builds RemoteViews (limited UI)
            │
            ▼
AppWidgetManager.updateAppWidget() pushes the view
            │
            ▼
Widget appears on home screen

Periodic Update:
Android calls onUpdate() every N minutes (you define)
→ You refresh the data → Widget updates
```

---

## ☕ Java vs Kotlin — What Changed?

### Old Way (Java — tedious boilerplate)

```java
// Java AppWidgetProvider
public class MyWidget extends AppWidgetProvider {
    @Override
    public void onUpdate(Context context, AppWidgetManager manager, int[] widgetIds) {
        for (int widgetId : widgetIds) {
            RemoteViews views = new RemoteViews(context.getPackageName(), R.layout.widget_layout);
            views.setTextViewText(R.id.tvCount, "Count: " + getCount());
            
            Intent intent = new Intent(context, MyWidget.class);
            intent.setAction(ACTION_INCREMENT);
            PendingIntent pi = PendingIntent.getBroadcast(context, 0, intent,
                PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_IMMUTABLE);
            views.setOnClickPendingIntent(R.id.btnIncrement, pi);
            
            manager.updateAppWidget(widgetId, views);
        }
    }
}
```

### New Way (Kotlin — same concept, cleaner syntax + Glance for Compose)

```kotlin
// Kotlin AppWidgetProvider
class MyWidget : AppWidgetProvider() {
    override fun onUpdate(context: Context, manager: AppWidgetManager, widgetIds: IntArray) {
        widgetIds.forEach { widgetId ->
            updateWidget(context, manager, widgetId)
        }
    }
}

fun updateWidget(context: Context, manager: AppWidgetManager, widgetId: Int) {
    val views = RemoteViews(context.packageName, R.layout.widget_layout).apply {
        setTextViewText(R.id.tvCount, "Count: ${getCount()}")
        setOnClickPendingIntent(R.id.btnIncrement, createPendingIntent(context))
    }
    manager.updateAppWidget(widgetId, views)
}
```

**Glance (Compose-based widgets) — the modern way:**

```kotlin
class MyGlanceWidget : GlanceAppWidget() {
    @Composable
    override fun Content() {
        Column {
            Text("Hello from widget!")
            Button("Click me", onClick = actionRunCallback<MyAction>())
        }
    }
}
```

> ✅ Glance lets you write widget UI using Compose-like syntax — no more `RemoteViews` with IDs!

---

## 🔑 Key Concepts

---

### 1. `AppWidgetProvider` — The Brain of the Widget

`AppWidgetProvider` is a `BroadcastReceiver` subclass. Android calls its methods when the widget needs to be updated, added, or removed.

```kotlin
class CounterWidget : AppWidgetProvider() {

    // Called on initial placement AND at the configured update interval
    override fun onUpdate(context: Context, manager: AppWidgetManager, ids: IntArray) {
        ids.forEach { updateWidget(context, manager, it) }
    }

    // Widget added to home screen
    override fun onEnabled(context: Context) { }

    // Last widget instance removed
    override fun onDisabled(context: Context) { }

    // Specific widget instance deleted
    override fun onDeleted(context: Context, appWidgetIds: IntArray) { }
}
```

---

### 2. `RemoteViews` — The Widget's UI (Limited View System)

Widgets cannot use all Android views. They can only use a **limited set** specified by Android:

| Allowed | Not Allowed |
|---|---|
| `TextView` | `RecyclerView` (use `ListView` in widget) |
| `Button` | `EditText` |
| `ImageView` | `WebView` |
| `LinearLayout`, `FrameLayout` | Custom views |
| `ProgressBar` | `Compose` (unless using Glance) |

```kotlin
val views = RemoteViews(context.packageName, R.layout.widget_layout)
views.setTextViewText(R.id.tvTitle, "Hello Widget")
views.setImageViewResource(R.id.ivIcon, R.drawable.ic_star)
views.setInt(R.id.container, "setBackgroundColor", Color.WHITE)
```

---

### 3. Widget Configuration in `AndroidManifest.xml`

```xml
<receiver
    android:name=".CounterWidget"
    android:exported="true">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE"/>
    </intent-filter>
    <meta-data
        android:name="android.appwidget.provider"
        android:resource="@xml/counter_widget_info"/>
</receiver>
```

---

### 4. Widget Info XML (`res/xml/counter_widget_info.xml`)

```xml
<appwidget-provider
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:minWidth="110dp"
    android:minHeight="40dp"
    android:updatePeriodMillis="1800000"  <!-- update every 30 minutes -->
    android:initialLayout="@layout/widget_layout"
    android:resizeMode="horizontal|vertical"
    android:widgetCategory="home_screen"
    android:configure="com.example.WidgetConfigActivity"/> <!-- optional config screen -->
```

---

### 5. Making a Button Click Work in Widget

Widgets use `PendingIntent` for click actions (same as notifications).

```kotlin
// Create a broadcast PendingIntent
private fun createIncrementIntent(context: Context, widgetId: Int): PendingIntent {
    val intent = Intent(context, CounterWidget::class.java).apply {
        action = ACTION_INCREMENT
        putExtra(AppWidgetManager.EXTRA_APPWIDGET_ID, widgetId)
    }
    return PendingIntent.getBroadcast(
        context, widgetId, intent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
    )
}

// Attach to a button
views.setOnClickPendingIntent(R.id.btnIncrement, createIncrementIntent(context, widgetId))

// Handle the broadcast
override fun onReceive(context: Context, intent: Intent) {
    super.onReceive(context, intent)
    if (intent.action == ACTION_INCREMENT) {
        val widgetId = intent.getIntExtra(AppWidgetManager.EXTRA_APPWIDGET_ID, -1)
        // update count, then refresh widget
        val manager = AppWidgetManager.getInstance(context)
        updateWidget(context, manager, widgetId)
    }
}
```

---

### 6. Glance — Modern Compose-Based Widgets

**Glance** is Google's modern library for building widgets using a Compose-like API.

```kotlin
// build.gradle
implementation("androidx.glance:glance-appwidget:1.1.0")
```

```kotlin
class CounterGlanceWidget : GlanceAppWidget() {

    override val stateDefinition = PreferencesGlanceStateDefinition

    @Composable
    override fun Content() {
        val prefs = currentState<Preferences>()
        val count = prefs[intPreferencesKey("count")] ?: 0

        Column(
            modifier = GlanceModifier
                .fillMaxSize()
                .background(Color.White)
                .padding(16.dp),
            verticalAlignment = Alignment.CenterVertically,
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            Text(
                text = "Count: $count",
                style = TextStyle(fontSize = 20.sp, fontWeight = FontWeight.Bold)
            )
            Spacer(GlanceModifier.height(8.dp))
            Button(
                text = "Increment",
                onClick = actionRunCallback<IncrementAction>()
            )
        }
    }
}

// Action class
class IncrementAction : ActionCallback {
    override suspend fun onAction(
        context: Context,
        glanceId: GlanceId,
        parameters: ActionParameters
    ) {
        val prefs = getAppWidgetState(context, PreferencesGlanceStateDefinition, glanceId)
        val currentCount = prefs[intPreferencesKey("count")] ?: 0
        updateAppWidgetState(context, glanceId) { state ->
            state[intPreferencesKey("count")] = currentCount + 1
        }
        CounterGlanceWidget().update(context, glanceId)
    }
}

// Receiver
class CounterWidgetReceiver : GlanceAppWidgetReceiver() {
    override val glanceAppWidget = CounterGlanceWidget()
}
```

---

## 💡 Full Practical Example — Counter Widget (Classic)

**`res/layout/widget_counter.xml`**
```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:gravity="center"
    android:background="#FFFFFF"
    android:padding="8dp"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView android:id="@+id/tvCount"
        android:text="0"
        android:textSize="32sp"
        android:textColor="#000000"/>

    <Button android:id="@+id/btnAdd"
        android:text="+"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"/>
</LinearLayout>
```

**`CounterWidget.kt`**
```kotlin
class CounterWidget : AppWidgetProvider() {

    companion object {
        const val ACTION_INCREMENT = "com.example.INCREMENT"
        private var count = 0

        fun updateWidget(context: Context, manager: AppWidgetManager, id: Int) {
            val views = RemoteViews(context.packageName, R.layout.widget_counter).apply {
                setTextViewText(R.id.tvCount, count.toString())
                val intent = Intent(context, CounterWidget::class.java).apply {
                    action = ACTION_INCREMENT
                    putExtra(AppWidgetManager.EXTRA_APPWIDGET_ID, id)
                }
                val pi = PendingIntent.getBroadcast(
                    context, id, intent,
                    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
                )
                setOnClickPendingIntent(R.id.btnAdd, pi)
            }
            manager.updateAppWidget(id, views)
        }
    }

    override fun onUpdate(context: Context, manager: AppWidgetManager, ids: IntArray) {
        ids.forEach { updateWidget(context, manager, it) }
    }

    override fun onReceive(context: Context, intent: Intent) {
        super.onReceive(context, intent)
        if (intent.action == ACTION_INCREMENT) {
            count++
            val id = intent.getIntExtra(AppWidgetManager.EXTRA_APPWIDGET_ID, -1)
            if (id != -1) updateWidget(context, AppWidgetManager.getInstance(context), id)
        }
    }
}
```

---

## 📊 Quick Comparison Summary

| Concept | Classic Widget | Glance (Modern) |
|---|---|---|
| UI building | `RemoteViews` + XML layout | Compose-like `@Composable` |
| Click actions | `PendingIntent` broadcasts | `actionRunCallback<>()` |
| State | Manual storage/SharedPrefs | `PreferencesGlanceStateDefinition` |
| Complexity | Medium — boilerplate heavy | Lower — declarative |
| Compose syntax | ❌ | ✅ (Glance-specific composables) |

| Old (Java) | New (Kotlin) |
|---|---|
| Verbose PendingIntent setup | `.apply {}` block keeps it clean |
| Static inner class for actions | `ActionCallback` interface |
| Same code, just Java syntax | Kotlin extension functions, lambdas |
