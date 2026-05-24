# Events

## What it is

An **event** is a "publisher/subscriber" channel built on delegates. The publisher declares an event; subscribers attach handlers; the publisher raises the event when something happens; all subscribed handlers run.

```csharp
public class Button {
    public event EventHandler? Clicked;

    public void OnClick() {
        Clicked?.Invoke(this, EventArgs.Empty);
    }
}

var b = new Button();
b.Clicked += (sender, e) => Console.WriteLine("clicked");
b.OnClick();   // prints "clicked"
```

Events are how UI frameworks, notification systems, and pub-sub patterns are wired in C#. They're delegates with **access restrictions**: outside callers can only `+=` and `-=`, not assign or invoke.

---

## Why they exist

A field of delegate type would *almost* work, but with two problems:

```csharp
public class Button {
    public Action? Clicked;
}

var b = new Button();
b.Clicked = () => Console.WriteLine("a");   // overwrites prior subscribers
b.Clicked();                                  // outside code can raise the event
```

External callers can:
1. **Reassign** the delegate, blowing away other subscribers.
2. **Invoke** the delegate, faking events.

Events fix both. Outside code can only subscribe (`+=`) and unsubscribe (`-=`); only the declaring class can invoke.

---

## Syntax

### Declaring

```csharp
public class Publisher {
    public event EventHandler? Something;
    public event EventHandler<MyEventArgs>? SomethingHappened;
    public event Action<string>? Status;
}
```

`event` is a modifier on a delegate-typed field. Use `EventHandler` (or `EventHandler<T>`) by convention — they have the standard `(object sender, EventArgs args)` signature.

### Subscribing

```csharp
publisher.Something += Handler;
publisher.Something += (s, e) => Console.WriteLine("got it");

void Handler(object? sender, EventArgs e) { /* ... */ }
```

### Unsubscribing

```csharp
publisher.Something -= Handler;
```

To unsubscribe a lambda, you must keep a reference:

```csharp
EventHandler h = (s, e) => Console.WriteLine("got it");
publisher.Something += h;
// later:
publisher.Something -= h;
```

You CAN'T unsubscribe `(s, e) => ...` by writing the lambda again — each lambda literal is a new delegate instance.

### Raising

```csharp
public void DoSomething() {
    Something?.Invoke(this, EventArgs.Empty);   // null-safe invocation
}
```

The `?.Invoke` pattern handles the case where nobody subscribed (event is null). The convention is to pass `this` as `sender`.

---

## The standard event pattern

By convention:
- Event type is `EventHandler` or `EventHandler<TEventArgs>`.
- First argument is the sender (the publishing object).
- Second argument is an `EventArgs` (or subclass).
- Event name reads as a past-tense verb: `Clicked`, `Closed`, `OrderPlaced`.

### Custom event args

```csharp
public class OrderPlacedEventArgs : EventArgs {
    public int OrderId { get; }
    public DateTime PlacedAt { get; }
    public OrderPlacedEventArgs(int orderId, DateTime placedAt) {
        OrderId = orderId;
        PlacedAt = placedAt;
    }
}

public class Shop {
    public event EventHandler<OrderPlacedEventArgs>? OrderPlaced;

    public void Place(Order o) {
        // ... business logic ...
        OrderPlaced?.Invoke(this, new OrderPlacedEventArgs(o.Id, DateTime.UtcNow));
    }
}
```

The `OrderPlacedEventArgs` class is the payload — anything subscribers might want to know about the event.

### "On" methods for inheritance

A common pattern lets subclasses participate:

```csharp
public class Button {
    public event EventHandler? Clicked;
    protected virtual void OnClicked(EventArgs e) {
        Clicked?.Invoke(this, e);
    }
    public void Click() => OnClicked(EventArgs.Empty);
}

public class FancyButton : Button {
    protected override void OnClicked(EventArgs e) {
        Console.WriteLine("FancyButton clicked");
        base.OnClicked(e);   // call base, which raises the event
    }
}
```

Subclasses can intercept by overriding. WinForms and WPF use this pattern extensively.

---

## Raising thread-safely

If multiple threads might modify subscribers while another thread raises, you can hit a race. The idiomatic pattern:

