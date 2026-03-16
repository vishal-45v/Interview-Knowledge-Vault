# Chapter 10: Reflection — Analogy Explanations

---

## Reflection (Inspecting a Class at Runtime)

Imagine you receive a black-box appliance — say, an unknown electronic device with no manual. You have two options. Option A: try to use it based on what you see (buttons, slots, dials). Option B: take it to an electronics lab, put it on the inspection bench, run it through diagnostic tools, open it up with special tools, read the serial numbers off every component, and even replace parts.

Normal Java programming is Option A — you know what you are working with at compile time. Reflection is Option B — at runtime, you can inspect any class: list all its fields, all its methods, read their names and types, invoke them even if they are private, and even create new instances of classes you never knew existed when you wrote your code.

```java
// Option A: normal programming — you know the type at compile time
String s = "hello";
int len = s.length(); // compiler knows String has length()

// Option B: reflection — discover and use at runtime
Class<?> clazz = Class.forName("java.lang.String");
Method lengthMethod = clazz.getMethod("length");
Object result = lengthMethod.invoke("hello"); // call length() at runtime
System.out.println(result); // 5
```

---

## getDeclaredMethods vs getMethods

Think of a company org chart with two questions: "Who works in THIS office?" vs "Who can I call in this entire organization including all parent departments?"

`getDeclaredMethods()` is "who works in this specific office (class)" — all methods declared directly in that class, including private ones. It does not include methods from parent classes.

`getMethods()` is "who can I publicly call in this entire organization" — all public methods, including those inherited from parent classes and interfaces. It excludes private, protected, and package-private methods.

```java
class Animal {
    public void breathe() {}
    protected void sleep() {}
    private void dream() {}
}

class Dog extends Animal {
    public void bark() {}
    private void wag() {}
}

// getDeclaredMethods() — only methods IN Dog, including private
Method[] declared = Dog.class.getDeclaredMethods();
// Returns: [bark, wag] — both from Dog, not from Animal

// getMethods() — public methods from Dog AND inherited public
Method[] all = Dog.class.getMethods();
// Returns: [bark, breathe, ...Object methods (toString, hashCode, equals...)]
// NOTE: sleep() not included (protected), wag() not included (private)
```

---

## setAccessible(true) Breaking Encapsulation

Imagine a filing cabinet with a lock marked "PRIVATE — HR USE ONLY." Normally, only HR staff can open it. But there is a master key in the security office that any maintenance worker can request. Once you use the master key (`setAccessible(true)`), you can open any locked cabinet, read any file, and even change the contents — regardless of the lock.

This is `setAccessible(true)`. It bypasses Java's access control (private, protected, package-private) entirely, allowing you to read or modify any field and call any method. In Java 9+, the module system adds a second lock that even the master key cannot always open — unless the module's `module-info.java` explicitly includes an `opens` directive.

```java
class BankAccount {
    private double balance = 1000.00; // private — only BankAccount can touch this
}

// Normal code — compile error:
// BankAccount acc = new BankAccount();
// double b = acc.balance; // ERROR: balance has private access

// With reflection master key:
BankAccount acc = new BankAccount();
Field balanceField = BankAccount.class.getDeclaredField("balance");
balanceField.setAccessible(true);  // use the master key

double balance = (double) balanceField.get(acc);  // read private field
System.out.println(balance); // 1000.0

balanceField.set(acc, 999999.99); // modify private field
```

---

## Dynamic Proxy

A dynamic proxy is like a front desk receptionist at a corporate office. The receptionist (proxy) intercepts every visitor (method call) before they reach the executive (real object). The receptionist logs the visit, checks ID, maybe reschedules, and then either forwards the visitor or handles them directly.

Unlike a regular receptionist (a concrete wrapper class you write yourself), a dynamic proxy is created automatically at runtime for any "role" (interface) you specify — you do not write the proxy class, Java generates it for you. The only thing you write is the receptionist's decision logic (`InvocationHandler`).

```java
interface Executive {
    void meetWith(String visitor);
    String signDocument(String doc);
}

class RealExecutive implements Executive {
    public void meetWith(String visitor) { System.out.println("Meeting: " + visitor); }
    public String signDocument(String doc) { return "SIGNED: " + doc; }
}

// The receptionist's script (InvocationHandler)
class Receptionist implements InvocationHandler {
    private final Executive executive;
    Receptionist(Executive e) { this.executive = e; }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Logging: call to " + method.getName());
        return method.invoke(executive, args); // forward to real executive
    }
}

// Create the proxy — no class file written, generated at runtime
Executive proxy = (Executive) Proxy.newProxyInstance(
    Executive.class.getClassLoader(),
    new Class[]{Executive.class},
    new Receptionist(new RealExecutive())
);
proxy.meetWith("Alice"); // goes through receptionist first
```

---

## Class.forName() and Class Loading

