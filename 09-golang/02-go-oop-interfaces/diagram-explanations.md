# Chapter 02 — Go OOP & Interfaces: Diagram Explanations

---

## Diagram 1: Struct Embedding — Promoted Fields and Methods

```
Embedding chain:

type Address struct {         type Contact struct {
    Street string                 Phone  string
    City   string                 Email  string
}                             }
func (a Address) Format()     func (c Contact) String()


type Person struct {
    Name    string            ← own field
    Age     int               ← own field
    Address                   ← embedded (no field name)
    Contact                   ← embedded (no field name)
}

Method and field lookup for Person:
┌──────────────────────────────────────────────────────────┐
│  p.Name         → Person.Name (direct)                   │
│  p.Age          → Person.Age  (direct)                   │
│  p.Street       → Person.Address.Street (promoted)       │
│  p.City         → Person.Address.City   (promoted)       │
│  p.Phone        → Person.Contact.Phone  (promoted)       │
│  p.Email        → Person.Contact.Email  (promoted)       │
│  p.Format()     → Person.Address.Format() (promoted)     │
│  p.String()     → Person.Contact.String() (promoted)     │
│                                                          │
│  p.Address.Format() → still works (explicit access)     │
│  p.Contact.String() → still works (explicit access)     │
└──────────────────────────────────────────────────────────┘

Ambiguity (same depth):
If both Address and Contact had a method called Info():
  p.Info()    → COMPILE ERROR: ambiguous selector
  p.Address.Info() → OK
  p.Contact.Info() → OK

If Person defines its own Info():
  p.Info()    → Person.Info() (shadows both)
```

---

## Diagram 2: Interface Internal Representation (itab + data)

```
An interface value occupies 16 bytes (two pointer-sized words):

┌──────────────────┬──────────────────┐
│   type pointer   │   data pointer   │
│    (*itab)       │   (*value)       │
└──────────────────┴──────────────────┘
         │                   │
         ▼                   ▼
   ┌───────────┐      ┌───────────────┐
   │   itab    │      │  concrete     │
   │ ─────────│      │  value data   │
   │ itype    │      │  (or pointer  │
   │ (interface│      │   to it)      │
   │  type)   │      └───────────────┘
   │ concrete │
   │  type    │
   │ ─────────│
   │ fun[0]   │──► Method 1 implementation
   │ fun[1]   │──► Method 2 implementation
   │ ...      │
   └───────────┘

Example: var r io.Reader = &os.File{...}

┌──────────────────┬──────────────────┐
│ itab for         │  pointer to      │
│ (*os.File,       │  os.File value   │
│  io.Reader)      │  0xc0001234      │
└──────────────────┴──────────────────┘
         │
         ▼
   ┌───────────────┐
   │ itype: Reader │ ← interface type descriptor
   │ type: *File   │ ← concrete type descriptor
   │ fun[0]:       │──► (*os.File).Read  ← actual method
   └───────────────┘

When r.Read(buf) is called:
1. Load itab pointer from interface
2. Load fun[0] from itab (the Read method pointer)
3. Load data pointer (the *os.File value)
4. Call fun[0](data, buf)
```

---

## Diagram 3: Nil Interface vs Interface Holding Nil Pointer

```
Case 1: Truly nil interface
  var err error = nil

  ┌──────────┬──────────┐
  │   nil    │   nil    │   ← BOTH components nil
  └──────────┴──────────┘
  err == nil → TRUE ✓

Case 2: Interface holding a nil pointer (THE TRAP)
  var myErr *MyError = nil
  var err error = myErr

  ┌──────────────────┬──────────┐
  │ itab for         │   nil    │   ← type is set, value is nil
  │ (*MyError,error) │          │
  └──────────────────┴──────────┘
  err == nil → FALSE ✗ (type component is non-nil!)

Visual comparison:

  Truly nil interface:   [  nil  |  nil  ]    == nil → true
  Typed nil pointer:     [ itab  |  nil  ]    == nil → FALSE ← TRAP!
  Normal interface:      [ itab  | 0xABC ]    == nil → false

Code that triggers the trap:
  func mayFail() error {
      var err *MyError = nil
      return err  // returns [itab|nil], not a nil interface!
  }

Code that avoids the trap:
  func mayFail() error {
      return nil  // returns [nil|nil], truly nil interface
  }
```

---

## Diagram 4: Go Polymorphism via Interface — Runtime Dispatch

