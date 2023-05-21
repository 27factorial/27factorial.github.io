# About this project

Swiftrs is a library that I have drafted up which will allow for (hopefully) seamless interaction
between compiled Rust libraries and Swift code. It makes use of a thin C FFI layer to pass data
between the languages without the need for serialization or certain runtime checks. While the design
is mostly done, I haven't yet finished figuring out hwo best to implement it, so there isn't any
code that you can use right now. If you are interested in how the library's API will work, read on!

# Overview

Swiftrs is an **experimental** library intended to help with using Rust code from Swift. It 
accomplishes this by using a combination of procedural macros, build scripts, and various Swift 
tools such as Sourcery and SwiftFormat.

In its most basic form, the library functions like so:
```rust
#[swiftrs::export]
pub struct Vector2 {
    x: f32,
    y: f32,
}

#[swiftrs::export]
impl Vector2 {
    pub fn new(x: f32, y: f32) -> Self {
        Self {
            x,
            y,
        }
    }
}
```

The `#[export]` macro is the core of the library, and has many different forms to meet different
needs. From the above code, the fully expanded form would look something like this in Rust:
```rust
#[repr(C)]
pub struct Vector2 {
    x: f32,
    y: f32,
}

impl Vector2 {
    pub fn new(x: f32, y: f32) -> Self {
        Self {
            x,
            y,
        }
    }
}

impl Vector2 {
    #[no_mangle]
    pub extern "C" fn __swiftrs_exported__Vector2_new(x: f32, y: f32) -> *mut Vector2 {
        ::std::boxed::Box::into_raw(::std::boxed::Box::new(Vector2::new(x, y)))
    }

    #[no_mangle]
    pub extern "C" fn __swiftrs_exported_drop__Vector2(ptr: *mut Vector2) {
        unsafe {
            _ = ::std::boxed::Box::from_raw(ptr);
        }
    }
}
```

Limited support is also available for generic data types and functions, which will be covered later.

In this document, "swiftrs" by itself refers to the Rust library. "Swiftrs Swift library" refers
to the library of the same name on the Swift side. 

# Exporting

As mentioned above, types, functions, and entire `impl` blocks can be exported. Exporting an item
exposes it through a C header file, which is then used to call Rust code from the generated Swift
implementation.

## Basic Data Types

Some basic data types are already exposed to the FFI layer without having to write any custom code.
These types include most of the primitive Rust types, a few common collection types, and some
commonly used standard library structs and enums. The full list of automatically available types is:

- `i{8, 16, 32, 64, 128, size}` => `Int{8, 16, 32, 64, 128}, Int`
- `u{8, 16, 32, 64, 128, size}` => `UInt{8, 16, 32, 64, 128}, UInt`
- `f{32, 64}` => `Float`, `Double`
- `char` => `RustChar`
- `bool` => `Bool`
- `()` => `() (or Void)`
- `[T; N]` => `RustArray<T>` (N is tracked at runtime in Swift)
- `*{const, mut} T` => `UnsafePointer<T>`, `UnsafeMutablePointer<T>`
- `&(mut) T` => `RustRef(Mut)<T>`
- `&(mut) [T]` => `RustSlice(Mut)<T>`
- `Option<T>` => `Optional<T>`
- `Result<T, E>` => `RustResult<T, E>`
- `Vec<T>` => `RustVec<T>`
- `String` => `RustString<T>`
- `AtomicU{8, 16, 32, 64, size}`
- `AtomicI{8, 16, 32, 64, size}`
- `AtomicBool`
- `AtomicPtr`
- `std::sync::Ordering`

where `T` is a valid type exported by swiftrs either automatically or through exported user-defined
types.

Note that the types shown above which do not have a stable layout are reimplemented in swiftrs to
have either a `#[repr(C)]` or a `#[repr(transparent)]` layout attribute attached to them. Types
which are reimplemented in swiftrs have `From`, `Into`, `TryFrom`, and `TryInto` implementations
for their corresponding standard library types if manual conversion between these types is needed.

Also, Since statically sized arrays do not have a Swift counterpart, some Swift code is
automatically generated for them when they are used, so that they may be safely converted to the
corresponding array type on the Rust side.

## The `#[export]` Attribute