Imagine a factory warehouse full of product blueprints (`.class` files). When your program starts, not all blueprints are loaded into the workshop — only the ones currently needed. `Class.forName("com.example.Plugin")` is like saying "go to the warehouse, find the blueprint labeled 'com.example.Plugin', bring it to the workshop, and make it ready to use."

Once the blueprint is in the workshop (class is loaded), you can inspect it (reflection) or use it to manufacture objects (`newInstance()`). If the warehouse does not have that blueprint (class not on classpath), you get a `ClassNotFoundException`.

```java
// Normal class loading — JVM loads it when first referenced in code
MyService service = new MyService(); // JVM loads MyService when this line first executes

// Class.forName() — load by name at runtime (for plugin systems, JDBC drivers)
try {
    // JDBC driver registration — old style
    Class.forName("com.mysql.cj.jdbc.Driver"); // loads driver, triggers static initializer

    // Plugin loading — load a class you don't know at compile time
    String pluginClassName = config.getProperty("plugin.class");
    Class<?> pluginClass = Class.forName(pluginClassName);
    Plugin plugin = (Plugin) pluginClass.getDeclaredConstructor().newInstance();
    plugin.execute();

} catch (ClassNotFoundException e) {
    System.out.println("Blueprint not found: " + e.getMessage());
}
```

---

## Annotations at Runtime

Think of a luggage tag at an airport. When you check in your bag (write your class), you attach tags (annotations) to it. The bag goes on the conveyor (compilation). When the bag arrives at the destination (runtime), the handling system (framework) scans each tag and acts accordingly: "Priority tag → put in first-class carousel. Fragile tag → handle gently."

`@Retention(RetentionPolicy.RUNTIME)` means the tag is durable — it survives the journey and is still readable at the destination. `@Retention(SOURCE)` means the tag dissolves during the journey and is unreadable at runtime.

```java
@Retention(RetentionPolicy.RUNTIME)   // durable tag — survives to runtime
@Target(ElementType.METHOD)
@interface RateLimit {
    int requestsPerSecond() default 10;
}

class ApiController {
    @RateLimit(requestsPerSecond = 5)  // attaching the luggage tag
    public Response getUsers() { return new Response(); }
}

// At the destination (runtime), reading the tag:
for (Method method : ApiController.class.getDeclaredMethods()) {
    RateLimit limit = method.getAnnotation(RateLimit.class);
    if (limit != null) {
        // Install rate limiter for this method
        RateLimiter.install(method, limit.requestsPerSecond());
    }
}
```

---

## Constructor.newInstance()

Think of a stamp with replaceable faces. Normally you pick up a physical stamp with a fixed face and stamp with it (normal `new` keyword — you know exactly what you are creating). With `Constructor.newInstance()`, you have a stamp-making kit. At runtime, you specify what the face should look like (which class, which constructor parameters), and the kit manufactures the stamp for you — you can then use that stamp to make as many objects as you want.

This is how dependency injection frameworks (Spring, Guice) create your beans: they read `@Autowired` or configuration files, look up constructor descriptors, and call `Constructor.newInstance()` to build your objects.

```java
// Normal object creation — must know the type at compile time
UserService service = new UserService(repo, validator);

// Constructor.newInstance() — create at runtime from description
Class<?> clazz = Class.forName("com.example.UserService");

// Find the constructor that takes a Repository and Validator
Constructor<?> ctor = clazz.getDeclaredConstructor(Repository.class, Validator.class);
ctor.setAccessible(true); // if constructor is not public

// Create an instance passing arguments
Object instance = ctor.newInstance(repository, validator);
UserService svc = (UserService) instance;

// vs Class.newInstance() — DEPRECATED in Java 9
// It only calls no-arg constructor and wraps checked exceptions poorly
// Always prefer getDeclaredConstructor().newInstance() instead
```

---

## Method.invoke()

Think of a remote control. You have a television (the object) with many buttons (methods). Normally you are standing right in front of the TV pressing buttons yourself (direct method calls). `Method.invoke()` is a universal remote control — you can send commands to any TV (any object), call any button (any method) by number/name at runtime, even buttons not shown on the regular remote (private methods).

The universal remote works even if you never saw the TV's manual at the time you programmed the remote.

```java
class Calculator {
    public int add(int a, int b) { return a + b; }
    private int secretMultiply(int a, int b) { return a * b; }
}

Calculator calc = new Calculator();

// Direct call — compile-time binding
int result = calc.add(3, 4); // 7

// Method.invoke() — runtime binding
Method addMethod = Calculator.class.getMethod("add", int.class, int.class);
int reflectResult = (int) addMethod.invoke(calc, 3, 4); // 7

// Calling a private method with the universal remote
Method secretMethod = Calculator.class.getDeclaredMethod("secretMultiply", int.class, int.class);
secretMethod.setAccessible(true); // override the "private" lock
int secret = (int) secretMethod.invoke(calc, 6, 7); // 42
```
