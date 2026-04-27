---
title: Go Syntax Guide
published: 2026-04-27
pinned: false
tags: [Syntax,Go]
category: Program Language
draft: false
---

This guide teaches you Go syntax through simple, self-contained examples. Every concept is illustrated with small programs you could type and run yourself. By the end, you will be able to read any `.go` file and understand what it does.

### What you'll encounter when reading a `.go` file

When you open any `.go` file, you'll see constructs roughly in this order:

1. A **package declaration** — which module this file belongs to
2. An **import block** — dependencies from the standard library and other packages
3. **Constants and variables** — package-level values (`const`, `var`)
4. **Type definitions** — structs, interfaces, and custom types
5. **Functions and methods** — standalone functions and methods attached to types

This guide covers each construct, then builds up to larger patterns: error handling, collections, control flow, concurrency, and testing. A [quick reference table](#quick-reference) at the end serves as a cheat sheet while you read code.

---

## Table of Contents

1. [Project Structure and Imports](#1-project-structure-and-imports)
2. [Variables and Types](#2-variables-and-types)
3. [Functions](#3-functions)
4. [Structs and Methods](#4-structs-and-methods)
5. [Interfaces](#5-interfaces)
6. [Pointers](#6-pointers)
7. [Error Handling](#7-error-handling)
8. [Collections: Slices, Arrays, and Maps](#8-collections-slices-arrays-and-maps)
9. [Control Flow](#9-control-flow)
10. [Concurrency](#10-concurrency)
11. [Testing](#11-testing)
12. [Common Patterns in Go](#12-common-patterns-in-go)

---

## 1. Project Structure and Imports

### `package main` and the entry point

Every Go program starts at `func main()` inside a file that declares `package main`. Here is the simplest Go program:

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, world!")
}
```

- `package main` tells the compiler this is an executable (not a library).
- `func main()` takes no arguments and returns nothing — it is the starting point of the program.
- `fmt.Println(...)` prints a line to the terminal. `fmt` is Go's formatting package.

### How Go organizes code

Every `.go` file starts with a `package` declaration. A package is a directory of `.go` files that are compiled together — similar to a module or namespace in other languages.

```go
// file: math/calculator.go
package math
```

```go
// file: store/inventory.go
package store
```

All files in the same directory must declare the same package name. You reference code from another package by importing it.

### The module file: `go.mod`

At the root of your project, `go.mod` defines the module path and dependencies. It serves a similar role to `package.json` (Node.js) or `requirements.txt` (Python).

```go
module github.com/yourname/myproject

go 1.21
```

- `module github.com/yourname/myproject` — this is the base import path. Every package in this project is imported relative to this path.
- `go 1.21` — minimum Go version required.

### Import blocks

Imports are grouped in parentheses. By convention, standard library imports come first, then external packages:

```go
import (
    "fmt"          // standard library — printing
    "math/rand"    // standard library — random numbers
    "strings"      // standard library — string utilities

    "github.com/yourname/myproject/store"    // your own package

    log "github.com/sirupsen/logrus"         // aliased import
)
```

- `"fmt"` — imports the `fmt` package. You use it as `fmt.Println(...)`.
- `"github.com/yourname/myproject/store"` — imports your `store` package. You use it as `store.AddItem(...)`.
- `log "github.com/sirupsen/logrus"` — imports a package under the alias `log`, so you write `log.Info(...)` instead of `logrus.Info(...)`. This avoids name collisions or provides a shorter name.

### Exported vs unexported: the capitalization rule

Go has no `public`/`private` keywords. Instead, the first letter of a name determines visibility:

- **Uppercase = exported** (accessible from other packages): `Add`, `UserName`, `MaxRetries`
- **Lowercase = unexported** (only accessible within the same package): `helper`, `count`, `defaultTimeout`

```go
package store

const MaxItems = 100       // Exported — other packages can use store.MaxItems

var itemCount int          // unexported — only used inside package store

func AddItem(name string) { ... }    // Exported
func validate(name string) { ... }   // unexported
```

This is enforced by the compiler: if you try to use `store.itemCount` from another package, it won't compile.

---

## 2. Variables and Types

### Type definitions

Go lets you create new named types. This gives you type safety and the ability to attach methods:

```go
type Celsius float64       // temperature in Celsius
type Fahrenheit float64    // temperature in Fahrenheit
type UserID int64          // a unique user identifier
```

- `type Celsius float64` creates a new type called `Celsius` that is backed by a `float64`. It is *not* an alias — `Celsius` and `float64` are distinct types. You can attach methods to `Celsius` but not to `float64`.
- You cannot accidentally pass a `Fahrenheit` where a `Celsius` is expected — the compiler catches this.

### Variable declarations: `var`

`var` declares a variable with an explicit type. It can appear at the package level or inside a function.

```go
var name string = "Alice"
var age int                   // zero value: 0
var isActive bool             // zero value: false
```

Package-level variables are often grouped in a `var (...)` block:

```go
var (
    maxRetries  = 3
    defaultName = "Guest"
    timeout     = 30   // seconds
)
```

### Short variable declaration: `:=`

Inside a function, `:=` declares and initializes a variable in one step. The type is inferred from the right side.

```go
func greet() {
    message := "Hello!"           // message is a string
    count := 42                   // count is an int
    price := 19.99                // price is a float64
    data, err := loadFile()       // declare two variables at once
    _ = err                       // _ discards a value you don't need
}
```

- `:=` can only be used inside functions, never at package level.
- The `_` (blank identifier) explicitly discards a value you don't need.

### Constants and `iota`

`const` declares compile-time constants. They cannot be changed after declaration:

```go
const Pi = 3.14159
const MaxUsers = 1000
```

When you need a sequence of related constants (like an enum), `iota` auto-generates the values. It starts at 0 and increments by 1 for each line in the `const` block:

```go
type Direction int

const (
    North Direction = iota   // North = 0
    East                     // East = 1
    South                    // South = 2
    West                     // West = 3
)
```

- `Direction` is a custom type based on `int`. Giving constants a named type makes the code self-documenting and catches misuse at compile time.
- `East` inherits both the type and the `iota` pattern from the first line — you don't need to repeat `Direction = iota`.

Another example — days of the week:

```go
type Weekday int

const (
    Monday    Weekday = iota + 1   // Monday = 1
    Tuesday                         // Tuesday = 2
    Wednesday                       // Wednesday = 3
    Thursday                        // Thursday = 4
    Friday                          // Friday = 5
    Saturday                        // Saturday = 6
    Sunday                          // Sunday = 7
)
```

### Zero values

In Go, every variable has a default value if not explicitly initialized. These are called **zero values**:

| Type | Zero value |
|------|-----------|
| `int`, `uint64`, etc. | `0` |
| `float64` | `0.0` |
| `bool` | `false` |
| `string` | `""` |
| pointer, slice, map, channel, interface | `nil` |
| struct | each field is its zero value |

This is used frequently in Go code. For example:

```go
var count int       // count is 0
var name string     // name is ""
var active bool     // active is false

if name == "" {
    fmt.Println("No name set")
}
```

For value types (integers, strings, structs, arrays), Go uses zero values instead of null. Reference types (pointers, slices, maps, channels, interfaces) can be `nil` — see [Pointers: Nil checks](#nil-checks).

### Type conversions

Go has no implicit type conversions. You must be explicit:

```go
var i int = 42
var f float64 = float64(i)       // int → float64
var u uint = uint(f)             // float64 → uint

length := len("hello")           // len() returns int
size := int64(length)            // int → int64
```

If types are incompatible (like `string` to `int`), you use standard library functions:

```go
import "strconv"

num, err := strconv.Atoi("123")          // string → int
text := strconv.Itoa(42)                  // int → string
```

---

## 3. Functions

### Basic function syntax

A Go function has: the `func` keyword, a name, parameters, optional return types, and a body:

```go
func add(a int, b int) int {
    return a + b
}

func greet(name string) string {
    return "Hello, " + name + "!"
}
```

- `func add` — the `func` keyword followed by the function name.
- `(a int, b int)` — two parameters, both integers.
- `int` after the closing paren — the return type.

When consecutive parameters share a type, you can shorten:

```go
func add(a, b int) int {       // a and b are both int
    return a + b
}
```

### Multiple return values

Go functions routinely return multiple values. The most common pattern is `(result, error)`:

```go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("cannot divide by zero")
    }
    return a / b, nil
}
```

- The function returns two values: a `float64` result and an `error`.
- On success, the error is `nil`. On failure, the result is the zero value (0) and the error describes what went wrong.

The caller must handle both values:

```go
result, err := divide(10, 3)
if err != nil {
    fmt.Println("Error:", err)
    return
}
fmt.Println("Result:", result)
```

### Named return values

Return values can be named, which pre-declares them as local variables initialized to their zero values:

```go
func split(total int) (half, remainder int) {
    half = total / 2
    remainder = total % 2
    return    // "naked return" — returns half and remainder implicitly
}
```

- `(half, remainder int)` declares two return variables.
- The bare `return` at the end returns whatever `half` and `remainder` currently hold.
- Named returns are useful for documentation, but can reduce clarity in longer functions.

### Variadic functions (`...`)

A variadic function accepts any number of arguments of a given type:

```go
func sum(numbers ...int) int {
    total := 0
    for _, n := range numbers {
        total += n
    }
    return total
}
```

- `numbers ...int` means "zero or more int arguments." Inside the function, `numbers` is a `[]int` (a slice).
- Called like: `sum(1, 2, 3)` or `sum(1, 2, 3, 4, 5)` or even `sum()`.

You can also expand a slice into variadic args:

```go
nums := []int{1, 2, 3, 4}
total := sum(nums...)          // expand slice into individual args
```

### Functions as values and closures

Functions are first-class values in Go. You can assign them to variables, pass them as arguments, and return them:

```go
func main() {
    // Assign a function to a variable
    double := func(x int) int {
        return x * 2
    }

    fmt.Println(double(5))    // prints 10

    // Pass a function as an argument
    result := apply(3, double)
    fmt.Println(result)       // prints 6
}

func apply(x int, fn func(int) int) int {
    return fn(x)
}
```

Functions that reference variables from their surrounding scope are **closures**:

```go
func makeCounter() func() int {
    count := 0
    return func() int {
        count++           // captures 'count' from outer function
        return count
    }
}

func main() {
    counter := makeCounter()
    fmt.Println(counter())    // 1
    fmt.Println(counter())    // 2
    fmt.Println(counter())    // 3
}
```

- The inner function captures `count` — each call increments the same variable.
- `makeCounter()` returns a function value, and each call to `makeCounter()` creates a fresh `count`.

---

## 4. Structs and Methods

### Struct definitions

A struct is a collection of named fields — Go's equivalent of a class (without inheritance):

```go
type Person struct {
    Name string
    Age  int
    Email string
}

type BankAccount struct {
    Owner   string
    Balance float64
    Active  bool
}
```

- Each field has a name and a type.
- Fields are accessed with dot notation: `person.Name`, `account.Balance`.

Creating struct values:

```go
// Named fields (preferred — order doesn't matter, self-documenting)
alice := Person{
    Name:  "Alice",
    Age:   30,
    Email: "alice@example.com",
}

// Positional (fragile — breaks if you add a field later)
bob := Person{"Bob", 25, "bob@example.com"}

// Partial initialization — omitted fields get zero values
guest := Person{Name: "Guest"}   // Age = 0, Email = ""
```

### Struct tags

Struct fields can have **tags** — metadata strings in backticks. These are used by encoding libraries:

```go
type User struct {
    ID        int    `json:"id"`
    FirstName string `json:"first_name"`
    Password  string `json:"-"`              // excluded from JSON
}
```

- `json:"first_name"` tells the JSON encoder to use `"first_name"` as the key instead of `"FirstName"`.
- `json:"-"` means this field is never included in JSON output.

### Methods: pointer receivers vs value receivers

Methods are functions attached to a type via a **receiver**. There are two kinds:

**Value receiver** — gets a copy of the struct. Used when the method doesn't modify the struct:

```go
func (p Person) FullInfo() string {
    return fmt.Sprintf("%s (age %d)", p.Name, p.Age)
}
```

- `(p Person)` — `p` is a copy. Changes to `p` wouldn't affect the original.
- Call it: `alice.FullInfo()`.

**Pointer receiver** — gets a pointer to the struct. Used when the method modifies the struct:

```go
func (a *BankAccount) Deposit(amount float64) {
    a.Balance += amount
}

func (a *BankAccount) Withdraw(amount float64) error {
    if amount > a.Balance {
        return fmt.Errorf("insufficient funds")
    }
    a.Balance -= amount
    return nil
}
```

- `(a *BankAccount)` — `a` is a pointer. `Deposit` modifies the original `BankAccount` in place.
- Rule of thumb: if a method needs to mutate the receiver, use a pointer receiver. If it's read-only and the type is small, either works, but pointer receivers are conventional for consistency.

### Embedded structs (composition over inheritance)

Go has no inheritance. Instead, you **embed** one struct inside another:

```go
type Address struct {
    Street string
    City   string
    Zip    string
}

type Employee struct {
    Person           // embedded — no field name, just the type
    Address          // embedded
    Department string
}
```

- `Person` and `Address` are embedded without field names. This means all of their fields are **promoted** to `Employee`.
- You can write `emp.Name` instead of `emp.Person.Name`, and `emp.City` instead of `emp.Address.City`.
- This is Go's version of composition. `Employee` *has a* `Person` and *has an* `Address`.

### Constructor functions: the `NewXxx` pattern

Go has no constructors. By convention, you write a function called `NewXxx` that returns a pointer to an initialized struct:

```go
func NewBankAccount(owner string, initialDeposit float64) *BankAccount {
    return &BankAccount{
        Owner:   owner,
        Balance: initialDeposit,
        Active:  true,
    }
}
```

- `&BankAccount{...}` creates a struct literal and takes its address — returns a pointer.
- This is the standard Go pattern for initialization that needs validation or default values.

Another way to allocate:

```go
account := new(BankAccount)   // allocates zeroed memory, returns *BankAccount
account.Owner = "Charlie"
account.Active = true
```

- `new(BankAccount)` allocates and returns a pointer. All fields start at their zero values.
- The `&Type{...}` form is preferred when you want to set fields immediately.

---

## 5. Interfaces

### Interface definitions

An interface defines a set of method signatures. Any type that implements those methods satisfies the interface — **no explicit declaration required**.

```go
type Shape interface {
    Area() float64
    Perimeter() float64
}
```

Any struct with `Area() float64` and `Perimeter() float64` methods automatically satisfies `Shape`. There's no `implements` keyword.

### Implicit interface satisfaction

Let's define two shapes that both satisfy `Shape`:

```go
type Circle struct {
    Radius float64
}

func (c Circle) Area() float64 {
    return 3.14159 * c.Radius * c.Radius
}

func (c Circle) Perimeter() float64 {
    return 2 * 3.14159 * c.Radius
}

type Rectangle struct {
    Width, Height float64
}

func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}
```

Now you can use either type wherever a `Shape` is expected:

```go
func printShapeInfo(s Shape) {
    fmt.Printf("Area: %.2f, Perimeter: %.2f\n", s.Area(), s.Perimeter())
}

func main() {
    c := Circle{Radius: 5}
    r := Rectangle{Width: 3, Height: 4}

    printShapeInfo(c)    // works — Circle satisfies Shape
    printShapeInfo(r)    // works — Rectangle satisfies Shape
}
```

### The empty interface: `interface{}`

`interface{}` has zero methods, so *every type* satisfies it. It's Go's way of saying "any type":

```go
func printAnything(value interface{}) {
    fmt.Println(value)
}

func main() {
    printAnything(42)
    printAnything("hello")
    printAnything(true)
    printAnything([]int{1, 2, 3})
}
```

Since Go 1.18, `any` is an alias for `interface{}` — you'll see both in code:

```go
func printAnything(value any) {    // same as interface{}
    fmt.Println(value)
}
```

### Type assertions

When you have a value of interface type and need the concrete type, use a **type assertion**:

```go
var s Shape = Circle{Radius: 5}

// One-value form — panics if wrong
c := s.(Circle)
fmt.Println(c.Radius)    // 5

// Two-value form — safe, returns ok=false if wrong
c, ok := s.(Circle)
if ok {
    fmt.Println("It's a circle with radius", c.Radius)
} else {
    fmt.Println("Not a circle")
}
```

- `.( Circle)` asserts that `s` is actually a `Circle`.
- The two-value form `c, ok := s.(Circle)` is safer — `ok` is `false` if the assertion fails.

### Type switches

A type switch branches based on the concrete type of an interface value:

```go
func describe(s Shape) string {
    switch shape := s.(type) {
    case Circle:
        return fmt.Sprintf("Circle with radius %.1f", shape.Radius)
    case Rectangle:
        return fmt.Sprintf("Rectangle %0.1f x %.1f", shape.Width, shape.Height)
    default:
        return "Unknown shape"
    }
}
```

- `s.(type)` can only be used in a switch statement.
- Each `case` binds `shape` to the concrete type, so you get type-safe access to its fields.

### Interface embedding

Interfaces can embed other interfaces to compose them:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type ReadWriter interface {
    Reader     // embedded — includes all Reader methods
    Writer     // embedded — includes all Writer methods
}
```

`ReadWriter` requires all methods of both `Reader` and `Writer`. This is how Go builds up larger interfaces from smaller ones — this exact pattern is used in the standard library's `io` package.

---

## 6. Pointers

### What are pointers?

A pointer holds the memory address of a value. Go uses `*` and `&`:

- `&x` — "address of x" — creates a pointer to `x`
- `*p` — "value at p" — dereferences a pointer to get the underlying value
- `*Type` in a declaration means "pointer to Type"

### Creating pointers

```go
x := 42
p := &x              // p is a *int (pointer to int), holds the address of x

fmt.Println(p)       // prints a memory address like 0xc000012080
fmt.Println(*p)      // prints 42 — the value p points to
```

With structs:

```go
// Two ways to create a pointer to a struct:
account := &BankAccount{Owner: "Alice", Balance: 100}    // & takes the address
account2 := new(BankAccount)                              // new() returns a pointer
```

### Dereferencing

```go
x := 42
p := &x

*p = 100             // change the value at the address p points to
fmt.Println(x)       // prints 100 — x was modified through the pointer
```

With struct fields, Go automatically dereferences pointers — you don't need `(*p).Field`:

```go
type Point struct{ X, Y int }

p := &Point{X: 1, Y: 2}
p.X = 10             // Go automatically dereferences — same as (*p).X = 10
fmt.Println(p.X)     // 10
```

### Nil checks

Pointers can be `nil` (they don't point to anything). Dereferencing a nil pointer panics, so you must check:

```go
func printName(p *Person) {
    if p == nil {
        fmt.Println("No person provided")
        return
    }
    fmt.Println(p.Name)
}

func main() {
    var nobody *Person          // nil pointer — points to nothing
    printName(nobody)           // prints "No person provided"
    printName(&Person{Name: "Alice"})   // prints "Alice"
}
```

### Why pointer receivers matter

A value receiver gets a copy — changes don't affect the original. A pointer receiver gets the original — changes persist:

```go
type Counter struct {
    Value int
}

// Value receiver — gets a copy. The increment is LOST.
func (c Counter) IncrementBroken() {
    c.Value++    // modifies the copy, not the original
}

// Pointer receiver — gets the original. The increment is KEPT.
func (c *Counter) Increment() {
    c.Value++    // modifies the original
}

func main() {
    c := Counter{Value: 0}

    c.IncrementBroken()
    fmt.Println(c.Value)     // still 0!

    c.Increment()
    fmt.Println(c.Value)     // now 1
}
```

---

## 7. Error Handling

### The `error` type

In Go, `error` is a built-in interface with one method:

```go
type error interface {
    Error() string
}
```

Any type with an `Error() string` method is an error. There is no try/catch — errors are returned as values and checked explicitly.

### The `if err != nil` pattern

This is the most common pattern in Go. You'll see it in almost every function:

```go
file, err := os.Open("data.txt")
if err != nil {
    fmt.Println("Failed to open file:", err)
    return
}
// use file...
```

The flow is always: call a function, check if `err` is not nil, handle (usually by returning the error to the caller). This replaces exceptions — the error is propagated explicitly up the call stack.

A chain of operations looks like this:

```go
func loadConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, err
    }

    var config Config
    err = json.Unmarshal(data, &config)
    if err != nil {
        return nil, err
    }

    return &config, nil
}
```

### Creating errors

The simplest way to create an error:

```go
import "errors"

var ErrNotFound = errors.New("item not found")
var ErrEmpty    = errors.New("list is empty")
```

For errors with dynamic content, use `fmt.Errorf`:

```go
func withdraw(balance, amount float64) error {
    if amount > balance {
        return fmt.Errorf("cannot withdraw %.2f: only %.2f available", amount, balance)
    }
    return nil
}
```

### Sentinel errors

Package-level error variables that callers can compare against:

```go
var (
    ErrNotFound     = errors.New("not found")           // Exported — part of the API
    ErrUnauthorized = errors.New("unauthorized")        // Exported
    errInternal     = errors.New("internal error")      // unexported — internal use
)

func findUser(id int) (*User, error) {
    if id <= 0 {
        return nil, ErrNotFound
    }
    // ...
}
```

Callers check: `if err == ErrNotFound { ... }` or the preferred form:

```go
if errors.Is(err, ErrNotFound) {
    fmt.Println("User does not exist")
}
```

### Custom error types

When you need to carry extra context, define a struct that implements the `error` interface:

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Message)
}

func validateAge(age int) error {
    if age < 0 {
        return &ValidationError{Field: "age", Message: "must be non-negative"}
    }
    if age > 150 {
        return &ValidationError{Field: "age", Message: "unrealistic value"}
    }
    return nil
}
```

Callers can type-assert to get the extra fields:

```go
err := validateAge(-5)
var valErr *ValidationError
if errors.As(err, &valErr) {
    fmt.Printf("Bad field: %s\n", valErr.Field)
}
```

### Error wrapping with `%w`

`fmt.Errorf` with `%w` wraps an error, preserving the original for inspection:

```go
func loadUser(id int) (*User, error) {
    user, err := db.FindByID(id)
    if err != nil {
        return nil, fmt.Errorf("loadUser(%d): %w", id, err)
    }
    return user, nil
}
```

The caller can then unwrap it:

```go
err := loadUser(42)
if errors.Is(err, ErrNotFound) {
    // still matches, even though the error was wrapped with extra context
}
```

---

## 8. Collections: Slices, Arrays, and Maps

### Arrays vs slices

**Arrays** have a fixed size, determined at compile time:

```go
var rgb [3]int                   // array: exactly 3 ints
colors := [3]string{"red", "green", "blue"}
```

**Slices** are dynamic-length views over arrays. Most code uses slices:

```go
var names []string               // slice: length can change
scores := []int{95, 87, 92}     // slice literal
```

### Creating slices with `make`

`make([]T, length)` or `make([]T, length, capacity)`:

```go
// Length 5 — all elements start at zero value
numbers := make([]int, 5)           // [0, 0, 0, 0, 0]

// Length 0, capacity 10 — pre-allocates memory for efficiency
results := make([]string, 0, 10)    // [] but room for 10 items
```

- `make([]string, 0, 10)` — creates an empty slice with pre-allocated capacity to avoid repeated allocations during `append`.

### `append` — adding to slices

```go
fruits := []string{"apple", "banana"}
fruits = append(fruits, "cherry")
fruits = append(fruits, "date", "elderberry")    // add multiple at once

fmt.Println(fruits)
// [apple banana cherry date elderberry]
```

- `append` may grow the underlying array if capacity is exceeded, returning a new slice header.
- You must always reassign the result: `fruits = append(fruits, ...)` — without the reassignment, the new length (and possibly new backing array) is lost.

Expanding one slice into another:

```go
moreFruits := []string{"fig", "grape"}
fruits = append(fruits, moreFruits...)    // ... expands the slice
```

### Slice operations

```go
nums := []int{0, 1, 2, 3, 4, 5}

sub := nums[1:4]     // [1, 2, 3] — from index 1 up to (not including) 4
first3 := nums[:3]   // [0, 1, 2] — from start up to 3
last2 := nums[4:]    // [4, 5]    — from index 4 to end

length := len(nums)  // 6 — number of elements
cap := cap(nums)     // capacity of underlying array
```

### Maps

Maps are Go's hash tables — like dictionaries or objects in other languages.

**Declaration and initialization:**

```go
// Using make
ages := make(map[string]int)
ages["Alice"] = 30
ages["Bob"] = 25

// Using a literal
colors := map[string]string{
    "red":   "#ff0000",
    "green": "#00ff00",
    "blue":  "#0000ff",
}
```

- `map[string]int` — keys are `string`, values are `int`.
- Maps must be initialized with `make` or a literal before use. A nil map panics on write.

**Checking if a key exists:**

```go
age, ok := ages["Charlie"]
if !ok {
    fmt.Println("Charlie not found")
} else {
    fmt.Println("Charlie is", age)
}
```

The two-value map lookup `value, ok := m[key]` returns `ok == true` if the key exists. This is the idiomatic way to check membership.

**Deleting a key:**

```go
delete(ages, "Bob")    // removes the "Bob" entry
```

### `range` — iterating over collections

**Over a slice:**

```go
fruits := []string{"apple", "banana", "cherry"}
for i, fruit := range fruits {
    fmt.Printf("%d: %s\n", i, fruit)
}

// If you only need the value:
for _, fruit := range fruits {
    fmt.Println(fruit)
}

// If you only need the index:
for i := range fruits {
    fmt.Println(i)
}
```

- `range` returns `(index, value)` for slices. `_` discards whichever you don't need.

**Over a map:**

```go
for name, age := range ages {
    fmt.Printf("%s is %d years old\n", name, age)
}
```

- For maps, `range` returns `(key, value)`. Iteration order is random.

---

## 9. Control Flow

### `for` — Go's only loop

Go has no `while` or `do-while`. The `for` keyword covers all loop patterns.

**Classic 3-part loop:**

```go
for i := 0; i < 10; i++ {
    fmt.Println(i)
}
```

**Condition-only loop (like `while`).** Go has no `while` keyword — you use `for` with just a condition:

```go
n := 1
for n < 100 {
    n *= 2
}
fmt.Println(n)    // 128
```

**Infinite loop.** `for` with no condition runs forever — you exit with `return`, `break`, or a channel signal:

```go
for {
    input := readUserInput()
    if input == "quit" {
        break          // exit the loop
    }
    processInput(input)
}
```

**For-range loop:**

```go
for i, item := range mySlice {
    fmt.Printf("Item %d: %v\n", i, item)
}
```

### `if` with init statement

Go's `if` can include an initialization statement before the condition:

```go
if err := doSomething(); err != nil {
    fmt.Println("Error:", err)
    return
}
```

- `err` is scoped to this `if` block — it doesn't leak into the surrounding function.
- This is extremely common in Go for inline error handling.

Another example with map lookup:

```go
if value, ok := myMap[key]; ok {
    fmt.Println("Found:", value)
} else {
    fmt.Println("Not found")
}
```

### `switch`

**Switch on an expression:**

```go
switch day {
case "Monday":
    fmt.Println("Start of work week")
case "Friday":
    fmt.Println("Almost weekend!")
case "Saturday", "Sunday":
    fmt.Println("Weekend!")
default:
    fmt.Println("Midweek")
}
```

Note: unlike C/Java, Go `switch` cases do **not fall through** by default. Each case breaks automatically. If you *want* fallthrough, you must say it explicitly with the `fallthrough` keyword.

**Switch with no expression** — acts like a clean if/else chain:

```go
switch {
case score >= 90:
    grade = "A"
case score >= 80:
    grade = "B"
case score >= 70:
    grade = "C"
default:
    grade = "F"
}
```

### `defer`

`defer` schedules a function call to run when the surrounding function returns. Deferred calls execute in **LIFO order** (last deferred = first executed).

**Common use: close a resource**

```go
file, err := os.Open("data.txt")
if err != nil {
    return err
}
defer file.Close()    // guaranteed to run when the function returns

// ... use file ...
// file.Close() runs automatically, even if we return early due to an error
```

**Common use: unlock a mutex**

```go
mu.Lock()
defer mu.Unlock()

// ... critical section ...
// Unlock runs when function returns, even if there's a panic
```

**LIFO order with multiple defers:**

```go
func example() {
    defer fmt.Println("first deferred — runs LAST")
    defer fmt.Println("second deferred — runs SECOND")
    defer fmt.Println("third deferred — runs FIRST")
    fmt.Println("normal code")
}
// Output:
// normal code
// third deferred — runs FIRST
// second deferred — runs SECOND
// first deferred — runs LAST
```

`defer` is Go's replacement for `finally` blocks. It guarantees cleanup even if the function panics.

### `panic` and `recover`

`panic` aborts the current function and unwinds the stack. It is used for programming errors and invariant violations — not for expected error conditions:

```go
func mustPositive(n int) {
    if n <= 0 {
        panic(fmt.Sprintf("expected positive number, got %d", n))
    }
}
```

`recover` catches a panic inside a deferred function:

```go
func safeCall(fn func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered from panic: %v", r)
        }
    }()
    fn()
    return nil
}
```

- `recover()` only works inside a `defer` function — calling it elsewhere always returns `nil`.
- The pattern is: `defer func() { if r := recover(); r != nil { handle(r) } }()`. The trailing `()` immediately invokes the anonymous function (but its execution is deferred).
- Use `panic`/`recover` sparingly — idiomatic Go uses error return values for expected failures.

---

## 10. Concurrency

### Goroutines

A goroutine is a lightweight thread managed by the Go runtime. You spawn one with the `go` keyword:

```go
func sayHello(name string) {
    fmt.Printf("Hello, %s!\n", name)
}

func main() {
    go sayHello("Alice")     // runs concurrently
    go sayHello("Bob")       // runs concurrently

    time.Sleep(time.Second)  // wait for goroutines (crude — see WaitGroup below)
}
```

With an anonymous function:

```go
go func(msg string) {
    fmt.Println(msg)
}("Hello from goroutine")    // arguments are passed immediately
```

- The arguments `("Hello from goroutine")` are evaluated immediately and passed to the goroutine. This avoids a common closure bug where loop variables change before the goroutine reads them.
- Goroutines are extremely cheap — thousands can run simultaneously.

### Channels

Channels are typed pipes for communication between goroutines.

**Unbuffered channels** — `make(chan T)` with no size argument. The sender blocks until a receiver is ready:

```go
ch := make(chan string)

go func() {
    ch <- "hello"    // send: blocks until someone receives
}()

msg := <-ch          // receive: blocks until someone sends
fmt.Println(msg)     // "hello"
```

- `ch <- value` sends a value into the channel.
- `value := <-ch` receives a value from the channel.
- The arrow always points left — think of it as "data flows into the arrow."

**Buffered channels** — `make(chan T, capacity)` with a size. The sender blocks only when the buffer is full:

```go
ch := make(chan int, 3)    // buffer size 3

ch <- 1    // doesn't block — buffer has room
ch <- 2    // doesn't block
ch <- 3    // doesn't block
// ch <- 4  // WOULD block — buffer is full

fmt.Println(<-ch)    // 1
fmt.Println(<-ch)    // 2
```

**Signal-only channels** — `chan struct{}` carries no data. Used purely for notification:

```go
done := make(chan struct{})

go func() {
    // ... do work ...
    close(done)           // signal that work is done
}()

<-done    // wait for the signal
fmt.Println("Work finished")
```

- `struct{}` takes zero bytes, so the channel carries no data — it's just a signal.
- `close(done)` unblocks all receivers.

### `select`

`select` lets a goroutine wait on multiple channel operations simultaneously. It is like a `switch` but for channels:

```go
func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        time.Sleep(1 * time.Second)
        ch1 <- "one"
    }()
    go func() {
        time.Sleep(2 * time.Second)
        ch2 <- "two"
    }()

    select {
    case msg := <-ch1:
        fmt.Println("Received from ch1:", msg)
    case msg := <-ch2:
        fmt.Println("Received from ch2:", msg)
    }
}
// Prints "Received from ch1: one" (it arrives first)
```

Walking through this:

- `case msg := <-ch1:` — if a value arrives on `ch1`, receive it and run this branch.
- `case msg := <-ch2:` — if a value arrives on `ch2`, receive it and run this branch.
- `select` blocks until **one** of the cases can proceed. If multiple are ready, one is chosen at random.

**Timeout pattern:**

```go
select {
case result := <-ch:
    fmt.Println("Got result:", result)
case <-time.After(3 * time.Second):
    fmt.Println("Timed out!")
}
```

**Non-blocking check** — adding a `default` case makes `select` non-blocking:

```go
select {
case msg := <-ch:
    fmt.Println("Received:", msg)
default:
    fmt.Println("No message available")
}
```

### `sync.Mutex` — mutual exclusion

A mutex protects shared data from concurrent access:

```go
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}
```

`sync.RWMutex` allows multiple concurrent readers but only one writer:

```go
type SafeMap struct {
    mu   sync.RWMutex
    data map[string]string
}

func (m *SafeMap) Get(key string) string {
    m.mu.RLock()              // multiple goroutines can RLock simultaneously
    defer m.mu.RUnlock()
    return m.data[key]
}

func (m *SafeMap) Set(key, value string) {
    m.mu.Lock()               // exclusive access for writing
    defer m.mu.Unlock()
    m.data[key] = value
}
```

### `sync.WaitGroup` — waiting for goroutines

A `WaitGroup` waits for a collection of goroutines to finish:

```go
func main() {
    var wg sync.WaitGroup

    for i := 0; i < 5; i++ {
        wg.Add(1)              // increment counter before launching goroutine
        go func(id int) {
            defer wg.Done()    // decrement counter when goroutine finishes
            fmt.Printf("Worker %d done\n", id)
        }(i)
    }

    wg.Wait()                  // blocks until counter reaches 0
    fmt.Println("All workers finished")
}
```

- `wg.Add(1)` — tell the WaitGroup one more goroutine is starting.
- `wg.Done()` — tell the WaitGroup one goroutine is finished (decrements by 1).
- `wg.Wait()` — block until all goroutines have called `Done()`.

### `sync.Once` — one-time initialization

Ensures a function runs exactly once, even from multiple goroutines:

```go
var (
    instance *Database
    once     sync.Once
)

func GetDatabase() *Database {
    once.Do(func() {
        instance = connectToDatabase()    // runs only on the first call
    })
    return instance
}
```

### Common concurrency pattern: worker pool

```go
func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)

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
        results <- job * 2     // do some "work"
    }
}
```

- `jobs <-chan int` — a receive-only channel (this function can only read from it).
- `results chan<- int` — a send-only channel (this function can only write to it).
- Channel direction in the type signature makes the intent clear and prevents misuse.

---

## 11. Testing

### Test file conventions

Test files are named `*_test.go` and live in the same directory as the code they test. Go's test runner finds them automatically.

```
math/calculator.go          // production code
math/calculator_test.go     // tests for calculator.go
```

### Test functions

Test functions must start with `Test` and take `*testing.T`:

```go
// math/calculator_test.go
package math

