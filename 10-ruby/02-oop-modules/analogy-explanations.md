# Chapter 02 — OOP & Modules: Analogy Explanations

---

## Analogy 1 — Classes as Blueprints

**Concept:** A class is a blueprint; objects are instances built from that blueprint.

**Analogy:** A class is like an architect's floor plan for an apartment. The plan describes how many rooms there are, where the kitchen goes, and what features the apartment will have. But the plan itself is not an apartment — it's just paper. When you build an actual apartment (call `.new`), you get a real place where people can live. You can build a hundred apartments from the same plan, and each one exists independently. If someone moves into apartment #1, that doesn't affect apartment #2, even though they were built from the same blueprint.

---

## Analogy 2 — Inheritance: The IS-A Relationship

**Concept:** A subclass IS-A more specific type of the parent class.

**Analogy:** Think of a company org chart. A `SoftwareEngineer` IS-A `Engineer` IS-A `Employee`. Every software engineer can do everything any employee can (badge scan, attend meetings, get paid) plus additional things specific to engineers. When HR sends an email to "all employees," software engineers receive it too — because they inherit from Employee. But HR can't give a dog the employee badge, because a dog does not IS-A Employee.

---

## Analogy 3 — include (Mixin as Trait)

**Concept:** include adds module methods to instances — like adding a skill to a person.

**Analogy:** A mixin via `include` is like earning a professional certification. You can be a `Doctor` who also has a `Pilot` certification and a `Chef` certification. The certifications (modules) give you new capabilities (`fly_plane`, `cook_meal`). Any Doctor can earn any combination of certifications. Multiple Doctors can share the same certification without any conflict. This is far more flexible than only inheriting from one parent class.

---

## Analogy 4 — extend (Class-Level Module)

**Concept:** extend adds module methods to the class itself (not instances).

**Analogy:** `include` gives skills to employees; `extend` gives skills to the company headquarters. If you `include Findable`, each user object gets the `find` method. If you `extend Findable`, the User class itself (the HR department) gets the `find` method — it can look up user records without creating a user first. The headquarters can search; individual employees cannot search for other employees through that same method.

---

## Analogy 5 — prepend (Method Decoration/Wrapping)

**Concept:** prepend inserts the module before the class in the method lookup chain.

**Analogy:** Prepend is like putting a security checkpoint at the entrance to a building. Normally when someone knocks, the building owner (your class) answers the door. With prepend, a security guard (the module) stands in front of the building and intercepts every knock first. The guard can check credentials (`validate`), log the visit (`audit`), or refuse entry. Then they call the actual owner (`super`). The building owner has no idea the guard is there — they just answer as normal.

---

## Analogy 6 — Method Resolution Order (MRO)

**Concept:** Ruby walks a linear chain of ancestors to find a method.

**Analogy:** Imagine you need a legal ruling. You ask your local lawyer first (your class). If they don't know, you ask the specialist firm they brought in (modules, last included first). If the specialist firm doesn't know, you escalate to the regional partner firm (superclass). Keep going up to the Supreme Court (BasicObject). The first person in the chain who can answer does — and everyone below them never gets asked. `super` is like explicitly saying "I'm going to pass this question up to my superior."

---

## Analogy 7 — Public, Protected, and Private

**Concept:** Method visibility controls who can call which methods.

**Analogy:** Think of an employee's work responsibilities:
- **Public methods** are like your official job duties — listed on your LinkedIn, anyone can ask you to do them. Your boss, clients, and colleagues can all request public work.
- **Protected methods** are like internal team processes — your teammates can call on them, and your manager can too, but a random customer from outside the company cannot. In Ruby, other instances of the same class can call protected methods on each other, which enables comparison methods.
- **Private methods** are like personal routines — how you organize your workspace, your mental checklist before starting. Only you call them; no one from outside triggers them directly.

---

## Analogy 8 — Duck Typing

**Concept:** Check what an object can do, not what it is.

**Analogy:** You need someone to drive a truck. You don't care if they're a "Trucker" or a "DeliveryPerson" or a "Contractor" — you just check: "Do you have a CDL and can you drive a truck?" If yes, get in and go. Ruby's `respond_to?(:drive_truck)` is that check. You don't need a formal "IDrivable" interface — you just ask the question and proceed. This is why Ruby code tends to be more flexible: a class you haven't even written yet can work in your system, as long as it responds to the right methods.

---

## Analogy 9 — Struct as a Labeled Tuple

**Concept:** A Struct is a lightweight named data container.

**Analogy:** A Struct is like a labeled envelope with fixed compartments. You label the envelope "Coordinate" and it has two labeled slots: "lat" and "lng." Whenever you create a Coordinate, you fill in the slots. Any two Coordinate envelopes with the same numbers in their slots are considered identical (value equality). You don't need a full filing cabinet (class) for this — just a labeled envelope is enough. It's perfect for GPS points, color values, configuration pairs, and any other simple, named data that doesn't need complex behavior.

---

## Analogy 10 — Singleton Class (Eigenclass)

**Concept:** Every object has its own private class where you can add methods for just that object.

**Analogy:** Every person has a shared resume template (the class), but each person also has a unique skills section where they can list certifications only they have earned. If Alice learned skateboarding, only Alice's unique skills section has that — not every other Person. Ruby's singleton class is that unique section. When you write `def alice.special_skill`, you're writing in Alice's personal section, not in the shared Person class.
