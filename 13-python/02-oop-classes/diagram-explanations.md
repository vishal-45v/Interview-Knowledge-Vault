# Chapter 02 — OOP & Classes: Diagram Explanations

---

## Diagram 1: Class Object Memory Layout

```
Class Definition: class Dog(Animal):
                      name = "Dog"
                      def bark(self): ...

┌─────────────────────────────────────────────────────────────────┐
│  CLASS OBJECT  (Dog)                                            │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  __dict__ (class namespace)                            │    │
│  │  ┌──────────────┬──────────────────────────────────┐   │    │
│  │  │  "name"      │  "Dog"                           │   │    │
│  │  │  "bark"      │  <function bark at 0x...>        │   │    │
│  │  │  "__init__"  │  <function __init__ at 0x...>    │   │    │
│  │  └──────────────┴──────────────────────────────────┘   │    │
│  │  __bases__ = (Animal,)                                  │    │
│  │  __mro__ = (Dog, Animal, object)                        │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
            │  instantiation: fido = Dog("Rex")
            ▼
┌─────────────────────────────────────────────────────────────────┐
│  INSTANCE OBJECT  (fido)                                        │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  __dict__ (instance namespace)                         │    │
│  │  ┌──────────────┬──────────────────────────────────┐   │    │
│  │  │  "name"      │  "Rex"   (instance attribute)    │   │    │
│  │  └──────────────┴──────────────────────────────────┘   │    │
│  │  __class__ ────────────────────────────────────────────┼───►Dog│
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘

Attribute lookup on fido.bark:
  1. Check fido.__dict__ → not found
  2. Check Dog.__dict__  → found! (also binds self=fido → bound method)
```

---

## Diagram 2: MRO — Diamond Inheritance C3 Resolution

```
Diamond inheritance:
        object
           │
           A
          / \
         B   C
          \ /
           D

        D(B, C)  →  MRO: D → B → C → A → object

Depth-first, left-to-right (WRONG — visits A twice):
  D → B → A → object → C → A → object   ← A appears twice, inconsistent

C3 Linearization (CORRECT):
  L(D) = D + merge(L(B), L(C), [B,C])
        = D + merge([B,A,object], [C,A,object], [B,C])
  Step 1: B is head of first list, not in tail of others → take B
        = D, B + merge([A,object], [C,A,object], [C])
  Step 2: A is head of first list, BUT A appears in tail of second → skip A
          C is head of second list, not in tail of others → take C
        = D, B, C + merge([A,object], [A,object], [])
  Step 3: A is head, not in any tail → take A
        = D, B, C, A + merge([object], [object], [])
  Step 4: object
  Final: D → B → C → A → object

┌─────────────────────────────────────────────────────────────────┐
│  super() in cooperative inheritance follows MRO, not "parent"  │
│                                                                 │
│  When D().method() calls super():                               │
│    D → calls B (next in D's MRO after D)                       │
│    B → calls C (next in MRO after B, when seen via D)          │
│    C → calls A (next in MRO after C, when seen via D)          │
│    A → calls object                                             │
│                                                                 │
│  Each class in the chain is visited EXACTLY once               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Diagram 3: Descriptor Protocol — Attribute Access Flow

```
Access: obj.attr
            │
            ▼
    Does type(obj).__mro__ contain a class
    whose __dict__["attr"] is a DATA descriptor?
    (has both __get__ and __set__)
            │
       ┌────┴────┐
      YES        NO
       │         │
       ▼         ▼
   Call          Check obj.__dict__["attr"]
   descriptor         │
   __get__(obj,   ┌───┴───┐
   type(obj))    FOUND    NOT FOUND
                  │          │
                  ▼          ▼
              Return it   Check type(obj).__mro__
                          for NON-DATA descriptor
                          (has __get__, no __set__)
                               │
                          ┌────┴────┐
                         FOUND     NOT FOUND
                          │          │
                          ▼          ▼
                    Call __get__   Raise AttributeError

Data descriptor:      __get__ + __set__  (e.g. @property with setter)
Non-data descriptor:  __get__ only       (e.g. plain function, @property without setter)
No descriptor:        regular value in class __dict__

