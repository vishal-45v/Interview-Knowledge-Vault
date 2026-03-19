# Chapter 02 — OOP & Classes: Theory Questions

---

1. What is the difference between a class, an instance, and an object in Python?
   How does `type(x)` relate to `x.__class__`? What does it mean that in Python
   "everything is an object"?

2. Explain the difference between instance methods, class methods, and static methods.
   When would you choose each? What are their respective first parameters and decorators?

3. What is `__init__`? Is it the constructor in Python? What is `__new__` and how does
   it differ from `__init__`? Give a use case where you would override `__new__`.

4. Explain Python's inheritance model. What is the difference between single inheritance
   and multiple inheritance? What problems can multiple inheritance cause?

5. What is the MRO (Method Resolution Order) and why does Python use C3 linearization?
   How do you inspect the MRO of a class? What does `super()` actually do under the hood?

6. What is `super()` and when should it be called? Why is calling `super().__init__()`
   important in inheritance? What happens if you call the parent class directly
   (`Parent.__init__(self)`) instead of using `super()`?

7. List at least 10 dunder (magic) methods and explain what each one controls. Which
   ones affect how an object is displayed, compared, used in arithmetic, or used in
   a container context?

8. What is the difference between `__str__` and `__repr__`? When is each called?
   What is the fallback behavior if only one is defined?

9. Explain `__eq__` and `__hash__`. What is the rule about the relationship between
   equality and hashability? What happens to `__hash__` when you define `__eq__`?

10. What are properties in Python? How does `@property` differ from a regular attribute?
    How do you define a setter and deleter? What is the advantage over using `getX()`
    and `setX()` methods?

11. What are descriptors? What is the descriptor protocol (`__get__`, `__set__`,
    `__delete__`)? How do `@property`, `@classmethod`, and `@staticmethod` relate to
    the descriptor protocol?

12. What are `__slots__`? When and why would you use them? What are the trade-offs —
    what do you give up when using `__slots__`?

13. What are dataclasses (`@dataclass`)? What code do they auto-generate? How do they
    compare to `namedtuple`, `TypedDict`, and writing a class manually?

14. What are Abstract Base Classes (ABCs)? How do you define an abstract method?
    What happens when you try to instantiate a class with unimplemented abstract methods?

15. What is a metaclass? What is the default metaclass? How does class creation work
    under the hood — when is `__init_subclass__` called vs `__init_class__`?

16. What is the difference between `@classmethod` and `@staticmethod`? Can a class
    method be called on an instance? Can a static method access class state?

17. Explain `__enter__` and `__exit__`. How does the `with` statement work? What
    arguments does `__exit__` receive and what does its return value control?

18. What is `__getattr__` vs `__getattribute__`? When is each called? What is the
    danger of defining `__getattribute__` incorrectly?

19. How does Python's operator overloading work? What is the difference between
    `__add__` and `__radd__`? When is `__radd__` called?

20. What is `__call__`? How do you make an instance of a class callable like a
    function? Give a real-world use case for `__call__`.
