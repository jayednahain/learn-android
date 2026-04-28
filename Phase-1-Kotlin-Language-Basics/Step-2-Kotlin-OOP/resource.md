# Step 2 — Kotlin OOP (Object-Oriented Programming)

---

## 📖 What Is It? (Simple Definition)

OOP is like building with LEGO bricks. Each brick is a **class** — a blueprint that tells you what shape and color it is. When you actually pick up a brick and use it, that's an **object**.

Kotlin has OOP just like Java, but with a lot fewer lines! Things that needed 50 lines in Java now need 10 in Kotlin.

---

## 🎯 Where Do We Use It?

You use OOP **everywhere** — every Activity, Fragment, ViewModel, and data model is a class. Understanding Kotlin classes is the foundation for writing Android apps properly.

---

## 🔄 Class Hierarchy Diagram

```
Any (Kotlin's top-level parent — like Object in Java)
  │
  ├── open class Animal          ← can be inherited
  │       └── class Dog : Animal ← Dog extends Animal
  │
  ├── abstract class Shape       ← must be implemented
  │       └── class Circle : Shape
  │
  ├── interface Flyable           ← behavior contract
  │       └── class Bird : Flyable
  │
  ├── data class User             ← for holding data
  ├── object Singleton            ← single instance
  └── companion object            ← like Java static
```

---

## ☕ Java vs Kotlin — What Changed?

### 1. Regular Class & Constructor
```java
// Java — constructor is a separate block, lots of boilerplate
public class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() { return name; }
    public int getAge() { return age; }
}
// Usage:
Person p = new Person("Jayed", 28);
```
```kotlin
// Kotlin — everything in ONE line! No getters/setters needed.
class Person(val name: String, val age: Int)

// Usage:
val p = Person("Jayed", 28)  // no "new" keyword!
println(p.name)               // direct access, no getter
println(p.age)
```
> 💡 Kotlin automatically creates getters for `val` and getters+setters for `var` in the constructor.

---

### 2. `data class` — The Game Changer
```java
// Java — you need 50+ lines: constructor, getters, equals(), hashCode(), toString()
public class User {
    private String name;
    private String email;
    // ... 40+ more lines of boilerplate
}
```
```kotlin
// Kotlin data class — just 1 line! Gets everything automatically.
data class User(val name: String, val email: String)

val u1 = User("Jayed", "jayed@gmail.com")
val u2 = u1.copy(email = "new@gmail.com") // copy with one field changed
println(u1)           // User(name=Jayed, email=jayed@gmail.com) — auto toString!
println(u1 == u2)     // false — auto equals() by value!
```
> 💡 `data class` automatically gives you: `toString()`, `equals()`, `hashCode()`, and `copy()`. In Java you had to write all of these yourself!

---

### 3. Inheritance — The `open` Keyword
```java
// Java — all classes can be extended by default
public class Animal { ... }
public class Dog extends Animal { ... }
```
```kotlin
// Kotlin — classes are CLOSED (final) by default!
// You must add "open" to allow inheritance
open class Animal(val name: String) {
    open fun speak() = println("...") // open = can be overridden
}

class Dog(name: String) : Animal(name) {  // : means extends
    override fun speak() = println("Woof!")
}

val dog = Dog("Rex")
dog.speak() // Woof!
```
> 💡 In Java, you had to remember to write `final` to PREVENT extending. In Kotlin, it's the opposite — classes are final by default, you must write `open` to ALLOW extending.

---

### 4. `interface` and `abstract class`
```kotlin
// Interface — defines WHAT to do, not HOW
interface Swimmer {
    fun swim()                      // must implement
    fun breathe() = println("Breathe air") // has default implementation
}

// Abstract class — can have both abstract and real methods
abstract class Vehicle(val brand: String) {
    abstract fun move()          // must override
    fun describe() = println("I am a $brand") // ready to use
}

class Car(brand: String) : Vehicle(brand), Swimmer {
    override fun move() = println("Drive on road")
    override fun swim() = println("Cars don't swim!")
}
```

---

