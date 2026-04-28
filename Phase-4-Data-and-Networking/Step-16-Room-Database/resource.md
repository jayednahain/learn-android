# Step 16 — Room Database (Replaces SQLiteOpenHelper)

---

## 📖 What Is It? (Simple Definition)

Imagine a perfectly organized filing cabinet on your phone. **Room** is that filing cabinet — it lets you save data on the device so your app works even offline.

Room is built on top of SQLite (the same database you used in Java), but it handles all the boring, error-prone SQL boilerplate for you. No more writing raw SQL strings by hand!

---

## 🎯 Where Do We Use It?

- Save user data for offline access
- Cache API responses (so app loads instantly next time)
- Store user preferences data (favorites, history, cart)
- Any data that needs to persist after app is closed

---

## 🔄 Room Architecture Diagram

```
Your Kotlin Code
      │
      ▼
┌──────────────────────────────────────┐
│         Room Database                │
│                                      │
│  @Entity       ← defines a TABLE    │
│  @Dao          ← defines QUERIES    │
│  @Database     ← the database file  │
└──────────────────────────────────────┘
      │
      ▼
     SQLite file on device
     (/data/data/com.yourapp/databases/app.db)
```

---

## ☕ Java vs Kotlin — What Changed?

### Old Java Way — SQLiteOpenHelper (painful!)

```java
// Java — 100+ lines of raw SQL, error-prone strings, manual cursor management
public class DatabaseHelper extends SQLiteOpenHelper {
    private static final String CREATE_USER_TABLE =
        "CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT, email TEXT)";

    public User getUser(int id) {
        SQLiteDatabase db = getReadableDatabase();
        Cursor cursor = db.rawQuery(
            "SELECT * FROM users WHERE id = ?", new String[]{String.valueOf(id)}
        );
        User user = null;
        if (cursor.moveToFirst()) {
            // Manually read each column by index — easy to make mistakes!
            user = new User(
                cursor.getInt(cursor.getColumnIndex("id")),
                cursor.getString(cursor.getColumnIndex("name")),
                cursor.getString(cursor.getColumnIndex("email"))
            );
        }
        cursor.close();
        return user;
    }
    // 70+ more lines... just for basic CRUD!
}
```

### New Way — Room (clean annotations, 20 lines total!)

```kotlin
// 1. @Entity — defines a database TABLE
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    @ColumnInfo(name = "full_name") val name: String,
    val email: String,
    val bio: String = "",
    @ColumnInfo(name = "created_at") val createdAt: Long = System.currentTimeMillis()
)
```

```kotlin
// 2. @Dao — Data Access Object (defines queries)
@Dao
interface UserDao {
    // INSERT
    @Insert(onConflict = OnConflictStrategy.REPLACE) // replace if same PK
    suspend fun insert(user: UserEntity)

    @Insert
    suspend fun insertAll(users: List<UserEntity>)

    // UPDATE
    @Update
    suspend fun update(user: UserEntity)

    // DELETE
    @Delete
    suspend fun delete(user: UserEntity)

    @Query("DELETE FROM users WHERE id = :userId")
    suspend fun deleteById(userId: Int)

    @Query("DELETE FROM users")
    suspend fun deleteAll()

    // QUERY — returns Flow! UI auto-updates when DB changes!
    @Query("SELECT * FROM users ORDER BY full_name ASC")
    fun getAllUsers(): Flow<List<UserEntity>>

    @Query("SELECT * FROM users WHERE id = :userId")
    suspend fun getUserById(userId: Int): UserEntity?

    @Query("SELECT * FROM users WHERE full_name LIKE '%' || :query || '%' OR email LIKE '%' || :query || '%'")
    fun searchUsers(query: String): Flow<List<UserEntity>>

    @Query("SELECT COUNT(*) FROM users")
    suspend fun getUserCount(): Int
}
```

```kotlin
// 3. @Database — the database class
@Database(
    entities = [UserEntity::class, ProductEntity::class],  // all tables
    version = 1,                                            // increment when schema changes
    exportSchema = false
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
    abstract fun productDao(): ProductDao

    companion object {
        @Volatile private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database"
                ).build().also { INSTANCE = it }
            }
        }
    }
}
// Note: With Hilt (Step 24), you won't need the singleton pattern above — Hilt handles it!
```

---

### Room + Coroutines + Flow

```kotlin
// DAO queries with Flow = REACTIVE database!
// When data in DB changes → Flow automatically emits new values → UI auto-updates!

@Query("SELECT * FROM users")
fun getAllUsers(): Flow<List<UserEntity>>  // suspend NOT needed for Flow

@Query("SELECT * FROM users WHERE id = :id")
suspend fun getUser(id: Int): UserEntity?  // suspend for single value
```

