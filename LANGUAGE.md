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

## Shadowing

A binding can be shadowed by a new binding of the same name in an inner scope. This is because the *outer binding* won't be affected.

```
let x = 5
{
    let x = 10
    print(x)    // 10
}
print(x)        // 5, outer x is unchanged

// Shadowing a mutable binding does not make the shadow mutable
let mut counter = 0
{
    let counter = 100       // immutable shadow
    counter = counter + 500 // not allowed !
}
print(counter)    // 0, outer counter unchanged
```

Shadowing is not mutation. The outer binding is never touched, and the shadow disappears when its scope ends.

## Type Definitions

```
// There are many primitive types in Fermion3, some of the most important are:
Float
Int
Bool
Any (any type!)
// .... and others

// We can make new type aliases for existing types
type MyInt = Int

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
        let mag = sqrt(v.x**2 + v.y**2)
        abs(mag - 1.0) < 0.0001
    }
)

// We can also do this inline by combining object and where
type UnitVector = object {
    x: Float,
    y: Float
} where (
    fn(v) => {
        let mag = sqrt(v.x**2 + v.y**2)
        abs(mag - 1.0) < 0.0001
    }
)

// Types are values, so we can define a "parametric type" which is a type constructed with parameters
type Bounded<T, low, high> = T where (
    fn(x) => { x >= low && x <= high } 
)

// Using a parametric type, this Byte is an alias
type Byte = Bounded<Int, 0, 255>

// We can also guard this alias
type NonZeroByte = Byte where (fn(x) => x != 0)

// A parametric object type
type Pair<A, B> = object {
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

    // Or just a variant with no data
    Point
}

// Parametric enums
type Option<T> = enum {
    Some { value: T },
    None
}
```

Types are values. `type X = ...` at the top level binds the name X to a type value, exactly like let binds a name to a regular value.

## Function Types

As functions are values, they also have types.

```
// A function taking an Int and returning an Int
Int -> Int

// A function taking two arguments
(Int, Int) -> Int

// A function taking no arguments
() -> Int

// Nested function types (a function returning a function)
Int -> (Int -> Int)
```

These can be used where we expect types too

```
// A type that expects a predicate function
type Predicate<T> = T -> Bool
```

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
        sqrt((self.x - other.x)**2 + (self.y - other.y)**2)

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
    fn(v) => abs(sqrt(v.x**2 + v.y**2) - 1.0) < 0.0001
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
type Byte = Bounded<Int, 0, 255> with {
    fn to_hex(self) => "0x${format_hex(self)}"
    fn to_binary(self) => "0b${format_binary(self)}"
}

