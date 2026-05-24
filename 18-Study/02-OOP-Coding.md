# 02 — OOP — Coding Questions

> Predict the output / find the bug. (Concepts: [02-OOP.md](02-OOP.md))

---

### Q1 — The classic: override vs new
```csharp
class Animal { public virtual string Speak() => "..."; }
class Dog : Animal { public override string Speak() => "Woof"; }
class Cat : Animal { public new string Speak() => "Meow"; }

Animal d = new Dog();
Animal c = new Cat();
Console.WriteLine(d.Speak());
Console.WriteLine(c.Speak());
```
<details><summary>Answer</summary>

**`Woof`** then **`...`**. `Dog` **overrides** → runtime dispatch picks `Dog.Speak`. `Cat` uses **`new`** (hiding) → since the variable is typed `Animal` (non-virtual), `Animal.Speak` runs. Cast to `Cat` and you'd get `Meow`. This is *the* most common OOP trick question.
</details>

---

### Q2 — What's the output?
```csharp
class Base { public Base() => Init(); public virtual void Init() => Console.WriteLine("Base"); }
class Derived : Base {
    private string _msg = "Derived field";
    public override void Init() => Console.WriteLine(_msg ?? "null!");
}
new Derived();
```
<details><summary>Answer</summary>

**`null!`**. The base constructor calls the **virtual** `Init`, which dispatches to `Derived.Init` — but this runs **before** `Derived`'s field initializers/ctor, so `_msg` is still `null`. **Never call virtual methods from a constructor.**
</details>

---

### Q3 — Will this compile? What runs?
```csharp
class A { public void M() => Console.WriteLine("A"); }
class B : A { public void M() => Console.WriteLine("B"); }   // no 'new', no 'virtual'
A x = new B();
x.M();
```
<details><summary>Answer</summary>

Compiles **with a warning** ("`B.M` hides inherited member; use `new`"). Output: **`A`** — `M` isn't virtual, so the declared type `A` decides. The missing `new`/`override` is the bug the warning flags.
</details>

---

### Q4 — Access modifier check
```csharp
class Base { protected int _x = 5; }
class Other { void Test(Base b) { Console.WriteLine(b._x); } }   // ?
```
<details><summary>Answer</summary>

**Compile error** — `protected` members are accessible only within the type and its **derived** types, not from an unrelated class in the same assembly. (Use `internal` for assembly-wide access.)
</details>

---

### Q5 — What's the output?
```csharp
interface IShape { string Name() => "shape"; }   // default interface method
class Circle : IShape { }

IShape s = new Circle();
Console.WriteLine(s.Name());
// Console.WriteLine(((Circle)s).Name());  // would this compile?
```
<details><summary>Answer</summary>

**`shape`** — the default interface method runs. The commented line **wouldn't compile**: a default interface method is **not** a member of the implementing class's surface — it's only callable through the interface type.
</details>

---

### Q6 — Find the LSP violation
```csharp
class Rectangle { public virtual int Width { get; set; } public virtual int Height { get; set; } }
class Square : Rectangle {
    public override int Width { set { base.Width = base.Height = value; } }
    public override int Height { set { base.Width = base.Height = value; } }
}
void Stretch(Rectangle r) { r.Width = 5; r.Height = 4; Assert(r.Width * r.Height == 20); }
```
<details><summary>Answer</summary>

`Stretch(new Square())` **fails the assert** (area = 16, not 20) — setting Height also changed Width. `Square` violates the `Rectangle` contract → **Liskov Substitution** violation. A subtype must be safely usable wherever the base is; here it isn't. (Classic LSP example — favor composition / don't model Square as a Rectangle.)
</details>

---

### Q7 — abstract vs interface: which compiles?
```csharp
abstract class Repo { public int Count; public abstract void Save(); }
interface IRepo { int Count; void Save(); }   // ?
```
<details><summary>Answer</summary>

The **interface line fails** — interfaces can't declare **instance fields** (`int Count;`). Abstract classes can hold state/fields; interfaces only declare members (and static fields). Use a property in the interface instead.
</details>

---

### Q8 — What does `sealed` enable here (senior)?
```csharp
public sealed class Logger { public void Log(string m) { } }
```
<details><summary>Answer</summary>

Besides preventing inheritance, `sealed` lets the JIT **devirtualize and inline** calls (no derived type can override), a real hot-path optimization — and signals "not designed for extension". The same applies to `sealed override` methods.
</details>

---

### Q9 — Output with explicit interface implementation
```csharp
interface IA { string Who(); }
class C : IA { string IA.Who() => "IA"; public string Who() => "C"; }

C c = new C();
IA a = c;
Console.WriteLine(c.Who());
Console.WriteLine(a.Who());
```
<details><summary>Answer</summary>

**`C`** then **`IA`**. The class's public `Who` is called via the class reference; the **explicit** `IA.Who` is called only via the interface reference. Explicit implementation hides the member from the class's public surface.
</details>

---

### Q10 — Polymorphism + collection
```csharp
var animals = new List<Animal> { new Dog(), new Cat() };   // from Q1
foreach (var a in animals) Console.WriteLine(a.Speak());
```
<details><summary>Answer</summary>

**`Woof`** then **`...`**. Each element is statically typed `Animal`. `Dog` overrides (→ `Woof`); `Cat` only hides with `new`, so through the `Animal` reference you get `Animal.Speak` (`...`). Hiding doesn't participate in polymorphism — even in a collection.
</details>