```
Compile time — interface variable 's' of type Shape:

  var s Shape = Circle{R: 5}
        │
        └── type info: *itab{itype:Shape, type:Circle}
            value ptr: Circle{R:5}

  s.Area() call:
    1. Load itab from 's'
    2. Find Area() in itab's method table → &Circle.Area
    3. Call Circle.Area(s.data)

Later: s = Rectangle{W:4, H:6}
        │
        └── type info: *itab{itype:Shape, type:Rectangle}
            value ptr: Rectangle{W:4, H:6}

  s.Area() call now routes to &Rectangle.Area

This is RUNTIME dispatch — the concrete method is chosen at runtime
based on what's stored in the interface variable.

Compare to Java:
  - Java: virtual method table in the class object (class-based hierarchy)
  - Go:   itab per (interface, concrete-type) pair (structural typing)

Polymorphism flow:
┌─────────────────┐
│ []Shape{...}    │
│  Circle{R:5}    │──► s.Area() ──► Circle.Area()  → 78.54
│  Rectangle{4,6} │──► s.Area() ──► Rectangle.Area() → 24.0
│  Triangle{3,4,5}│──► s.Area() ──► Triangle.Area() → 6.0
└─────────────────┘
   Each element has its own itab — dispatch to the right method
```

---

## Diagram 5: io.Reader Composition Chain

```
Data flows LEFT to RIGHT through a pipeline of io.Reader implementations:

Physical Source          Transformations               Your Code
     │                        │                           │
     ▼                        ▼                           ▼

┌──────────┐     ┌──────────────┐     ┌────────────┐     ┌──────────────┐
│ os.File  │────►│ gzip.Reader  │────►│ bufio.     │────►│ io.Copy,     │
│          │     │ (decompress) │     │ Reader     │     │ scanner,     │
│ (disk)   │     │              │     │ (buffered) │     │ your code    │
└──────────┘     └──────────────┘     └────────────┘     └──────────────┘
  implements       wraps any           wraps any           receives any
  io.Reader        io.Reader           io.Reader           io.Reader

Every layer just sees a  Read(p []byte) (int, error)  interface.
No layer knows what's behind it. Swap any layer without changing others.

Example construction:
  f, _ := os.Open("data.gz")        // *os.File    implements io.Reader
  gr, _ := gzip.NewReader(f)        // *gzip.Reader wraps f (io.Reader)
  br := bufio.NewReader(gr)         // *bufio.Reader wraps gr (io.Reader)
  scanner := bufio.NewScanner(br)   // uses br as io.Reader

Alternative (network + TLS + gzip):
  conn, _ := net.Dial("tcp", "...")      // net.Conn implements io.Reader
  tls  := tls.Client(conn, config)       // *tls.Conn wraps conn
  gr,_ := gzip.NewReader(tls)            // decompress TLS stream
  // Same io.Reader interface throughout!

Testing: replace os.File with strings.Reader — no other code changes:
  strings.NewReader("fake file content")  // io.Reader for tests
```

---

## Diagram 6: Method Sets and Interface Satisfaction Rules

```
Type T:
  value receiver methods:   Area(), String()
  pointer receiver methods: Scale(), Reset()

Method set of T (value type):
  ┌──────────────────────┐
  │  Area()              │  ← value receiver
  │  String()            │  ← value receiver
  └──────────────────────┘
  Can satisfy interfaces requiring only Area() and/or String()

Method set of *T (pointer type):
  ┌──────────────────────┐
  │  Area()              │  ← value receiver (inherited)
  │  String()            │  ← value receiver (inherited)
  │  Scale()             │  ← pointer receiver
  │  Reset()             │  ← pointer receiver
  └──────────────────────┘
  Can satisfy interfaces requiring ANY combination of these methods

Interface satisfaction matrix:
  Interface I1 { Area() }          → satisfied by T  ✓ and *T ✓
  Interface I2 { Scale() }         → satisfied by *T ✓ only, NOT T ✗
  Interface I3 { Area(); Scale() } → satisfied by *T ✓ only, NOT T ✗

Why you cannot use T to satisfy I2:
  var t T = T{}
  var i I2 = t   // COMPILE ERROR: T does not implement I2 (Scale has pointer receiver)
  var i I2 = &t  // OK — *T implements I2

  // Map values are not addressable:
  m := map[string]T{"key": T{}}
  // m["key"].Scale() — COMPILE ERROR if Scale has pointer receiver
  // Cannot take address of map value
```
