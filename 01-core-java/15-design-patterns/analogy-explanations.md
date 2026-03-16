# Design Patterns — Analogy Explanations

These analogies make abstract design pattern concepts intuitive and memorable. Use them to quickly explain patterns in an interview or to cement the concept in memory.

---

## Singleton — The One Headmaster

Every school has exactly **one principal/headmaster**. There is never two at the same time. No matter which classroom you walk into and ask "who is the headmaster?", you always get the same person. The school itself makes sure there is only ever one created, and it hands you a reference to that same person every time.

In code: no matter how many times you call `getInstance()`, you always get back the exact same object. The class itself controls instantiation and prevents anyone from creating a second copy.

---

## Factory Method — The Toy-Making Machine

Imagine a **toy vending machine** at a fair. You press a button labeled "car", and out comes a toy car. You press "robot", and out comes a toy robot. You don't know the internal mechanics — you don't mold the plastic yourself. You just tell the machine *what* you want, and it figures out *how* to make it.

In code: the Factory Method lets you request an object by type without knowing the concrete class. The factory decides which subclass to instantiate.

---

## Builder — Building a Sandwich Step by Step

When you order a **custom sandwich** at a deli, you don't dump all ingredients in at once. You say:
- "Start with sourdough bread"
- "Add turkey"
- "Add lettuce and tomato"
- "No mustard"
- "Wrap it up"

Each step builds on the previous one, and at the end, the deli hands you the finished sandwich. You can make completely different sandwiches using the same process just by choosing different options at each step.

In code: the Builder pattern lets you construct a complex object step by step, with optional and mandatory parts, and only "builds" the final object when you call `build()`. This eliminates the telescoping constructor anti-pattern where a constructor has 10 optional parameters.

---

## Adapter — The Power Socket Adapter

You travel from India (where sockets are round-pin) to the UK (where sockets are rectangular). Your Indian phone charger won't fit. So you use a **travel adapter** — it wraps your Indian plug on one side and presents a UK plug shape on the other. You plug the adapter into the UK socket, and your phone charges as if nothing changed.

In code: an Adapter class wraps an incompatible object and exposes an interface that the client expects. The client never knows it's talking to a different underlying class. This is useful when integrating third-party libraries that have incompatible interfaces.

---

## Decorator — Adding Toppings to a Pizza

You start with a **plain pizza base**. Then you add cheese — now it's a cheese pizza. Then you add olives on top — now it's a cheese-and-olive pizza. Then pepperoni — cheese, olive, and pepperoni pizza. Each layer *wraps* the previous one and adds something new, but the base pizza is never modified. You can stack as many toppings as you want in any combination.

In code: a Decorator wraps an existing object and adds behavior before or after delegating to the wrapped object. You can stack multiple decorators. This avoids creating a subclass explosion. Java I/O streams (`BufferedInputStream` wrapping `FileInputStream`) are a classic real-world example.

---

## Observer — The News Subscription

You **subscribe to a newspaper**. Every morning, when the newspaper publishes a new edition, it automatically lands at your door — you don't have to go check the printing press manually. If your neighbor also subscribes, they get the same paper. If you cancel your subscription, you stop getting papers. The newspaper (publisher) doesn't know or care who its individual readers are — it just notifies all current subscribers.

In code: the Subject maintains a list of Observers. When the Subject's state changes, it calls `notifyObservers()` on all registered Observers. Observers can subscribe or unsubscribe at any time. Java's `EventListener` and Spring's `ApplicationEvent` system follow this pattern.

---

## Strategy — Different Routes to School

Every morning you need to **get to school**. You have three strategies:
- Walk (slow, free, healthy)
- Ride a bike (medium, free, some effort)
- Take the bus (fast, costs money, no effort)

The goal is the same — reach school. The strategy you pick depends on context: if you're late, take the bus. If the weather is nice, walk. You can switch strategies without changing the destination.

In code: the Strategy pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable. The context object holds a reference to a strategy and delegates work to it, allowing you to swap strategies at runtime without altering the context class. Spring Security's `AuthenticationStrategy` is a real-world example.

---

## Command — The TV Remote

Your **TV remote** has buttons: Power, Volume Up, Volume Down, Channel Next, Mute. Each button is a **command**. When you press "Volume Up", the remote doesn't directly wire into the TV's speaker — it sends a command object that knows how to increase volume. You can even log all button presses, replay them, or add an "Undo Last Command" button without changing how the buttons work.

In code: the Command pattern encapsulates a request as an object. This lets you parameterize actions, queue them, log them, and support undo/redo by storing the inverse operation. Spring's `@Transactional` mechanism conceptually resembles this: the operation is wrapped, and it can be rolled back (undone).

---

## Template Method — A Recipe Template

A **recipe book** for bread says:
1. Prepare the ingredients (fixed step)
2. Mix the dough (fixed step)
3. **Add your flavoring** (customizable — herbs, cheese, raisins — you decide)
4. Let it rise (fixed step)
5. **Shape the loaf** (customizable — round, baguette, braided)
6. Bake at 200°C for 40 minutes (fixed step)

The overall structure of bread-making is fixed. But certain steps are left for you to fill in. Different bakers produce different breads using the same template.

In code: the Template Method defines the skeleton of an algorithm in a base class, with certain steps declared as `abstract` (or overridable). Subclasses fill in those steps without changing the overall algorithm structure. Spring's `JdbcTemplate` is a classic example — it handles connection management, error handling, and resource cleanup, but lets you supply the SQL and result mapping.

---

## Proxy — The Secretary Who Screens Calls

The **CEO is very busy**. So there is a secretary. When someone calls asking to speak to the CEO, the secretary first asks: "Who are you? What is this about?" If the caller isn't authorized or the request is trivial, the secretary handles it or rejects it. Only verified, important calls get connected to the CEO. The caller always dials the same number (the proxy), never the CEO's direct line.

In code: a Proxy object controls access to a RealSubject. It can add access control, lazy initialization, logging, caching, or remote communication — all transparently. The client talks to the proxy as if it were the real object. Spring AOP uses dynamic proxies to add transaction management, security checks, and logging around your beans.

---

## Summary Table

| Pattern | Core Idea | Real-World Analogy | Java Example |
|---|---|---|---|
| Singleton | One instance only | School principal | `Runtime.getRuntime()` |
| Factory Method | Delegate object creation | Toy vending machine | `Calendar.getInstance()` |
| Builder | Step-by-step construction | Custom sandwich | `StringBuilder`, Lombok `@Builder` |
| Adapter | Bridge incompatible interfaces | Travel plug adapter | `Arrays.asList()` |
| Decorator | Wrap to add behavior | Pizza toppings | `BufferedInputStream` |
| Observer | Notify all subscribers | Newspaper subscription | `EventListener`, Spring Events |
| Strategy | Swappable algorithms | Routes to school | `Comparator`, Spring Security |
| Command | Encapsulate a request | TV remote button | Runnable, Spring Batch |
| Template Method | Fixed skeleton, custom steps | Bread recipe | `JdbcTemplate` |
| Proxy | Controlled access | CEO's secretary | Spring AOP, JDK dynamic proxy |