```csharp
public void Click() {
    var handler = Clicked;   // snapshot
    handler?.Invoke(this, EventArgs.Empty);
}
```

Read the delegate field into a local first. If a subscriber unsubscribes between read and invoke, the unsubscription doesn't affect this raise (since you have the snapshot).

`Clicked?.Invoke(...)` does this for you under the hood — the compiler generates the snapshot. So in practice, just use the null-conditional form and trust the compiler.

---

## Memory leaks via events

The #1 footgun. **Subscribing pins the subscriber in the publisher's memory**:

```csharp
public class LongLivedPublisher {
    public event EventHandler? Updated;
}

public class ShortLivedSubscriber {
    public ShortLivedSubscriber(LongLivedPublisher pub) {
        pub.Updated += OnUpdated;   // pub now holds reference to this
    }
    void OnUpdated(object? sender, EventArgs e) { /* ... */ }
}

var pub = new LongLivedPublisher();   // lives for app lifetime
for (int i = 0; i < 10000; i++) {
    var sub = new ShortLivedSubscriber(pub);   // each one pinned
}
// All 10,000 subscribers are still alive!
```

The subscriber's `OnUpdated` is registered in the publisher's delegate chain. The delegate references `this` (the subscriber). The publisher → delegate → subscriber chain prevents GC.

### Fixes

**1. Unsubscribe explicitly:**

```csharp
public class Subscriber : IDisposable {
    private readonly LongLivedPublisher _pub;
    public Subscriber(LongLivedPublisher pub) {
        _pub = pub;
        _pub.Updated += OnUpdated;
    }
    public void Dispose() => _pub.Updated -= OnUpdated;
    void OnUpdated(object? s, EventArgs e) { /* ... */ }
}

using var sub = new Subscriber(pub);   // unsubscribes at end of scope
```

**2. Use weak event patterns:**

WPF has `WeakEventManager`. For other contexts, you can use `WeakReference<T>` manually, or libraries like Weak.NET.

**3. Use events sparingly for long-running scenarios:**

Modern alternatives include:
- `IObservable<T>` (Reactive Extensions).
- `Channel<T>` for async producer/consumer.
- `MediatR`-style pub-sub buses.

---

## Custom add/remove accessors

You can customize how subscribers are stored:

```csharp
public class Publisher {
    private EventHandler? _internal;
    public event EventHandler Updated {
        add { _internal += value; Console.WriteLine("Subscriber added"); }
        remove { _internal -= value; Console.WriteLine("Subscriber removed"); }
    }
}
```

Useful for:
- Logging subscriptions.
- Lazy initialization of the underlying mechanism.
- Routing to a different storage (e.g., a `Dictionary<string, EventHandler>` keyed by category).

Most events don't need this. The default field-like event works fine.

---

## Partial events (C# 14+)

C# 14 added support for partial events alongside partial classes:

```csharp
// File: Generated.cs
public partial class Service {
    public partial event EventHandler? Started;
}

// File: Service.cs
public partial class Service {
    public partial event EventHandler? Started;
    // ... user code can add custom add/remove if desired
}
```

Used by source generators that emit event declarations and let user code provide the bodies.

---

## Internals — what an event really is

An event is **two methods** (`add` and `remove`) on a field-like delegate:

```csharp
public event EventHandler? Updated;
```

Compiles roughly to:

```csharp
private EventHandler? Updated;     // private field

public event EventHandler? Updated {
    add { Updated = (EventHandler?)Delegate.Combine(Updated, value); }
    remove { Updated = (EventHandler?)Delegate.Remove(Updated, value); }
}
```

Externally visible: just `+=` and `-=`. The internal field is private to the declaring class.

In IL:

```il
.event [System.Runtime]System.EventHandler Updated
{
    .addon instance void Publisher::add_Updated(class [System.Runtime]System.EventHandler)
    .removeon instance void Publisher::remove_Updated(class [System.Runtime]System.EventHandler)
}
```

The `.event` metadata tells reflection and tooling about the event. The actual `add` and `remove` methods are what get called.

### The compiler generates thread-safe accessors by default

