# Language

If you're interested in learning the syntax of `Fermion3` you've come to the right place!

`Fermion3` is an imperative dynamically typed object-based language that compiles to Quark3 assembly and Lepton3 bytecode. 

It is designed to be simple and nice to write, with types as first-class values, runtime guards, immutable arrays and objects, and macros.

This file serves not as a specification of the language but as an introduction to it.

## Comments

There is only a simple single way to write a comment, which begins with a `//`.

```
// Single line comment
```

## Literals

```
// Integer literals
42
-7
0

// Hex
0xFFFF

// Binary
0b0111

// Octal
0o1010

// Floating point literals
3.14
-0.05
1.0
.05
1.
3.15e10

// Boolean literals
true
false

// Unit literal
()

// String literals
"hello"
"world"

// String interpolation
"hello ${name}, you are ${age} years old"

// Array literals
[1, 2, 3]
[]

// Untyped object literals
// These cannot be used directly as they
// are untyped/unguarded as explained below
{ x: 1.0, y: 2.0 }
```

## Functions

```
// A simple function
fn add(a, b) => a + b

// Multiline functions, the last
// value of the function is returned
fn mul(a, b) => { 
    a * b 
}

// The above is the same as
fn mul(a, b) => { 
    return a * b 
}


// A closure
let double = fn(x) => x * 2 

// Functions are values and can be passed around
fn apply(f, x) => f(x)

// Evaluates to 10
apply(double, 5)
```

## Closures

```
// closures capture their surrounding scope
fn make_adder(n) => fn(x) => x + n

let add5 = make_adder(5)
add5(3)    // 8
```

## Function Calls

```
// Functions are normally called using parentheses.
add(1, 2)

// Functions taking two arguments may also be called using infix notation.
1 add 2
```