```kotlin
// In Repository:
class UserRepositoryImpl(private val userDao: UserDao) : UserRepository {

    // This Flow stays "alive" and emits whenever DB data changes!
    override fun getUsers(): Flow<List<User>> =
        userDao.getAllUsers().map { entities -> entities.map { it.toUser() } }

    override suspend fun saveUser(user: User) {
        userDao.insert(user.toEntity())
        // getAllUsers() Flow emits automatically — no manual refresh needed!
    }
}
```

---

### Type Converters — Store Complex Types

```kotlin
// Room can only store: Int, Long, Double, Float, String, Boolean
// For custom types (Date, List<String>, Enum), use TypeConverter

class Converters {
    @TypeConverter
    fun fromTimestamp(value: Long?): Date? = value?.let { Date(it) }

    @TypeConverter
    fun dateToTimestamp(date: Date?): Long? = date?.time

    @TypeConverter
    fun fromStringList(value: String?): List<String> =
        value?.split(",") ?: emptyList()

    @TypeConverter
    fun toStringList(list: List<String>): String = list.joinToString(",")
}

// Register in @Database:
@Database(entities = [...], version = 1)
@TypeConverters(Converters::class)  // add this!
abstract class AppDatabase : RoomDatabase() { ... }
```

---

### Database Migrations — Handling Schema Changes

```kotlin
// Version 1 → Version 2: added "phone_number" column to users table
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("ALTER TABLE users ADD COLUMN phone_number TEXT DEFAULT '' NOT NULL")
    }
}

// Register in database builder:
Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
    .addMigrations(MIGRATION_1_2)
    .build()
```

> 💡 Always increment `version` in `@Database` when you change the table structure! If you forget migrations, Room will crash on existing installs (it won't wipe user data silently).

---

## 🔑 Key Concepts

| Annotation | What It Does | Analogy |
|-----------|-------------|---------|
| `@Entity` | Defines a table | Excel sheet template |
| `@PrimaryKey` | Unique row identifier | Row number |
| `@ColumnInfo` | Rename column | Column header |
| `@Dao` | Defines SQL queries | SQL query interface |
| `@Insert` | Insert row | Add row to spreadsheet |
| `@Update` | Update row | Edit a cell |
| `@Delete` | Delete row | Delete a row |
| `@Query` | Custom SQL | Any SQL you write |
| `@Database` | The database itself | The whole Excel file |
| `Flow<T>` return | Reactive query | Auto-refreshes on change |
| `suspend` return | One-time query | Returns once |
| `@TypeConverter` | Custom type storage | Convert object to storable string |
| `Migration` | Schema update | Alter table without losing data |

---

## 💡 Good Example — Shopping Cart with Room

```kotlin
@Entity(tableName = "cart_items")
data class CartItemEntity(
    @PrimaryKey val productId: Int,
    val productName: String,
    val price: Double,
    val quantity: Int = 1,
    val imageUrl: String
)

@Dao
interface CartDao {
    @Query("SELECT * FROM cart_items")
    fun getCartItems(): Flow<List<CartItemEntity>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun addItem(item: CartItemEntity)

    @Query("UPDATE cart_items SET quantity = quantity + 1 WHERE productId = :productId")
    suspend fun increaseQuantity(productId: Int)

    @Query("UPDATE cart_items SET quantity = quantity - 1 WHERE productId = :productId AND quantity > 1")
    suspend fun decreaseQuantity(productId: Int)

    @Query("DELETE FROM cart_items WHERE productId = :productId")
    suspend fun removeItem(productId: Int)

    @Query("DELETE FROM cart_items")
    suspend fun clearCart()

    @Query("SELECT SUM(price * quantity) FROM cart_items")
    fun getTotalPrice(): Flow<Double?>  // Flow so UI updates on any change

    @Query("SELECT COUNT(*) FROM cart_items")
    fun getCartCount(): Flow<Int>
}

// ViewModel using Cart DAO via Repository
class CartViewModel(private val cartRepository: CartRepository) : ViewModel() {
    val cartItems: StateFlow<List<CartItem>> = cartRepository.getCartItems()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    val totalPrice: StateFlow<Double> = cartRepository.getTotalPrice()
        .map { it ?: 0.0 }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), 0.0)

    fun addToCart(product: Product) {
        viewModelScope.launch { cartRepository.addItem(product.toCartItem()) }
    }

    fun checkout() {
        viewModelScope.launch { cartRepository.clearCart() }
    }
}
```