import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    if result != 5 {
        t.Errorf("Add(2, 3) = %d; want 5", result)
    }
}

func TestDivide(t *testing.T) {
    result, err := Divide(10, 2)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if result != 5 {
        t.Errorf("Divide(10, 2) = %f; want 5", result)
    }
}
```

- `t.Errorf(...)` reports a failure but continues running the test.
- `t.Fatalf(...)` reports a failure and stops the test immediately.

Run tests with `go test ./math/...`.

### Table-driven tests

The most common test pattern in Go — define a slice of test cases, iterate with `range`:

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        a, b     int
        expected int
    }{
        {1, 2, 3},
        {0, 0, 0},
        {-1, 1, 0},
        {100, -50, 50},
    }

    for _, tt := range tests {
        result := Add(tt.a, tt.b)
        if result != tt.expected {
            t.Errorf("Add(%d, %d) = %d; want %d", tt.a, tt.b, result, tt.expected)
        }
    }
}
```

- `[]struct{...}{...}` — a slice of anonymous structs, defined and initialized inline.
- Adding a new test case is just one more line in the slice.

**Subtests** allow individual test cases to have names:

```go
func TestDivide(t *testing.T) {
    tests := []struct {
        name      string
        a, b      float64
        want      float64
        wantErr   bool
    }{
        {"normal", 10, 2, 5, false},
        {"divide by zero", 10, 0, 0, true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Divide(tt.a, tt.b)
            if (err != nil) != tt.wantErr {
                t.Errorf("error = %v, wantErr %v", err, tt.wantErr)
                return
            }
            if got != tt.want {
                t.Errorf("Divide(%f, %f) = %f; want %f", tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

- `t.Run(name, func)` creates a named subtest — the output shows which case failed.

### Benchmark functions

Benchmark functions start with `Benchmark` and take `*testing.B`:

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(100, 200)
    }
}
```