```il
.method public hidebysig specialname instance void
        add_Updated(class [System.Runtime]System.EventHandler 'value') cil managed
{
    .maxstack 3
    .locals init (class System.EventHandler V_0, class System.EventHandler V_1, class System.EventHandler V_2)
    ldarg.0
    ldfld class System.EventHandler Publisher::Updated
    stloc.0
.L_0008:
    ldloc.0
    stloc.1
    ldloc.1
    ldarg.1
    call class System.Delegate System.Delegate::Combine(System.Delegate, System.Delegate)
    castclass System.EventHandler
    stloc.2
    ldarg.0
    ldflda class System.EventHandler Publisher::Updated
    ldloc.2
    ldloc.1
    call !!0 System.Threading.Interlocked::CompareExchange<class System.EventHandler>(!!0&, !!0, !!0)
    stloc.0
    ldloc.0
    ldloc.1
    bne.un.s .L_0008
    ret
}
```

This is a CAS loop using `Interlocked.CompareExchange` for thread-safe subscription updates. The compiler generates this automatically for field-like events.

### Custom accessors lose thread safety

If you write custom `add`/`remove`, the CAS loop is gone — you'd have to add it (or a lock) manually. For most events, just use the field-like form.

---

## Common patterns

### Notify on property change (MVVM)

```csharp
public class ViewModel : INotifyPropertyChanged {
    public event PropertyChangedEventHandler? PropertyChanged;
    private string _name = "";
    public string Name {
        get => _name;
        set {
            if (_name != value) {
                _name = value;
                PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Name)));
            }
        }
    }
}
```

A small helper:

```csharp
protected void OnPropertyChanged([CallerMemberName] string? name = null) =>
    PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
```

Then setters become:

```csharp
public string Name {
    get => _name;
    set {
        if (_name != value) { _name = value; OnPropertyChanged(); }
    }
}
```

### Domain events

```csharp
public class Order {
    public event EventHandler<EventArgs>? Placed;
    public event EventHandler<EventArgs>? Shipped;
    public event EventHandler<EventArgs>? Cancelled;
    // ...
}
```

For complex domain logic, consider an event bus (like MediatR) instead of per-aggregate events.

### Worker pattern

```csharp
public class Worker {
    public event EventHandler<string>? Progress;
    public event EventHandler? Completed;
    public event EventHandler<Exception>? Errored;

    public async Task RunAsync() {
        try {
            for (int i = 0; i < 100; i++) {
                Progress?.Invoke(this, $"step {i}");
                await Task.Delay(100);
            }
            Completed?.Invoke(this, EventArgs.Empty);
        } catch (Exception ex) {
            Errored?.Invoke(this, ex);
        }
    }
}
```

---

## Common bugs

- **Forgetting to unsubscribe** — long-lived publisher pins short-lived subscribers in memory.
- **Subscribing a lambda then trying to unsubscribe** — each lambda literal is a new delegate.
- **Raising a null event without `?.`** — NullReferenceException.
- **Raising on a UI thread vs background thread mismatch** — UI frameworks require updates from the UI thread. Use `Dispatcher.Invoke` (WPF) / `Control.Invoke` (WinForms) when raising from a worker.
- **Reassigning an event field externally** — compile error in event use, but easy to make happen accidentally during refactoring.
- **Exceptions in event handlers** — escape from the publisher's call. The publisher gets the exception, even though it didn't write the handler. Wrap in try/catch if you're publishing in a robust loop.

---

## Performance

- Subscribe / unsubscribe: O(1) for typical use (CAS loop, usually one iteration).
- Raising: O(N) over subscribers (each one is invoked sequentially).
- Each subscription holds one delegate (~24 bytes).
- Modifications to the subscriber list during raise are not visible to the in-progress raise (snapshot semantics).

For very high-frequency events with many subscribers, profile. Sometimes `IObservable<T>` (Rx.NET) or `Channel<T>` is a better fit.

---

## When to use events

✓ One-to-many notification.
✓ Decoupling a publisher from a varying set of subscribers.
✓ UI frameworks, callbacks.
✓ Domain events on aggregate roots.

✗ When you need ordered, async, or transactional delivery — use `Channel<T>` or a real message bus.
✗ For very high frequency — events have small but real overhead.
✗ When the lifetimes are mismatched — memory leak risk.

→ Next: [06-LocalFunctions.md](06-LocalFunctions.md)