Priority:  data descriptor > instance __dict__ > non-data descriptor
```

---

## Diagram 4: `__init__` vs `__new__` — Object Creation Flow

```
User calls: obj = MyClass(arg1, arg2)
                │
                ▼
    Python calls type.__call__(MyClass, arg1, arg2)
                │
                ▼
    1. instance = MyClass.__new__(MyClass, arg1, arg2)
       ┌──────────────────────────────────────────────────┐
       │  Allocates memory for the new object             │
       │  Returns the new (uninitialised) instance        │
       │  For immutable types: sets the value HERE        │
       │  Most classes just call super().__new__(cls)     │
       └──────────────────────────────────────────────────┘
                │
                ▼
    2. if isinstance(instance, MyClass):
           MyClass.__init__(instance, arg1, arg2)
       ┌──────────────────────────────────────────────────┐
       │  Initialises the already-created instance        │
       │  Sets instance attributes (self.x = arg1, etc.) │
       │  Returns None (ignored)                          │
       └──────────────────────────────────────────────────┘
                │
                ▼
    3. Returns instance to caller

Override __new__ when:
  ├── Creating a Singleton (return existing instance)
  ├── Subclassing immutable types (int, str, tuple)
  └── Factory pattern returning different subclasses
```

---

## Diagram 5: @property Implementation as Descriptor

```
@property equivalent (shows what Python actually does):

@property               is syntactic sugar for:
def radius(self):   ──► radius = property(fget=radius_getter,
    return self._r              fget=None, fdel=None)

Then @radius.setter adds fset.

class property:  # simplified CPython equivalent
    def __init__(self, fget=None, fset=None, fdel=None):
        self.fget = fget
        self.fset = fset
        self.fdel = fdel

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self          ← returns descriptor when accessed on class
        if self.fget is None:
            raise AttributeError
        return self.fget(obj)    ← calls the getter function with instance

    def __set__(self, obj, value):
        if self.fset is None:
            raise AttributeError("can't set attribute")
        self.fset(obj, value)    ← calls the setter function

    def setter(self, fset):
        return type(self)(self.fget, fset, self.fdel)  ← returns NEW property

So: @radius.setter creates a NEW property object (fget=old, fset=new)
    and rebinds the name "radius" to it.
```

---

## Diagram 6: Dunder Method Reference — Key Magic Methods

```
┌─────────────────────────────────────────────────────────────────────┐
│  CATEGORY          DUNDER METHOD        TRIGGERED BY                 │
├─────────────────────────────────────────────────────────────────────┤
│  Object lifecycle  __new__              MyClass(args)  — allocation  │
│                    __init__             MyClass(args)  — init        │
│                    __del__              object garbage collected      │
├─────────────────────────────────────────────────────────────────────┤
│  String repr       __repr__             repr(obj), containers        │
│                    __str__              str(obj), print()            │
│                    __format__           format(obj, spec)            │
├─────────────────────────────────────────────────────────────────────┤
│  Comparison        __eq__               obj == other                 │
│                    __ne__               obj != other                 │
│                    __lt__ / __gt__      obj < other                  │
│                    __le__ / __ge__      obj <= other                 │
│                    __hash__             hash(obj), dict key          │
├─────────────────────────────────────────────────────────────────────┤
│  Arithmetic        __add__              obj + other                  │
│                    __radd__             other + obj (reflected)      │
│                    __iadd__             obj += other (in-place)      │
│                    __mul__ / __rmul__   obj * n  /  n * obj          │
│                    __neg__              -obj                         │
│                    __abs__              abs(obj)                     │
├─────────────────────────────────────────────────────────────────────┤
│  Container         __len__              len(obj)                     │
│                    __getitem__          obj[key]                     │
│                    __setitem__          obj[key] = val               │
│                    __delitem__          del obj[key]                 │
│                    __contains__         x in obj                     │
│                    __iter__             for x in obj                 │
│                    __next__             next(obj)                    │
├─────────────────────────────────────────────────────────────────────┤
│  Context manager   __enter__            with obj as x:              │
│                    __exit__             end of with block            │
├─────────────────────────────────────────────────────────────────────┤
│  Attribute access  __getattr__          obj.name (fallback)          │
│                    __getattribute__     obj.name (every access)      │
│                    __setattr__          obj.name = val               │
│                    __delattr__          del obj.name                 │
├─────────────────────────────────────────────────────────────────────┤
│  Callable          __call__             obj(args)                    │
│  Boolean           __bool__             bool(obj), if obj:           │
└─────────────────────────────────────────────────────────────────────┘
```