- `b.N` is set by the test framework to get stable timing. Run benchmarks with `go test -bench=. ./math/...`.

---

## 12. Common Patterns in Go

### `init()` functions

`init()` runs automatically when a package is loaded — before `main()`. It takes no arguments and returns nothing:

```go
package main

var config map[string]string

func init() {
    config = map[string]string{
        "host": "localhost",
        "port": "8080",
    }
}

func main() {
    fmt.Println(config["host"])    // "localhost" — init() already ran
}
```

A package can have multiple `init()` functions. They all run in the order they appear. This is used for registering drivers, setting defaults, and one-time setup.

### Stringer interface

If you define a `String() string` method on a type, `fmt.Println` and friends will use it automatically:

```go
type Direction int

const (
    North Direction = iota
    East
    South
    West
)

func (d Direction) String() string {
    names := [...]string{"North", "East", "South", "West"}
    if d < North || d > West {
        return "Unknown"
    }
    return names[d]
}

func main() {
    fmt.Println(North)    // prints "North", not "0"
}
```

### The comma-ok idiom

Several Go operations use the two-value return to signal success:

```go
// Map lookup
value, ok := myMap[key]

// Type assertion
str, ok := value.(string)

// Channel receive
msg, ok := <-ch           // ok is false when channel is closed and empty
```