Chaining multiple infix calls is not permitted without parenthesis for precedence (same for `+, -, *, /` and stuff since they're also a function!)

```
// Illegal!
1 add 2 mul 3

// Allowed!
(1 add 2) mul 3
```

## Bindings

```
// A immutable binding (default is immutable)
let x = 5
let name = "Alice"

// Mutable local binding
let mut counter = 0
counter = counter + 1

// Block expressions, the last value is the actual bound value
let sum = {
    let a = 1
    let b = 1
    a + b
}
```

Immutable bindings cannot be rebound. 

Mutable bindings can be rebound but only within the scope they are declared in.

## Type Definitions

```
// A type can be a "guard" which narrows down some concrete type using a closure that accepts the value
type Positive = Int where (fn(x) => x > 0)

// An object (product type)
type Point = object {
    x: Float,
    y: Float
}

// An object type with a guard
type UnitVector = Point where (
    fn(v) => {
        let mag = sqrt(v.x * v.x + v.y * v.y)
        abs(mag - 1.0) < 0.0001
    }
)

// We can also do this inline by combining object and where
type UnitVector = object {
    x: Float,
    y: Float
} where (
    fn(v) => {
        let mag = sqrt(v.x * v.x + v.y * v.y)
        abs(mag - 1.0) < 0.0001
    }
)

// Types are values, so we can define a "parametric type" which is a type constructed with parameters
type Bounded(T, low, high) = T where (
    fn(x) => { x >= low && x <= high } 
)

// Using a parametric type, this Byte is an alias
type Byte = Bounded(Int, 0, 255)

// We can also guard this alias
type NonZeroByte = Byte where (fn(x) => x != 0)

// A parametric object type
type Pair(A, B) = object {
    first: A,
    second: B
}

// An enum (sum type)
type Direction = enum {
    North,
    South,
    East,
    West
}

// Enum with variants that carry data
// The data carrying variants are essentially objects and
// object the same syntax
type Shape = enum {
    Circle { radius: Float },
    Rectangle { width: Float, height: Float},
    Point
}

// Parametric enums
type Option(T) = enum {
    Some { value: T },
    None
}

// "Nameless" enum variants that carry data
// also exist which essentially assign each field
// in the order of definition to an index
type Result(T, E) = enum {
    Ok(T),
    Err(E)
}

// This is inherently equal to
type Result(T, E) = enum {
    Ok { 0: T },
    Err { 0: E }
}
```

Types are values. `type X = ...` at the top level binds the name X to a type value, exactly like let binds a name to a regular value.

## Methods

```
// Methods can be attached to an object in a `with` block
type Point = object {
    x: Float,
    y: Float
} with {

    // This is essentially a constructor
    fn origin() => Point { x: 0.0, y: 0.0 }

    // Distance to another point
    fn distance_to(self, other) =>
        sqrt((self.x - other.x)^2 + (self.y - other.y)^2)

    // Translate this point by some amount..
    fn translate(self, dx, dy) => {
        Point {
            ...self,
            x: self.x + dx,
            y: self.y + dy
        }
    }
}

// With a guard, the guard comes before the methods
type UnitVector = object {
    x: Float,
    y: Float
} where (
    fn(v) => abs(sqrt(v.x^2 + v.y^2) - 1.0) < 0.0001
) with {
    fn dot(self, other) =>
        self.x * other.x + self.y * other.y

    fn flip(self) => UnitVector { ...self, x: -self.x, y: -self.y }
}

// Enums can also have methods
type Shape = enum {
    Circle { radius: Float },
    Rectangle { width: Float, height: Float },
    Point
} with {
    fn area(self) => match self {
        Shape.Circle { radius } => pi * radius * radius,
        Shape.Rectangle { width, height } => width * height,
        Shape.Point, => 0.0
    }
}

// We can also add methods to an alias
// This is the byte type from earlier but with the `with` block
// This keeps all the methods from bounded 
type Byte = Bounded(Int, 0, 255) with {
    fn to_hex(self) => "0x${format_hex(self)}"
    fn to_binary(self) => "0b${format_binary(self)}"
}
```

## Guard Annotations

Fermion3 is dynamically typed, however values may optionally be guarded using type annotations. 

This lets us be confident about the values used without needing to manually convert to the type in every case.

```
// Parameter annotations 
fn add(a: Int, b: Int) => a + b

// Return annotation 
fn add(a: Int, b: Int) -> Int => a + b

// Local bindings 
let x: Positive = 5

// Parametric types in the parameters
type Bounded(T: Positive, low, high) = T where (
    fn(x) => { x >= low && x <= high } 
)
```

Annotations will automatically resolve such that on binding to the thing, say a local name or a function parameter or a parametric type, the guard is checked against the value.

Return annotations are checked when a function completes, such that the result must satisfy the declared return type before being returned to the caller.

**These mechanisms are all at runtime.**

## Type Casts

Values may be explicitly validated against a type using the as operator.

```
type Positive = Int where (
    fn(x) => x > 0
)

// Automatically runs the guard for validation here
let x = 42 as Positive

// This would fail!
let y = -42 as Positive

// Since types are values, we can also do that
let T = Positive 
let value = 42 as T
```

## Object Construction

Object literals are untyped and cannot be used directly since they do not refer to an object "schema"/type definition

```
// ??? What is this
{ x: 1.0, y: 2.0 }
```

To construct a typed object, the target type must be specified.

```
let my_point = Point {
    x: 1.0,
    y: 2.0
}
```

This is sugar for writing out

```
let my_point = {
    x: 1.0,
    y: 2.0
} as Point
```

## Object Field Accessors

A field can be accessed by name in a "get" operation

```
let x = my_point.x
```

## Object Updating

To update an object, there is no direct method for mutating an object. Instead we produce a new object of the same kind with certain fields updated.

This is done through spreading an object in the object literal:

```
SomeObject {
    ...existing,
    field: new_value
}
```

This desugars in the same way as object construction does. The guard then runs on the new object

## Array Construction

Arrays are immutable. All operations that would modify an array produce a new array. The original is unaffected.

```
// Literal
let xs = [1, 2, 3]

// Access by index (warn: will raise an out of bounds!!)
let first = xs[0]

// Prepend (produces new array)
let ys = [0, ...xs]          // [0, 1, 2, 3]

// append (produces new array)
let zs = [...xs, 4]          // [1, 2, 3, 4]

// spread in array literal
let a = [1, 2]
let b = [3, 4]
let c = [...a, ...b]         // [1, 2, 3, 4]
```

## If Control Flow

```
// Expression form (both branches required)
let result = if x > 0 then "positive" else "non-positive"

// Block form
if x > 0 then {
    do_something()
    "positive"
} else {
    "non-positive"
}

// Else-if exists for when that's needed.
if x > 0 then "positive"
else if x < 0 then "negative"
else "zero"
```

## Match statements

```
// Match on values
match x {
    0 => "zero",
    1 => "one",
    n => "other: ${n}"
}

// Match with extra if on the type
// _ is a wildcard, for the last one if you want
// to match anything just supply an identifier like above
match x {
    n if n > 0 => "positive",
    n if n < 0 => "negative",
    _ => "zero"
}

// Match on objects
match point {
    Point { x: 0.0, y } => "on y axis at ${y}",
    Point { x, y }      => "at ${x}, ${my_y}"
}

// Match on arrays
match xs {
    []              => "empty",
    [x]             => "one element: ${x}",
    [x, y]          => "two elements",
    [head, ...tail] => "head is ${head}"
}

// Match on enums with tags
match result {
    Result.Ok(value) => value,
    Result.Err(e)    => raise e
}

// Match on enums with objects similar in the same way as Object matching
match result {
    Shape.Circle { radius } => radius,
    Shape.Rectangle { width, height} => width * height,
    Shape.Point => 0.0
}

// Match with block body
match n {
    0 => {
        log("got zero")
        "zero"
    },
    n => "other"
}
```

## Loops

```
// while loop, used for boolean condition
let mut i = 0
while i < 10 {
    i = i + 1
}

// for loop, used for an access to each element
// in a collection
for x in [1, 2, 3] {
    log(x)
}

// An infinite loop
loop {
    let x = compute()
}

// Loops can be broken out of with the "break" keyword similar to other languages, which returns a value, an empty break is therefore a break on the unit type
// When no value is returned by the loop the () unit value is bound
let result = loop {
    break 5
}

// The block binding still applies, the last value computed in the last iteration of the loop is the value that result is bound to here, which would be log(3)
// If the list is empty here since no value is returned, result is bound to () 
let result = for x in [1,2,3] {
    log(x)
}
```

## Raising Errors

```
// A single value can be raised as an error
raise "something went wrong"
raise Result.Err("bad input")
raise { code: 404, message: "not found" }
```

## Guard Errors

A `fail` clause can be attached to a type with a guard, which is the error message returned when the guard fails for a type.

```
type Probability = Float
    where (fn(x) => x >= 0.0 && x <= 1.0)
    fail (fn(x) => "expected a Float between 0.0 and 1.0 but got ${x}")
```

## Try/Catch

```
// Catch and handle, same block binding possible
// anything in try that raises will be caught.
let result = try {
    divide(10, 0)
} catch e {
    log("error: ${e}")
    -1
}
```

## Tags

```
// Create a fresh unique tag (different every time)
let my_tag = tag()

// check tag equality
let a = tag()
let b = a       // same identity
a == b          // true

let c = tag()
a == c          // false

// inspect the type tag of any value
let t = type_of(42)       // returns the Int type tag
let t = type_of("hello")  // returns the String type tag
let t = type_of(p)        // returns Point's unique tag if p is a Point

// An enum is a sum type, each variant has its own tag
let t = type_of(Shape.Circle { radius: 0.5 }) // returns the Circle type tag
```

## Modules

Modules are the mechanism for importing different files and components into other files in `Fermion3`. The basic idea is that the public elements of a file are exported as an object.

```
// math.f3

// All declarations are private by default, prefix with "pub" to declare it as public and exported
pub fn add(x, y) => x + y
pub fn mul(x, y) => x * y

// A public binding
pub let pi = 3.14159

// Public type
pub type PubInt = Int

// Not public! wont be accessible
fn my_private(x) => x
```

Then in another file we can use the object created by main.f3

```
// main.f3
import math

let result = math.add(1, 2)
let area = math.pi * r * r

// import with alias
import math as m
let result = m.add(1, 2)
```

## Macros

Macros are hygenic and operate on the AST. They are defined with `macro fn` (`pub macro fn` for exported macros as part of  a module) and invoked with `@`.

```
// a macro is a function from AST to AST
macro fn swap(a, b) => '{
    let tmp = $(a)
    $(a) = $(b)
    $(b) = tmp
}
```

The quote operator `'(...)` or `'{ ... }` produces an AST value. The splice operator `$(...)` inserts a value into a quoted AST.

```
// Invoke a macro using @ with the function name, this is at compile time rather than at runtime
let mut x = 1
let mut y = 1
@swap(x, y)

// This will generate code that looks like this:
let mut x = 1
let mut y = 2
let tmp = x
x = y
y = tmp
```

## Hygiene

By default, names introduced inside a macro are invisible to the call site. The tmp in swap above will never clash with a tmp in the caller's scope.

To intentionally export a name to the call site, prefix it with `#`:

```
// Anaphoric If, essentially "it" can be referred to
macro fn aif(cond, then_branch, else_branch) = '{
    let #it = $(cond)
    if #it then $(then_branch) else $(else_branch)
}

// caller can refer to 'it'
@aif(find_user(id),
    log("found: ${it}"),
    log("not found")
)

// This expands to:
let it = find_user(id)
if it then log("found: ${it}") else log("not found")
```

## Pattern Matching in Macros

```
// This will match on the AST (notice we are not generating a quoted thing) and produce the output at compile time
macro fn optimise_add(expr) = match expr {
    '($(a) + 0) => a,
    '(0 + $(b)) => b,
    other       => other
}
```

## Pipeline Operator

```
x |> f          // equivalent to f(x)
x |> f(y)       // equivalent to f(x, y)

// chaining,
// this desugars to
// length(filter(map([1, 2, 3], fn(x) => x * 2), fn(x) => x > 2))
[1, 2, 3]
    |> map(fn(x) => x * 2)
    |> filter(fn(x) => x > 2)
    |> length
```

## Is

The `is` operator checks whether a value belongs to a type.

```
// check against a primitive type
42 is Int           // true
42 is Float         // false

// check against a guarded type — runs the guard, returns false on failure
42 is Positive      // true
-5 is Positive      // false, does not raise

// check against an object type
let p = Point { x: 1.0, y: 2.0 }
p is Point          // true

// check against the whole enum type
let s = Shape.Circle { radius: 5.0 }
s is Shape          // true

// check against a specific variant
s is Shape.Circle   // true
s is Shape.Rectangle // false

// Types can also be checked
Shape.Circle is Shape // true
```

`is` differs from `as` in that `as` raises on failure while `is` returns false.

## Protocols

There is a powerful function called `has_method` that will come in handy for type guards!

```
// has_method checks that a value has a method of a given name attached to it (say in an object)
// The third parameter is the number of arguments the method takes excluding `self`
has_method(x, "to_string", 0)   // has to_string taking no arguments
has_method(x, "compare", 1)     // has compare taking one argument

// For a static method with no `self`, the third parameter is just the literal argument count with nothing excluded.
```

We can use this super powerful function as part of guards and aliases in a similar fashion to type classes in other languages (though this is all at runtime)! I like to call these ***Protocols***.

```
type Displayable = Any where (
    fn(x) => has_method(x, "to_string", 0)
)

type Comparable = Any where (
    fn(x) => has_method(x, "compare", 1)
)
```

Any type that has the required methods automatically satisfies the protocol. 

These protocols have some sugar that helps with their construction

```
protocol Comparable {
    fn compare(self, other)
}

protocol Displayable {
    fn to_string(self)
}

protocol Constructable {
    fn new()
    fn clone(self)
}

// These desugar to
type Displayable = Any where (
    fn(x) => has_method(x, "to_string", 0)
)

type Comparable = Any where (
    fn(x) => has_method(x, "compare", 1)
)

type Constructable = Any where (
    fn(x) => has_method(x, "new", 0) && has_method(x, "clone", 0)
)

// A type satisfies a protocol if it has the methods
type Point = object {
    x: Float,
    y: Float
} with {
    fn new() => Point { x: 0.0, y: 0.0 }
    fn clone(self) => Point { ...self }
}

Point is Constructable   // true
```