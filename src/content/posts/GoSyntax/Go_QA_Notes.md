---
title: Go QA Notes
published: 2026-04-28
pinned: false
tags: [Syntax,Go]
category: Program Language
draft: false
---

Questions and answers from learning Go syntax. These cover tricky concepts that aren't immediately obvious from reading the guide.

---

## Table of Contents

1. [Methods and Receivers](#1-methods-and-receivers)
2. [Pointers: `*` and `&`](#2-pointers--and-)
3. [Error Creation](#3-error-creation)
4. [Slices and Maps Initialization](#4-slices-and-maps-initialization)
5. [`if` with Init Statement](#5-if-with-init-statement)
6. [`defer`](#6-defer)
7. [`panic` and `recover`](#7-panic-and-recover)
8. [Goroutines](#8-goroutines)
9. [Channels](#9-channels)
10. [Mutex](#10-mutex)
11. [`sync.Once`](#11-synconce)
12. [Worker Pool Pattern](#12-worker-pool-pattern)
13. [Buffered vs Unbuffered Channels](#13-buffered-vs-unbuffered-channels)

---

## 1. Methods and Receivers

### How do methods actually work?

A method is just a regular function where Go automatically passes the "caller" as the first parameter:

```go
// This method:
func (p Person) FullInfo() string {
    return fmt.Sprintf("%s (age %d)", p.Name, p.Age)
}

// Is essentially the same as this standalone function:
func FullInfo(p Person) string {
    return fmt.Sprintf("%s (age %d)", p.Name, p.Age)
}
```

The difference is how you call it:

```go
alice := Person{Name: "Alice", Age: 30}

alice.FullInfo()       // method syntax — Go passes alice as `p`
FullInfo(alice)        // standalone function — you pass alice yourself
```

The `(p Person)` part between `func` and the method name "attaches" the function to the `Person` type. After that, every `Person` value gets `.FullInfo()` available on it.

### Does defining a method add it to the type?

Yes. You can keep adding methods to build up the type's capabilities:

```go
func (p Person) FullInfo() string { ... }
func (p Person) IsAdult() bool { ... }
func (p Person) Greet(other string) string { ... }
```

Now `Person` has three methods:

```go
alice.FullInfo()        // "Alice (age 30)"
alice.IsAdult()         // true
alice.Greet("Bob")      // "Hi Bob, I'm Alice!"
```

This is how Go builds up types without classes — define a struct, then attach behavior one method at a time.

### Value receiver vs pointer receiver

The difference: a value receiver gets a **copy**, a pointer receiver gets the **original**.

```go
// Value receiver — works on a copy
func (p Person) HaveBirthday() {
    p.Age++    // changes the COPY only
}

// Pointer receiver — works on the original
func (p *Person) HaveBirthdayForReal() {
    p.Age++    // changes the ORIGINAL
}
```

```go
alice := Person{Name: "Alice", Age: 30}

alice.HaveBirthday()
fmt.Println(alice.Age)          // still 30 — the copy was modified, not alice

alice.HaveBirthdayForReal()
fmt.Println(alice.Age)          // 31 — alice herself was modified
```

### Where does the modified copy go?

Nowhere. It's gone.

1. Go creates a **copy** of `alice` → `p = Person{Name: "Alice", Age: 30}`
2. Inside the method, `p.Age++` → the copy now has `Age: 31`
3. The method returns → the copy is **discarded**

Nobody holds a reference to that copy. The `Age: 31` value simply disappears. That's why value receivers are useless for modification.

---

## 2. Pointers: `*` and `&`

### Why do `*` and `&` appear together in constructor functions?

```go
func NewBankAccount(owner string, initialDeposit float64) *BankAccount {
    return &BankAccount{
        Owner:   owner,
        Balance: initialDeposit,
        Active:  true,
    }
}
```

They do two different jobs:

- `*BankAccount` — in the return type, means "I'm going to return a **pointer to** a BankAccount"
- `&BankAccount{...}` — means "create a BankAccount and give me its **address**"

Think of it in two steps:

```go
// Step 1: create a BankAccount value (lives somewhere in memory)
BankAccount{Owner: "Alice", Balance: 100, Active: true}

// Step 2: & takes its address (like writing down where it lives)
&BankAccount{Owner: "Alice", Balance: 100, Active: true}
```

Analogy:

```
*BankAccount       →  "a piece of paper with a house's address on it"
&BankAccount{...}  →  "build a house and write down its address on paper"
```

You can also write it in separate steps:

```go
func NewBankAccount(owner string, initialDeposit float64) *BankAccount {
    account := BankAccount{       // create the value
        Owner:   owner,
        Balance: initialDeposit,
        Active:  true,
    }
    return &account               // return its address
}
```

---

## 3. Error Creation

### Is `errors.New()` the way to create a new error type?

No — there are two different things:

**`errors.New()` — creates a simple error value (not a new type):**

```go
var ErrNotFound = errors.New("not found")
```

This just creates a value that satisfies the `error` interface. It can only carry a fixed string message.

**A custom struct — creates an actual new error type:**

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Message)
}
```

This is a real new type. It satisfies the `error` interface because it has an `Error() string` method. The advantage: it can carry extra data that callers can inspect.

| Approach | What it gives you |
|----------|------------------|
| `errors.New("msg")` | A simple error with a fixed message |
| `fmt.Errorf("msg %d", val)` | A simple error with formatted message |
| Custom struct + `Error()` method | A rich error carrying extra data |

---

## 4. Slices and Maps Initialization

### Why use `make` to create slices and maps?

**Slices — both literal and `make` work:**

```go
nums := []int{1, 2, 3}            // literal — when you know the values
nums := make([]int, 0, 100)       // make — when you'll fill it later
```

**Maps — you MUST use `make` or a literal:**

```go
ages := map[string]int{"Alice": 30}    // literal — works
ages := make(map[string]int)            // make — works

var ages map[string]int
ages["Alice"] = 30    // PANIC! writing to a nil map
```

`var ages map[string]int` gives you a `nil` map. A nil map can be read (returns zero values) but **panics if you write to it**. `make` allocates the internal hash table so writes are safe.

| Situation | Use |
|-----------|-----|
| You know the values upfront | Literal: `[]int{1, 2, 3}` |
| You'll fill it later, know the size | `make([]int, 0, size)` |
| You'll fill it later, unknown size | `make([]int, 0)` or `make(map[string]int)` |
| Empty map you'll write to | Must use `make` or `map[string]int{}` |

---

## 5. `if` with Init Statement

### What does `if err := doSomething(); err != nil` mean?

It's two lines compressed into one. These are equivalent:

**Long form:**

```go
err := doSomething()       // execute function, get err
if err != nil {            // check err
    fmt.Println("Error:", err)
    return
}
// err still exists here
```

**Short form:**

```go
if err := doSomething(); err != nil {
    fmt.Println("Error:", err)
    return
}
// err does NOT exist here — scoped to the if block
```

The semicolon `;` splits it into two parts:

```
if err := doSomething(); err != nil {
   ^^^^^^^^^^^^^^^^^^^^^^  ^^^^^^^^^^
   init statement           condition
```

The advantage: `err` only exists inside the `if` block, so you can reuse the name:

```go
if err := step1(); err != nil {
    return err
}
// err is gone

if err := step2(); err != nil {    // no conflict
    return err
}
```

---

## 6. `defer`

### What is `defer`?

`defer` means "run this later — when the function exits."

```go
func example() {
    fmt.Println("first")
    defer fmt.Println("deferred")
    fmt.Println("second")
}

// Output:
// first
// second
// deferred      ← runs last, right before the function returns
```

### Why is it useful?

It guarantees cleanup happens no matter how the function exits:

```go
file, err := os.Open("data.txt")
if err != nil {
    return err
}
defer file.Close()    // guaranteed — runs no matter what happens below
```

Without `defer`, you'd have to call `file.Close()` before every `return` — easy to forget.

### What is a resource leak?

If you forget to close a file/connection, it stays open and occupies system resources. If this happens repeatedly (e.g., in a loop), you eventually run out and the program crashes with "too many open files."

**How to fix:** Find where the resource is opened but never closed, and add `defer Close()`.

**How to prevent:** Every time you see `Open`, `Lock`, `Dial`, or `Acquire` — the very next line should be `defer Close/Unlock/Release`.

---

## 7. `panic` and `recover`

### What is `panic`?

`panic` = your program screams "something is terribly wrong!" and crashes.

```go
func main() {
    fmt.Println("before panic")
    panic("everything is broken")
    fmt.Println("after panic")    // NEVER runs
}

// Output:
// before panic
// panic: everything is broken
// (program crashes)
```

When `panic` is called, the function immediately stops. No code after it runs. Use it only for bugs that should never happen — not for normal errors.

### What is `recover`?

`recover` = catch the panic before it crashes the program.

```go
func safeCall() {
    defer func() {
        r := recover()       // catch the panic
        if r != nil {
            fmt.Println("caught a panic:", r)
        }
    }()

    panic("oh no!")          // would normally crash...but recover catches it
}
```

Step by step:

1. `panic("oh no!")` fires — function starts to exit
2. Before exiting, Go runs `defer` functions
3. Inside the deferred function, `recover()` catches the panic value
4. Because the panic was caught, the program doesn't crash

**Important:** `recover()` only works inside a `defer` function.

**Analogy:**

```
panic   = pulling the fire alarm (everything stops)
recover = a firefighter catching it and saying "false alarm, carry on"
```

---

## 8. Goroutines

### What is a goroutine?

A goroutine is like a thread — it means "do something at the same time." (Technically lighter than OS threads, but mentally the same.)

**Without goroutines — sequential (one at a time):**

```go
sayHello("Alice")     // wait until this finishes...
sayHello("Bob")       // then run this
```

**With goroutines — concurrent (at the same time):**

```go
go sayHello("Alice")   // start this, DON'T wait for it
go sayHello("Bob")     // start this too, DON'T wait
```

`go` means: "start this function in the background, and immediately move to the next line."

### Why do we need `time.Sleep` or `WaitGroup`?

When `main` returns, the program exits and **kills all goroutines** — even if they haven't finished. You need something to keep `main` alive:

- `time.Sleep` — crude, you're guessing how long to wait
- `sync.WaitGroup` — proper way, waits until all goroutines finish

```go
var wg sync.WaitGroup

wg.Add(1)              // "1 goroutine starting"
go func() {
    defer wg.Done()    // "this goroutine finished"
    sayHello("Alice")
}()

wg.Wait()              // blocks until counter = 0
```

---

## 9. Channels

### What is a channel?

A channel is a **pipe** between goroutines. One side puts something in, the other side takes it out.

```go
ch := make(chan string)    // create a pipe that carries strings

go func() {
    ch <- "hello"          // put INTO the pipe
}()

msg := <-ch                // take OUT of the pipe (waits until something arrives)
fmt.Println(msg)           // "hello"
```

### Why not just use a variable?

Because goroutines run at the same time. Without a channel, you don't know when the other goroutine is done writing:

```go
// WITHOUT channel — broken:
var msg string
go func() {
    msg = "hello"         // when does this finish? who knows!
}()
fmt.Println(msg)          // might print "" — goroutine hasn't finished yet

// WITH channel — safe:
ch := make(chan string)
go func() {
    ch <- "hello"
}()
msg := <-ch               // WAITS until the goroutine sends
fmt.Println(msg)          // always prints "hello"
```

### Unbuffered vs buffered?

**Unbuffered = hand-to-hand delivery.** Sender waits until receiver is ready.

```go
ch := make(chan string)     // no buffer — both sides must be present
```

**Buffered = a mailbox with slots.** Sender drops it off and leaves (until mailbox is full).

```go
ch := make(chan string, 3)  // 3 slots — sender can drop off 3 without waiting
```

### What is `select` with timeout?

"Wait for a result, but give up after X seconds":

```go
select {
case result := <-ch:
    fmt.Println("Got result:", result)
case <-time.After(3 * time.Second):
    fmt.Println("Timed out!")
}
```

Whichever happens first wins. If the result arrives within 3 seconds, first case runs. If 3 seconds pass with no result, second case runs.

---

## 10. Mutex

### What happens when two goroutines access shared data at the same time?

**Without mutex — broken:**

```
Goroutine A: reads count (0)
Goroutine B: reads count (0)      ← both read 0
Goroutine A: writes count = 1
Goroutine B: writes count = 1     ← should be 2, but it's 1! Lost an increment.
```

**With mutex — safe:**

```
Goroutine A: Lock() ✓ → reads count, increments, writes → Unlock()
Goroutine B: Lock() → WAITS... → A unlocks → B gets lock ✓ → reads, increments, writes
```

Only one goroutine holds the lock at a time. The other must wait.

### `sync.Mutex` vs `sync.RWMutex`?

| Situation | `Mutex` | `RWMutex` |
|-----------|---------|-----------|
| Multiple readers at once | No — one at a time | Yes — all read simultaneously |
| Reader + writer at once | No | No — one must wait |
| Multiple writers at once | No | No |

`RWMutex` is an optimization: reading doesn't corrupt data, so there's no reason to block readers from each other. Only writing is dangerous.

### How does Go know which goroutine gets the lock first?

Whoever calls `Lock()`/`RLock()` first wins. It's essentially random. But that's fine — the mutex guarantees **it doesn't matter** which order they run. Both orders are safe.

---

## 11. `sync.Once`

### What does `sync.Once` do?

It ensures a function runs **exactly once**, even if called from multiple goroutines simultaneously.

```go
var once sync.Once

func GetDatabase() *Database {
    once.Do(func() {
        instance = connectToDatabase()    // runs only the first time
    })
    return instance
}
```

```
Goroutine 1: calls GetDatabase() → once.Do runs connectToDatabase() ✓
Goroutine 2: calls GetDatabase() → once.Do sees "already ran" → skips
Goroutine 3: calls GetDatabase() → once.Do sees "already ran" → skips
```

**Without `sync.Once` — race condition:**

```go
if instance == nil {               // two goroutines both check at the same time
    instance = connectToDatabase() // both connect — oops, connected twice!
}
```

**Analogy:** Opening a store. Ten employees arrive at the same time. Only one should unlock the door — the rest just walk in.

---

## 12. Worker Pool Pattern

### Walkthrough

```go
func main() {
    jobs := make(chan int, 100)       // to-do list (buffered)
    results := make(chan int, 100)    // finished pile (buffered)

    // Start 3 workers
    for w := 0; w < 3; w++ {
        go worker(jobs, results)
    }

    // Send 9 jobs
    for j := 0; j < 9; j++ {
        jobs <- j
    }
    close(jobs)

    // Collect results
    for r := 0; r < 9; r++ {
        fmt.Println(<-results)
    }
}

func worker(jobs <-chan int, results chan<- int) {
    for job := range jobs {
        results <- job * 2
    }
}
```

Step by step:

1. **Create pipes** — `jobs` for sending work to workers, `results` for getting answers back
2. **Start 3 workers** — each runs in the background, waiting at the `jobs` pipe
3. **Send 9 jobs** — numbers 0-8 go into the buffer
4. **`close(jobs)`** — tells workers "no more jobs coming"
5. **Workers grab jobs** — each takes the next available item from the buffer, computes `job * 2`, sends result
6. **Collect results** — main reads 9 values from `results`

The workers share the `jobs` channel. Only one worker gets each job — it's like taking an item off a shelf (once taken, it's gone).

### How does `for job := range jobs` work?

```go
// This:
for job := range jobs {
    // use job
}

// Is the same as:
for {
    job, ok := <-jobs     // take from pipe
    if !ok {              // pipe closed and empty?
        break             // stop
    }
    // use job
}
```

The loop keeps grabbing jobs until the channel is closed and empty.

### What are the results?

Values: 0, 2, 4, 6, 8, 10, 12, 14, 16 (each job × 2). But the **order is unpredictable** because 3 workers run in parallel — whoever finishes first sends their result first.

---

## 13. Buffered vs Unbuffered Channels

### "Both seem the same — workers take items one by one either way?"

From the worker's perspective, yes — no difference. The difference is on the **sender's side**:

**Unbuffered — sender must wait for a worker to take each item:**

```
Main sends 0 → waits... → worker takes it ✓
Main sends 1 → waits... → worker takes it ✓
Main sends 2 → waits... → worker takes it ✓
(9 waits total)
```

**Buffered(100) — sender dumps everything instantly:**

```
Main sends 0,1,2,3,4,5,6,7,8 → all go in buffer immediately, main moves on
Workers grab from buffer at their own pace
```

Analogy:

```
Unbuffered = hand-delivering tasks to workers one by one, waiting each time
Buffered   = putting all tasks on a table and walking away — workers help themselves
```

### Is buffered always better?

No. Unbuffered has one advantage: **backpressure**.

```
Buffered(1000): main sends 1000 jobs instantly.
  If workers are slow → 1000 items sit in memory → high memory usage.

Unbuffered: main is FORCED to slow down to workers' speed.
  Only sends the next job when a worker is free → low memory usage.
```

| Use | When |
|-----|------|
| Unbuffered | You want sender to slow down if receiver is behind (backpressure) |
| Unbuffered | You need confirmation the other side received it |
| Buffered | Sender and receiver work at different speeds |
| Buffered | You know the max items and don't want sender to wait |