type NonZeroByte = Byte where (fn(x) => x != 0) with {
    fn to_hex(self) -> String => "0x${format_hex(self)} (nonzero)"

    // Cast self to the parent alias to call its version of an overridden method
    //
    // Methods are inherited from parent aliases in the chain. 
    // The most derived alias wins when a method name conflicts.
    fn to_hex_plain(self) -> String => (self as Byte).to_hex()
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
type Bounded<T: Positive, low, high> = T where (
    fn(x) => { x >= low && x <= high } 
)

// Type parameters are in scope inside a with block, 
// so they can be used as annotations on methods just like any other type.
type Pair<A, B> = object {
    first: A,
    second: B
} with {
    fn swap(self) -> Pair<B, A> =>
        Pair { first: self.second, second: self.first }

    fn map_first(self, f: A -> B) -> Pair<B, B> =>
        Pair { first: f(self.first), second: self.second }
}
```

Annotations will automatically resolve such that on binding to the thing, say a local name or a function parameter or a parametric type, the guard is checked against the value.

Return annotations are checked when a function completes, such that the result must satisfy the declared return type before being returned to the caller.


**These mechanisms are all at runtime.**

## Type Universes and Chunking

Each value is typed with the types stored next to the value in bit chunks.

Whenever a value is passed into a context which checks its type, the validator will check if the type's bit is in the bit chunks of the value.

Whenever a type is casted using the guard with annotations or "as" and the type is not already in the bit chunks, the bit for the type will be set.

The full bit array is inherently then a universe of types. Each concrete defined type gets a bit.

This is highly applicable to immutable data which can simply continue gaining type bits without needing to be rechecked on every safe cast. This applies even across function call boundaries (passing a value into a function with a new type check will assign the type bit to all references to the same typed value).

Mutable data cannot guarantee the new data meets all the types in the chunks. Therefore on reassigning a mutable binding, only the direct annotated or casted type will be checked/kept.

## Parametric types in functions

Functions can also have types as parameters similar to how parametric types have types as parameters.

```
// A type as a parameter to a function, in the same syntax as parametric types
fn identity<T>(x: T) -> T => x

// Assume some cons cell type with checks...
type List<T> = ...

// We can also constrain the type in the parametric types similar to normal type definitions
fn sort<T: Comparable>(xs: List<T>) -> List<T> => ...

// Closures can also have parametric types in the same way
let i = fn<T>(x: T) -> T => x
```

This lets you enforce relationships between types, in the case of `first` we ensure that the return type of first is the same as the element type of the input array.

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
let result = if x > 0 "positive" else "non-positive"

// Block form
if x > 0 {
    do_something()
    "positive"
} else {
    "non-positive"
}

// Else-if exists for when that's needed.
if x > 0 "positive"
else if x < 0 "negative"
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
// Untyped object literals cannot be used directly.
// A field can be rebound to another name on matching.
match point {
    Point { x: 0.0, y } => "on y axis at ${y}",
    Point { x, y: my_y }      => "at ${x}, ${my_y}"
}

// Match on arrays
match xs {
    []              => "empty",
    [x]             => "one element: ${x}",
    [x, y]          => "two elements",
    [head, ...tail] => "head is ${head}"
}

// Match on enums with tags
type Result<T, E> = enum {
    Ok { value: T },
    Err { error: E }
}

match result {
    Result.Ok { value }  => value,
    Result.Err { error }    => raise error
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
        print("got zero")
        "zero"
    },
    n => "other"
}
```

## Loops

```
// While loop, used for boolean condition
let mut i = 0
while i < 10 {
    i = i + 1
}

// For loop, used for an access to each element
// in a collection
for x in [1, 2, 3] {
    print(x)
}

// An infinite loop, which can be broken out of with the break keyword
loop {
    let x = compute()
    if x == 37 {
        break
    }
}

// Values can be returned from a loop using the break keyword
// When no break is reached, or break is used without a value,
// the () unit value is returned
let result = loop {
    break 5
}

// The same applies to for and while loops
let result = for x in [1, 2, 3] {
    if x == 2 {
        break x // result is 2
    }
}


// If no break is reached, the loop returns the last value
// produced by the block in its final iteration
// Here the block's last expression is x, so result is 3
let result = for x in [1, 2, 3] {
    x
}

// If the block's last expression doesn't always produce a value,
// the result may be () even if the loop ran
// Here the block ends in an if with no else, which is ()
// when the condition isn't met, so result is ()
let result = for x in [1, 2, 3] {
    if x == 99 {
        break x
    }
}

// An empty collection also produces ()
let result = for x in [] {
    x
}
```

## Raising Errors

```
// A single value can be raised as an error
raise "something went wrong"
raise Result.Err { error: "bad input" }
```

## Guard Errors

A `fail` clause can be attached to a type with a guard, which is the error message returned when the guard fails for a type.

```
type Probability = Float
    where (fn(x) => x >= 0.0 && x <= 1.0)
    fail (fn(x) => "expected a Float between 0.0 and 1.0 but got ${x}")
```

When a type is an alias over another, all guards in the chain run in order from the outermost parent down to the most derived type. Validation short-circuits on the first failure, so if a parent guard fails the derived guards are never checked. This means fail messages will always come from the most relevant failing guard:

```
type Bounded<T, low, high> = T where (
    fn(x) => x >= low && x <= high
) fail (fn(x) => "expected value between ${low} and ${high} but got ${x}")

type Byte = Bounded<Int, 0, 255>

type NonZeroByte = Byte where (
    fn(x) => x != 0
) fail (fn(x) => "expected a non-zero Byte but got ${x}")

// Fails with the Bounded fail message, NonZeroByte guard never runs
let a = 300 as NonZeroByte

// Fails with the NonZeroByte fail message, Bounded guard passes
let b = 0 as NonZeroByte
```

## Try/Catch

```
// Catch and handle, same block binding possible
// anything in try that raises will be caught.
let result = try {
    divide(10, 0)
} catch e {
    print("error: ${e}")
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
// "it" can be referred to outside of this macro
macro fn aif(cond, then_branch, else_branch) => '{
    let #it = $(cond)
    if #it $(then_branch) else $(else_branch)
}

// caller can refer to 'it'
@aif(find_user(id),
    print("found: ${it}"),
    print("not found")
)

// This expands to:
let it = find_user(id)
if it print("found: ${it}") else print("not found")
```

## Pattern Matching in Macros

```
// This will match on the AST (notice we are not generating a quoted thing) and produce the output at compile time
macro fn optimise_add(expr) => match expr {
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

// You can also use the _ placeholder for the value
x |> f(_)

"hello" |> _.upper()
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

We can use this super powerful function as part of guards. I like to call these ***Protocols***.

```
type Displayable = Any where (
    fn(x) => has_method(x, "to_string", 0)
)

type Comparable = Any where (
    fn(x) => has_method(x, "compare", 1)
)
```

## Capabilities

Capabilities are the mechanism for foreign function interfacing in `Lepton3`, which can call rust code directly from an instruction.

Using the builtin variadic function `cap` these can be called, the first argument is the capability no, the rest are the arguments to the capability. An example with file system based capabilities:

```
let FILE_OPEN  = 1

// An "open" file capability wrapper to ensure safe typing
pub fn open(path: String, mode: String) -> Option<FileDescriptor> => {
    try {
        // The cap function automatically handles pushing things on the stack for the capability handler to take
        Option.Some { value: cap(FILE_OPEN, path, mode) as FileDescriptor }
    } catch _ {
        Option.None
    }
}
```

## Destructuring

Destructuring lets you unpack values into bindings directly, using the same patterns available in match. It works anywhere a binding is introduced.

In let bindings:

```
// Object destructuring, similar to match an untyped object literal cannot be used directly
let Point { x, y } = my_point

// Rename a field on extraction
let Point { x: px, y: py } = my_point    // binds px and py

// Only want a certain number of fields?
let Point { x, _ } = my_point    // only binds x

// Array destructuring
let [first, second] = xs
let [head, ...tail] = xs

// Nested
let Point { x, y } = my_point
let Pair { first: Point { x, y }, second } = my_pair

// Enum variants
let Option.Some { value } = maybe
```

These will check the type during bindings and will error if it does not match in the same way as an `as`.

In function parameters:

```
// Destructure directly in the parameter list
fn magnitude(Point { x, y }) => sqrt(x**2 + y**2)

// Mixed with normal parameters
fn translate(Point { x, y }, dx, dy) =>
    Point { x: x + dx, y: y + dy }

// Nested
fn describe(Person { name, address: Address { city, country } }) =>
    "${name} lives in ${city}, ${country}"
```

And in for loops:

```
// Destructure each element as it is bound
for Point { x, y } in points {
    print("point at ${x}, ${y}")
}

for [a, b] in pairs {
    print("${a} and ${b}")
}
```

## Entry Point

Lastly, where does a `Fermion3` program begin? With the `main` function!

```
fn main(args) => {
    let name = "world"
    print("hello ${name}")
}
```

It must always take a parameter `args` which is the arguments to the program (if any).