### 5. `object` — Singleton (One and Only One Instance)
```java
// Java Singleton — 10+ lines with double-checked locking
public class DatabaseHelper {
    private static DatabaseHelper instance;
    private DatabaseHelper() {}
    public static synchronized DatabaseHelper getInstance() {
        if (instance == null) instance = new DatabaseHelper();
        return instance;
    }
}
```
```kotlin
// Kotlin — just use "object"! Automatic thread-safe singleton.
object DatabaseHelper {
    fun getConnection() = println("Connected!")
}

// Usage — no getInstance() needed!
DatabaseHelper.getConnection()
```

---

### 6. `companion object` — Replaces Java `static`
```java
// Java static
public class MathUtils {
    public static int square(int n) { return n * n; }
}
MathUtils.square(5); // 25
```
```kotlin
// Kotlin — no static keyword. Use companion object
class MathUtils {
    companion object {
        fun square(n: Int) = n * n
    }
}
MathUtils.square(5) // 25
```

---

### 7. `lateinit` and `lazy`
```kotlin
// lateinit — I'll set this value LATER (not in the constructor)
// Use for View references, injected dependencies
class MyActivity : AppCompatActivity() {
    lateinit var binding: ActivityMainBinding  // set in onCreate()
}

// lazy — compute this value only WHEN first accessed (computed once, cached)
val heavyObject: HeavyClass by lazy {
    println("Creating now!")
    HeavyClass()  // only created when first accessed
}
```
> 💡 `lateinit` = "I promise I'll set it before using it."  
> `lazy` = "Don't make it until someone asks for it."

---

## 🔑 Key Concepts

| Concept | What It Is | When to Use |
|---------|-----------|-------------|
| `class` | Blueprint for objects | Every model, controller |
| `data class` | Class for holding data | API responses, DB entities |
| `open class` | Class that can be extended | Parent/base classes |
| `abstract class` | Partial blueprint, must be completed | Base classes with shared logic |
| `interface` | Contract of behaviors | Multiple-inheritance behavior |
| `object` | Singleton — one instance ever | Utils, helpers, singletons |
| `companion object` | Static members of a class | Factory methods, constants |
| `lateinit var` | Non-null var set later | View bindings, DI |
| `by lazy` | Computed once, on first access | Expensive initializations |

---

## 💡 Good Example — A Complete Mini App Model

```kotlin
// Interface — what a Profile can do
interface Displayable {
    fun display(): String
}

// Data class — holds user data (auto toString, equals, copy)
data class Address(val city: String, val country: String)

// Open class — can be extended
open class Person(val name: String, val age: Int) {
    open fun introduce() = "Hi, I'm $name, $age years old."
}

// Inherits Person and implements Displayable
class UserProfile(
    name: String,
    age: Int,
    val email: String,
    val address: Address
) : Person(name, age), Displayable {

    // lateinit for something set later
    lateinit var profilePicUrl: String

    override fun introduce() = "${super.introduce()} Email: $email"
    override fun display() = "[$name | $email | ${address.city}]"

    companion object {
        fun createGuest() = UserProfile("Guest", 0, "guest@app.com", Address("Unknown", "Unknown"))
    }
}

// Singleton helper
object ProfileCache {
    private val cache = mutableMapOf<String, UserProfile>()
    fun save(profile: UserProfile) { cache[profile.email] = profile }
    fun get(email: String) = cache[email]
}

// --- Using it all ---
val user = UserProfile("Jayed", 28, "jayed@gmail.com", Address("Dhaka", "BD"))
user.profilePicUrl = "https://cdn.example.com/jayed.jpg"

println(user.introduce())                   // Hi, I'm Jayed, 28 years old. Email: jayed@gmail.com
println(user.display())                     // [Jayed | jayed@gmail.com | Dhaka]

val updated = user.copy(address = Address("Chittagong", "BD"))
println(updated.address.city)               // Chittagong

ProfileCache.save(user)
println(ProfileCache.get("jayed@gmail.com")?.name) // Jayed

val guest = UserProfile.createGuest()
println(guest.name)                         // Guest
```