In the simplest case, to export a type and expose it to Swift, you just have to add the
`#[export]` attribute to the type, function, or `impl` block you would like to expose.
However, most code isn't going to conform to the simplest case, so let's take a look at how else you
might use the attribute.

### `opaque`

This modifier can be added to any exported struct to make swiftrs generate an opaque type in the
place of the concrete type used in Rust. This can be useful when, for example, using types that are
not exposed to Swift, such as those in the standard library. This cannot be used in conjunction
with the `copy` modifier, as it only allows the type to be used through pointers on the Swift side.

Example:

```rust
#[swiftrs::export(opaque)]
pub struct SharedCounter {
    // Arc<T> is not an automatically exported type, thus the enclosing struct must
    // be made opaque to Swift.
    inner: Arc<AtomicU32>
}

#[swiftrs::export]
impl SharedCounter {
    pub fn new(init: u32) -> Self {
        Self {
            inner: Arc::new(AtomicU32::new(init)),
        }
    }
}
```

From this Rust code, the generated C header might look something like:

```c
// *includes omitted*

struct SharedCounter;
typedef struct SharedCounter SharedCounter;

SharedCounter *__swiftrs_exported__SharedCounter_new(uint32_t init)

void __swiftrs_exported_drop__SharedCounter(SharedCounter *ptr)
```

### `copy`

