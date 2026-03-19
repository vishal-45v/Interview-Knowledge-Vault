# Java Basics — Follow-Up Trap Questions

> These are the tricky follow-up questions interviewers ask after you give a correct basic answer.

---

## Trap 1: "You said String is immutable — so why does `s = s + "x"` work?"

**Answer:** It works because the variable `s` is not the String — it's a reference. The statement creates a NEW String object containing the original content plus "x", then reassigns the `s` variable to point to the new object. The original String is unchanged and becomes eligible for garbage collection. Immutability refers to the object, not the variable.

---

## Trap 2: "You said `==` compares references for objects. What about enums?"

**Answer:** For enums, `==` is safe and preferred. Enum constants are guaranteed to be singletons by the JVM. `Status.ACTIVE == Status.ACTIVE` is always `true`, and you will never have two `Status.ACTIVE` instances. The Java spec guarantees this — even after deserialization, enum constants return the same singleton instance.

---

## Trap 3: "If `final` doesn't make objects immutable, what does?"

**Answer:** True immutability requires: (1) all fields are `private final`, (2) no setters, (3) the class is `final` to prevent subclassing, (4) mutable fields have defensive copies in constructor and getters, (5) no methods that modify internal state or expose internal references. Examples: `String`, `Integer`, `LocalDate`, Java records.

---

## Trap 4: "You mentioned autoboxing cache is -128 to 127. Can you change the upper bound?"

**Answer:** Yes, with the JVM flag `-XX:AutoBoxCacheMax=<N>`. This extends the Integer cache upper bound. You should never rely on this in application code — it's implementation-specific and makes code behavior depend on JVM startup flags. The cache for `Byte`, `Short`, `Character`, and `Boolean` are fixed and cannot be changed.

---

## Trap 5: "What's the difference between `String.intern()` and using a String literal?"

**Answer:** String literals are automatically placed in the String pool by the compiler. `intern()` is a runtime call that manually adds a String to the pool (or returns the existing pooled instance). Use cases for `intern()`: when you have dynamically created strings (from parsing, network I/O) that have high repetition and you want to reduce memory usage. Caveat: the pool is a fixed-size hash table; over-interning can cause contention.

---

## Trap 6: "You said Java is pass-by-value. What about arrays?"

**Answer:** Arrays are objects in Java. When you pass an array to a method, you pass a copy of the reference to the array. The method can modify the array's contents (through the copied reference), but cannot make the original variable point to a different array. Same principle as all objects — pass-by-value of the reference.

---

## Trap 7: "What is the difference between `Comparable` and `Comparator`?"

**Answer:** `Comparable<T>` is implemented by the class itself — it defines the "natural ordering" via `compareTo(T other)`. One ordering per class. `Comparator<T>` is external — defines an ordering without modifying the class. Multiple `Comparator` strategies possible. Use `Comparator` when: (1) the class is from a library and can't be modified, (2) you need multiple sort orderings, (3) you want to sort by different fields in different contexts.

---

## Trap 8: "Can a constructor be private? When would you use this?"

**Answer:** Yes. Private constructors are used in: (1) Singleton pattern — prevents direct instantiation, (2) Utility classes — classes with only static methods (like `Math`, `Collections`) should not be instantiated, (3) Builder pattern — outer class has private constructor, Builder class calls it, (4) Factory method pattern — static factory methods control instantiation. Java records and enums use private constructors internally.

---

## Trap 9: "What happens if you throw an exception in a `finally` block?"

**Answer:** The exception from `finally` completely replaces the original exception from `try` or `catch`. The original exception is silently lost (suppressed). This is a common bug. Since Java 7, you can use `Throwable.addSuppressed()` to attach suppressed exceptions. `try-with-resources` handles this automatically — if both the `try` block and `close()` throw, the `close()` exception is suppressed and accessible via `Throwable.getSuppressed()`.

---

## Trap 10: "Is `null` an object in Java?"

**Answer:** `null` is a special literal that represents the absence of an object reference. It is not an instance of any class. `null instanceof Anything` is `false`. `null.anyMethod()` throws NullPointerException. You can cast `null` to any reference type without ClassCastException. `null` can be assigned to any reference variable. Calling `null.getClass()` throws NPE (so you can't determine the type of null).

---

## Trap 11: "What is the output of `System.out.println(1 + 2 + "3")`?"

**Answer:** `"33"`. Evaluation is left-to-right: `1 + 2 = 3` (int arithmetic), then `3 + "3" = "33"` (string concatenation). Compare with `"1" + 2 + 3 = "123"` (all concatenation from left to right once a String is encountered).

---

## Trap 12: "Can you override a static method?"

**Answer:** No. Static methods belong to the class, not instances. You can define a static method with the same signature in a subclass, but that's **hiding**, not **overriding**. The method called depends on the declared type of the reference (compile-time binding), not the runtime type. This is why static methods cannot participate in polymorphism.

```java
class Parent { static void hello() { System.out.println("Parent"); } }
class Child extends Parent { static void hello() { System.out.println("Child"); } }

Parent p = new Child();
p.hello();  // "Parent" — static dispatch, not dynamic
```

---

## Trap 13: "What is the difference between `String.valueOf(null)` and `null.toString()`?"

**Answer:** `String.valueOf(null)` returns the string `"null"` (four characters). `null.toString()` throws `NullPointerException`. This is because `String.valueOf(Object obj)` has a null check: if `obj == null` return `"null"`, else return `obj.toString()`. Use `String.valueOf()` when you want null-safe string conversion.