### Functional options pattern

When a function needs many optional parameters, Go uses functional options:

```go
type Server struct {
    host    string
    port    int
    timeout time.Duration
}

type Option func(*Server)

func WithPort(port int) Option {
    return func(s *Server) {
        s.port = port
    }
}

func WithTimeout(d time.Duration) Option {
    return func(s *Server) {
        s.timeout = d
    }
}

func NewServer(host string, opts ...Option) *Server {
    s := &Server{
        host:    host,
        port:    8080,               // default
        timeout: 30 * time.Second,   // default
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Usage:
srv := NewServer("localhost", WithPort(9090), WithTimeout(5*time.Second))
```

### The `sync.Pool` pattern

For objects that are frequently allocated and discarded, `sync.Pool` recycles them to reduce garbage collection pressure:

```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func process(data []byte) string {
    buf := bufferPool.Get().(*bytes.Buffer)    // get a recycled buffer (or new)
    defer bufferPool.Put(buf)                   // return it when done
    buf.Reset()                                 // clear previous content

    buf.Write(data)
    return buf.String()
}
```

### Graceful shutdown pattern

A common pattern for long-running services that need to shut down cleanly:

```go
func main() {
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt, syscall.SIGTERM)

    srv := startServer()

    <-quit    // block until we receive a shutdown signal
    fmt.Println("Shutting down...")

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    srv.Shutdown(ctx)
}
```

---

## Quick Reference <a id="quick-reference"></a>

| Go | What it means |
|----|--------------|
| `:=` | Declare + assign (type inferred) |
| `_` | Blank identifier — discard a value |
| `&x` | Address of x (creates pointer) |
| `*p` | Value pointed to by p (dereference) |
| `*Type` | Pointer to Type |
| `[]Type` | Slice of Type |
| `[N]Type` | Array of N elements |
| `map[K]V` | Map from K to V |
| `chan T` | Channel carrying values of type T |
| `<-chan T` | Receive-only channel |
| `chan<- T` | Send-only channel |
| `interface{}` / `any` | Any type |
| `go f()` | Run f() in a new goroutine |
| `defer f()` | Run f() when this function returns |
| `x.(Type)` | Type assertion |
| `x.(type)` | Type switch (in switch only) |
| `Uppercase` | Exported (public) |
| `lowercase` | Unexported (package-private) |
| `...Type` | Variadic parameter |
| `slice...` | Expand slice into individual args |
| `make(T, ...)` | Allocate and initialize slice/map/channel |
| `new(T)` | Allocate zeroed T, return pointer |