In Rust, types which implement the `Copy` trait can be trivially passed by value without moving
them. Instead, the Rust compiler copies the value and leaves the original intact. `Copy` types also
don't need to be dropped, since any type implementing `Drop` cannot implement `Copy`, so no drop
function is created in the generated C header file. This also means that the Swift side can use
the exported type by value without allocating or worrying about Rust's move semantics. This modifier
can be used on `impl` blocks, and cannot be used with the `opaque` modifier for reasons described in
[that section](#opaque). If the `impl` block that this modifier is used on does not correspond to a
type that implements `Copy`, a compiler error will be generated.

Example:

```rust
#[derive(Copy)]
#[swiftrs::export]
pub struct Pos {
    x: f32,
    y: f32,
    z: f32,
}

#[swiftrs::export(copy)]
impl Pos {
    pub fn new() -> Self {
        Self {
            x: 0.0,
            y: 0.0,
            z: 0.0,
        }
    }

    pub fn add(self, other: Pos) -> Pos {
        Pos {
            x: self.x + other.x,
            y: self.y + other.y,
            z: self.z + other.z,
        }
    }
}
```

From this Rust code, the generated C header might look something like:

```c
// *includes omitted*

typedef struct Pos {
    float x;
    float y;
    float z;
} Pos;

Pos __swiftrs_exported__Position_new(void);
Pos __swiftrs_exported__Position_add(Pos self, Pos other)
```

### ***TODO: Explain other \#\[export\] modifiers that may be created***

## Other Attributes

The `#[export]` macro does not cover every case, and some additional attributes are required to
change the behavior of the generated C header or Swift module.

### `#[init]`

By default, swiftrs assumes that the default initializer for your Rust type is named `new`, as this 
is the convention for Rust. However, if your type's initializer is something else, you can use the
`#[init]` attribute to instruct swiftrs to call the annotated function in the generated Swift 
initializer instead.

### ***TODO: Example***

### `#[indirect]`

In Swift, recursive enums are required to have the `indirect` keyword either before the 
specific recursive case, or before the `enum` keyword to apply an extra layer of indrection to all 
of the enum's variants. When using swiftrs, you can use the `#[indirect]` attribute in the same way.
Doing so will make the specific case or the entire enum `indirect`. For example:

```rust
#[swiftrs::export(copy)]
#[derive(Copy)]
pub enum Operator {
    Add,
    Sub,
    Mul,
    Div,
}

#[swiftrs::export]
pub enum Expr {
    Integer(i64),
    Float(f64),
    #[indirect]
    Arithmetic(Operator, Box<Expr>, Box<Expr>)
}
```

For this Rust code, the following Swift code will be generated:

```swift
public enum Operator {
    case add
    case sub
    case mul
    case div
}

public enum Expr {
    case integer(Int64)
    case float(Double)
    indirect case Arithmetic(Operator, Expr, Expr)
}
```

### `#[generic]`

In some cases, it can be very inconvenient to represent parts of your API using only concrete types.
Since the Rust compiler doesn't have any information about how the generated Swift code will be
used, monomorphization of generic functions used through FFI can not be done automatically. Instead,
user-defined types and functions must specify the types for which monomorphized versions will be 
generated. In Rust, this might look like the following:

```rust
// This is placeholder syntax. This may not be how the final version of this attribute looks.
#[swiftrs::export]
#[swiftrs::generic(T: [i32, u32, i64, u64])]
pub fn print_value<T: Display>(t: T) {
    println!("The value is {t}");
}
```

The above Rust code will generate a set of C function definitions in the header file like this:

```c
// *includes omitted*

void __swiftrs_exported_generic__print_value_i32(int32_t t);
void __swiftrs_exported_generic__print_value_u32(uint32_t t);
void __swiftrs_exported_generic__print_value_i64(int64_t t);
void __swiftrs_exported_generic__print_value_u64(uint64_t t);
```

If we were to use these functions directly from the Swift side, it would still be just as
inconvenient as manually specifying concrete types. To alleviate this, swiftrs automatically
generates a protocol which acts as a "dispatcher" to statically dispatch to the correct Rust 
function. For example:

```swift
// *This assumes C header functions are in-scope*

public func printValue<T: __swiftrs_dispatch__print_value_T>(t: T) {
    T.print_value(t)
}

/// Automatically generated by Swiftrs. Adopting this protocol with any types that do not already 
/// conform to it is unsupported, as its usage is a private implementation detail. Doing so may 
/// cause Undefined Behavior. 
public protocol __swiftrs_dispatch__print_value_T {
    func print_value(t: Self)
}

// Rust type: i32
extension Int32: __swiftrs_dispatch__print_value_T {
    func print_value(t: Int32) {
        __swiftrs_exported_generic__print_value_i32(t)
    }
}

// Rust type: u32
extension UInt32: __swiftrs_dispatch__print_value_T {
    func print_value(t: UInt32) {
        __swiftrs_exported_generic__print_value_u32(t)
    }
}

// etc, etc.
```

### ***TODO: What about multiple generic type parameters? Const generics? discuss enum dispatch?***

## Safety

Safety can be split into two different sections: type safety and memory safety.

Both the Rust and Swift side of the library are entirely type safe, assuming that special care is
taken on the Swift side to not allow invalid types to adopt certain protocols. Nothing different 
needs to be done on the Rust side to ensure type safety, as the compiler and library will 
automatically check types for you. The Swift side may become entirely type-safe without having to 
worry about invalid types adopting protocols if a system that allows "sealed" protocols is 
implemented.

As for memory safety, the Rust side should have the same memory safety guarantees as Rust does on
its own[^1], but users must be careful with their Swift code to ensure that they do not cause memory
safety problems. For example, it is possible in Swift to generate two mutable references to the same
object, which is Undefined Behavior in Rust. Some runtime checks can be done to prevent more trivial
cases of memory unsafety, but you still must ensure that you don't cause [Undefined Behavior][ubrs].

[^1]: This of course assumes that there are no bugs in the library.

[ubrs]: https://doc.rust-lang.org/reference/behavior-considered-undefined.html

## Type Representation and Swift's ARC

When a type is exported by using the `#[export]` attribute, swiftrs automatically adds the 
`#[repr(C)]` attribute unless you use the `opaque` modifier. This allows the value to be used in
Swift through the C FFI layer. When `opaque` is used, an opaque type is created in the C header file
instead.

### Product types

In Swift, product types can be represented using either structs or classes. On the other hand, Rust
only has one kind of product type: structs, so how do we know what Swift type to use for each Rust
type?

The answer lies in the `Drop` and `Copy` traits. Types which do not have any drop glue, and which
implement the `Copy` trait can be represented using a struct on the Swift side. However in Swift, 
only class types can have custom deinitializers. This means that any types that have drop glue (even
implicitly) must be represented with a class in Swift. If this were not the case, using these values
may cause memory leaks, as all non-`Copy` types are allocated on the heap using a `Box`.

### Enums

Rust enums which implement the `Copy` trait are mostly represented as-is in Swift. For non-`Copy` 
enums, more care is needed to track which variant is in use. Swiftrs does this by first generating
a Swift enum that describes which variant is used on the Rust side. After this variant-tracking
enum is created, the library then creates a class with both the current variant and a pointer to the
Rust data. This representation is required as Swift enums are "value types", which means that
these values are copied whenever they are assigned to a variable[^2]. Enums in Rust can also run
arbitrary code on drop, which means that a deinitializer is required on the Swift side, and only
classes can have deinitializers.

Using a class instead of Swift's enums can be much less convenient when it comes to destructuring
and `switch` expressions. Swiftrs attempts to alleviate this by doing two things:

1. Generating computed properties which correspond to each variant. For a Rust enum that contains a
variant with the signature `Integer(i64)`, a property named `integer` would be generated in Swift,
with the type `Int64?`. This allows you to use `if let` or optional chaining in Swift if you need
to destructure one variant.
2. To allow `switch`-like behavior, two `match` methods are generated for the class, taking closures
for each possible variant. One of the `match` methods takes only the closures for each variant,
while the other one also allows you to specify a default case. If the default case is not specified,
and an unexpected variant is encounter for whatever reason (as might be the case with 
non-exhaustive enums), `fatalError()` is called instead. To match the behavior of Rust's `match`
expressions as closely as possible, each closure is also able to return a value, but that value
must be the same for every closure passed in.

For example, consider the following Rust code:

```rust
#[swiftrs::export]
pub enum Data {
    Integer(i64),
    Float(f64),
    Boolean(bool),
}

impl Drop for Data {
    fn drop(&mut self) {
        match self {
            Data::Integer(_) => println!("Dropping Integer"),
            Data::Float(_) => println!("Dropping Float"),
            Data::Boolean(_) => println!("Dropping Boolean"),
        }
    }
}
```

This Rust code might generate something like the following Swift code:

```swift
public enum DataVariant {
    case integer
    case float
    case boolean
}

public class Data {
    let ptr: UnsafeMutableRawPointer;
    let variant: DataVariant;

    public var integer: Int64? {
        // --snip-- TODO: Fill this in once details are decided.
    }

    public var float: Double? {
        // --snip-- TODO: Fill this in once details are decided.
    }

    public var boolean: Bool? {
        // --snip-- TODO: Fill this in once details are decided.
    }

    public init(integer: Int64) {
        self.ptr = __swiftrs_exported_variant__Data_Integer(integer)
        self.variant = DataVariant.integer
    }

    public init(float: Double) {
        self.ptr = __swiftrs_exported_variant__Data_Float(float)
        self.variant = DataVariant.float
    }

    public init(boolean: Bool) {
        self.ptr = __swiftrs_exported_variant__Data_Integer(boolean)
        self.variant = DataVariant.boolean
    }

    deinit {
        __swiftrs_exported_drop__Data(self.ptr)
    }

    public func match<Ret>(
        integer fnInteger: (Int64) throws -> Ret,
        float fnFloat: (Double) throws -> Ret,
        boolean fnBoolean: (Bool) throws -> Ret
    ) rethrows -> Ret 
    {
        if let val = self.integer {
            return try fnInteger(val)
        } else if let val = self.float {
            return try fnFloat(val)
        } else if let val = self.boolean {
            return try fnBoolean(val)
        } else {
            fatalError("unexpected enum variant")
        }
    }

    public func match<Ret>(
        integer fnInteger: (Int64) throws -> Ret,
        float fnFloat: (Double) throws -> Ret,
        boolean fnBoolean: (Bool) throws -> Ret,
        otherwise: () throws -> Ret
    ) rethrows -> Ret 
    {
        if let val = self.integer {
            return try fnInteger(val)
        } else if let val = self.float {
            return try fnFloat(val)
        } else if let val = self.boolean {
            return try fnBoolean(val)
        } else {
            return try otherwise()
        }
    }


}
```

Note that when exporting an enum from Rust, no variant can have the name `Otherwise` (with any
permutation of capital and lowercase letters), as this variable name needs to be reserved for the
default case in the `match` method.

[^2]: There may be optimizations that remove unnecessary copies under the hood, but enums should
still be treated as if they are copied whenever they are assigned to a variable.

### Automatic Reference Counting


