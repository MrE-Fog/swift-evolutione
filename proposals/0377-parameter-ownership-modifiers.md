# `borrow` and `consume` parameter ownership modifiers

* Proposal: [SE-0377](0377-parameter-ownership-modifiers.md)
* Authors: [Michael Gottesman](https://github.com/gottesmm), [Joe Groff](https://github.com/jckarter)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Active Review (December 7 - December 14, 2022)**
* Implementation: available using the internal names `__shared` and `__owned`
* Review: ([first pitch](https://forums.swift.org/t/pitch-formally-defining-consuming-and-nonconsuming-argument-type-modifiers/54313)) ([second pitch](https://forums.swift.org/t/borrow-and-take-parameter-ownership-modifiers/59581)) ([first review](https://forums.swift.org/t/se-0377-borrow-and-take-parameter-ownership-modifiers/61020)) ([second review](https://forums.swift.org/t/combined-se-0366-third-review-and-se-0377-second-review-rename-take-taking-to-consume-consuming/61904))
* Previous Revisions: [1](https://github.com/apple/swift-evolution/blob/3f984e6183ce832307bb73ec72c842f6cb0aab86/proposals/0377-parameter-ownership-modifiers.md)

## Introduction

We propose new `borrow` and `consume` parameter modifiers to allow developers to
explicitly choose the ownership convention that a function uses to receive
immutable parameters. This allows for fine-tuning of performance by reducing
the number of ARC calls or copies needed to call a function, and provides a
necessary prerequisite feature for move-only types to specify whether a function
consumes a move-only value or not.

## Motivation

Swift uses automatic reference counting to manage the lifetimes of reference-
counted objects. There are two broad conventions that the compiler uses to
maintain memory safety when passing an object by value from a caller to a
callee in a function call:

* The callee can **borrow** the parameter. The caller
  guarantees that its argument object will stay alive for the duration of the
  call, and the callee does not need to release it (except to balance any
  additional retains it performs itself).
* The callee can **consume** the parameter. The callee
  becomes responsible for either releasing the parameter or passing ownership
  of it along somewhere else. If a caller doesn't want to give up its own
  ownership of its argument, it must retain the argument so that the callee
  can consume the extra reference count.

These two conventions generalize to value types, where a "retain"
becomes an independent copy of the value, and "release" the destruction and
deallocation of the copy. By default Swift chooses which convention to use 
based on some rules informed by the typical behavior of Swift code:
initializers and property setters are more likely to use their parameters to
construct or update another value, so it is likely more efficient for them to
*consume* their parameters and forward ownership to the new value they construct.
Other functions default to *borrowing* their parameters, since we have found
this to be more efficient in most situations.

These choices typically work well, but aren't always optimal.
Although the optimizer supports "function signature optimization" that can
change the convention used by a function when it sees an opportunity to reduce
overall ARC traffic, the circumstances in which we can automate this are
limited. The ownership convention becomes part of the ABI for public API, so
cannot be changed once established for ABI-stable libraries. The optimizer
also does not try to optimize polymorphic interfaces, such as non-final class
methods or protocol requirements. If a programmer wants behavior different
from the default in these circumstances, there is currently no way to do so.

Looking to the future, as part of our [ongoing project to add ownership to Swift](https://forums.swift.org/t/manifesto-ownership/5212), we will eventually have
move-only values and types. Since move-only types do not have the ability to
be copied, the distinction between the two conventions becomes an important
part of the API contract: functions that *borrow* move-only values make
temporary use of the value and leave it valid for further use, like reading
from a file handle, whereas functions that *consume* a move-only value consume
it and prevent its further use, like closing a file handle. Relying on
implicit selection of the parameter convention will not suffice for these
types.

## Proposed solution

We give developers direct control over the ownership convention of
parameters by introducing two new parameter modifiers `borrow` and `consume`.

## Detailed design

`borrow` and `consume` become contextual keywords inside parameter type
declarations.  They can appear in the same places as the `inout` modifier, and
are mutually exclusive with each other and with `inout`. In a `func`,
`subscript`, or `init` declaration, they appear as follows:

```swift
func foo(_: borrow Foo)
func foo(_: consume Foo)
func foo(_: inout Foo)
```

In a closure:

```swift
bar { (a: borrow Foo) in a.foo() }
bar { (a: consume Foo) in a.foo() }
bar { (a: inout Foo) in a.foo() }
```

In a function type:

```swift
let f: (borrow Foo) -> Void = { a in a.foo() }
let f: (consume Foo) -> Void = { a in a.foo() }
let f: (inout Foo) -> Void = { a in a.foo() }
```

Methods can use the `consuming` or `borrowing` modifier to indicate
respectively that they consume ownership of their `self` parameter or that they
borrow it. These modifiers are mutually exclusive with each other and with the
existing `mutating` modifier:

```swift
struct Foo {
  consuming func foo() // `consume` ownership of self
  borrowing func foo() // `borrow` self
  mutating func foo() // modify self with `inout` semantics
}
```

`consume` cannot be applied to parameters of nonescaping closure type, which by
their nature are always borrowed:

```swift
// ERROR: cannot `consume` a nonescaping closure
func foo(f: consume () -> ()) {
}
```

`consume` or `borrow` on a parameter do not affect the caller-side syntax for
passing an argument to the affected declaration, nor do `consuming` or
`borrowing` affect the application of `self` in a method call. For typical
Swift code, adding, removing, or changing these modifiers does not have any
source-breaking effects. (See "related directions" below for interactions with
other language features being considered currently or in the near future which
might interact with these modifiers in ways that cause them to break source.)

Protocol requirements can also use `consume` and `borrow`, and the modifiers will
affect the convention used by the generic interface to call the requirement.
The requirement may still be satisfied by an implementation that uses different
conventions for parameters of copyable types:

```swift
protocol P {
  func foo(x: consume Foo, y: borrow Foo)
}

// These are valid conformances:

struct A: P {
  func foo(x: Foo, y: Foo)
}

struct B: P {
  func foo(x: borrow Foo, y: consume Foo)
}

struct C: P {
  func foo(x: consume Foo, y: borrow Foo)
}
```

Function values can also be implicitly converted to function types that change
the convention of parameters of copyable types among unspecified, `borrow`,
or `consume`:

```swift
let f = { (a: Foo) in print(a) }

let g: (borrow Foo) -> Void = f
let h: (consume Foo) -> Void = f

let f2: (Foo) -> Void = h
```

Inside of a function or closure body, `consume` parameters may be mutated, as can
the `self` parameter of a `consuming func` method. These
mutations are performed on the value that the function itself took ownership of,
and will not be evident in any copies of the value that might still exist in
the caller. This makes it easy to take advantage of the uniqueness of values
after ownership transfer to do efficient local mutations of the value:

```swift
extension String {
  // Append `self` to another String, using in-place modification if
  // possible
  consuming func plus(_ other: String) -> String {
    // Modify our owned copy of `self` in-place, taking advantage of
    // uniqueness if possible
    self += other
    return self
  }
}

// This is amortized O(n) instead of O(n^2)!
let helloWorld = "hello ".plus("cruel ").plus("world")
```

## Source compatibility

Adding `consume` or `borrow` to a parameter in the language today does not
otherwise affect source compatibility. Callers can continue to call the
function as normal, and the function body can use the parameter as it already
does. A method with `consume` or `borrow` modifiers on its parameters can still
be used to satisfy a protocol requirement with different modifiers. The
compiler will introduce implicit copies as needed to maintain the expected
conventions. This allows for API authors to use `consume` and `borrow` annotations
to fine-tune the copying behavior of their implementations, without forcing
clients to be aware of ownership to use the annotated APIs. Source-only
packages can add, remove, or adjust these annotations on copyable types
over time without breaking their clients.

This will change if we introduce features that limit the compiler's ability
to implicitly copy values, such as move-only types, "no implicit copy" values
or scopes, and `consume` or `borrow` operators in expressions. Changing the
parameter convention changes where copies may be necessary to perform the call.
Passing an uncopyable value as an argument to a `consume` parameter ends its
lifetime, and that value cannot be used again after it's consumed.

## Effect on ABI stability

`consume` or `borrow` affects the ABI-level calling convention and cannot be
changed without breaking ABI-stable libraries (except on "trivial types"
for which copying is equivalent to `memcpy` and destroying is a no-op; however,
`consume` or `borrow` also has no practical effect on parameters of trivial type).

## Effect on API resilience

`consume` or `borrow` break ABI for ABI-stable libraries, but are intended to have
minimal impact on source-level API. When using copyable types, adding or
changing these annotations to an API should not affect its existing clients.

## Alternatives considered

### Leaving `consume` parameter bindings immutable inside the callee

We propose that `consume` parameters should be mutable inside of the callee,
because it is likely that the callee will want to perform mutations using
the value it has ownership of. There is a concern that some users may find this
behavior unintuitive, since those mutations would not be visible in copies
of the value in the caller. This was the motivation behind
[SE-0003](https://github.com/apple/swift-evolution/blob/main/proposals/0003-remove-var-parameters.md),
which explicitly removed the former ability to declare parameters as `var`
because of this potential for confusion. However, whereas `var` and `inout`
both suggest mutability, and `var` does not provide explicit directionality as
to where mutations become visible, `consume` on the other hand does not
suggest any kind of mutability to the caller, and it explicitly states the
directionality of ownership transfer. Furthermore, with move-only types, the
chance for confusion is moot, because the transfer of ownership means the
caller cannot even use the value after the callee takes ownership anyway.

Another argument for `consume` parameters to remain immutable is to serve the
proposal's stated goal of minimizing the source-breaking impact of
parameter ownership modifiers. When `consume` parameters are mutable,
changing a `consume` parameter to `borrow`, or removing the
`consume` annotation altogether, is potentially source-breaking. However,
any such breakage is purely localized to the callee; callers are still
unaffected (as long as copyable arguments are involved). If a developer wants
to change a `consume` parameter back into a `borrow`, they can still assign the
borrowed value to a local variable and use that local variable for local
mutation.

### Naming

We have considered several alternative naming schemes for these modifiers:

- The current implementation in the compiler uses `__shared` and `__owned`,
  and we could remove the underscores to make these simply `shared` and
  `owned`. These names refer to the way a borrowed parameter receives a
  "shared" borrow (as opposed to the "exclusive" borrow on an `inout`
  parameter), whereas a consumed parameter becomes "owned" by the callee.
  found that the "shared" versus "exclusive" language for discussing borrows,
  while technically correct, is unnecessarily confusing for explaining the
  model.
- A previous pitch used the names `nonconsuming` and `consuming`. The current
  implementation also uses `__consuming func` to notate a method that takes
  ownership of its `self` parameter. `consuming` parallels `mutating` as a
  method modifier for a method that consumes `self`, but we like the imperative
  form `consume` for parameter modifiers as a parallel with the `consume` operator
  in callers.
- The first reviewed revision used `take` instead of `consume`. Along with
  `borrow`, `take` arose during [the first review of
SE-0366](https://forums.swift.org/t/se-0366-move-function-use-after-move-diagnostic/59202).
  These names also work well as names for operators that explicitly
  transfer ownership of a variable or borrow it in place. However,
  reviewers observed that `take` is possibly confusing, since it conflicts with
  colloquial discussion of function calls "taking their arguments". `consume`
  reads about as well while being more specific.
- Reviewers offered `use`, `own`, or `sink` as alternatives to `consume`.

We think it is helpful to align the naming of these parameter modifiers with
the corresponding `consume` and `borrow` operators (discussed below under
Future Directions), since it helps reinforce the relationship between the
calling conventions and the expression operators: to explicitly transfer
ownership of an argument in a call site to a parameter in a function, use
`foo(consume x)` at the call site, and use `func foo(_: consume T)` in the
function declaration. Similarly, to explicitly pass an argument by borrow
without copying, use `foo(borrow x)` at the call site, and `func foo(_: borrow T)`
in the function declaration.

Some review discussion explored the possibility of
using different verb forms for the various roles; since we're already using
`consuming func` and `borrowing func` as the modifiers for the `self` parameter,
we could also conjugate the parameter modifiers so you write
`foo(x: borrowed T)` or `bar(x: consumed T)`, while still using `foo(borrow x)`
and `bar(consume x)` as the call-site operators.

### Effect on call sites and uses of the parameter

This proposal designs the `consume` and `borrow` modifiers to have minimal source
impact when applied to parameters, on the expectation that, in typical Swift
code that isn't using move-only types or other copy-controlling features,
adjusting the convention is a useful optimization on its own without otherwise
changing the programming model, letting the optimizer automatically minimize
copies once the convention is manually optimized.

It could alternatively be argued that explicitly stating the convention for
a value argument indicates that the developer is interested in guaranteeing
that the optimization occurs, and having the annotation imply changed behavior
at call sites or inside the function definition, such as disabling implicit
copies of the parameter inside the function, or implicitly taking an argument
to a `consume` parameter and ending its lifetime inside the caller after the
call site. We believe that it is better to keep the behavior of the call in
expressions independent of the declaration (to the degree possible with
implicitly copyable values), and that explicit operators on the call site
can be used in the important, but relatively rare, cases where the default
optimizer behavior is insufficient to get optimal code.

## Related directions

### Caller-side controls on implicit copying

There are a number of caller-side operators we are considering to allow for
performance-sensitive code to make assertions about call behavior. These
are closely related to the `consume` and `borrow` parameter modifiers and so
share their names. See also the
[Selective control of implicit copying behavior](https://forums.swift.org/t/selective-control-of-implicit-copying-behavior-take-borrow-and-copy-operators-noimplicitcopy/60168)
thread on the Swift forums for deeper discussion of this suite of features

#### `consume` operator

Currently under review as
[SE-0366](https://github.com/apple/swift-evolution/blob/main/proposals/0366-move-function.md),
it is useful to have an operator that explicitly ends the lifetime of a
variable before the end of its scope. This allows the compiler to reliably
destroy the value of the variable, or transfer ownership, at the point of its
last use, without depending on optimization and vague ARC optimizer rules.
When the lifetime of the variable ends in an argument to a `consume` parameter,
then we can transfer ownership to the callee without any copies:

```swift
func consume(x: consume Foo)

func produce() {
  let x = Foo()
  consume(x: consume x)
  doOtherStuffNotInvolvingX()
}
```

#### `borrow` operator

Relatedly, there are circumstances where the compiler defaults to copying
when it is theoretically possible to borrow, particularly when working with
shared mutable state such as global or static variables, escaped closure
captures, and class stored properties. The compiler does
this to avoid running afoul of the law of exclusivity with mutations. In
the example below, if `callUseFoo()` passed `global` to `useFoo` by borrow
instead of passing a copy, then the mutation of `global` inside of `useFoo`
would trigger a dynamic exclusivity failure (or UB if exclusivity checks
are disabled):

```swift
var global = Foo()

func useFoo(x: borrow Foo) {
  // We need exclusive access to `global` here
  global = Foo()
}

func callUseFoo() {
  // callUseFoo doesn't know whether `useFoo` accesses global,
  // so we want to avoid imposing shared access to it for longer
  // than necessary, and we'll pass a copy of the value. This:
  useFoo(x: global)

  // will compile more like:

  /*
  let globalCopy = copy(global)
  useFoo(x: globalCopy)
  destroy(globalCopy)
   */
}
```

It is difficult for the compiler to conclusively prove that there aren't
potential interfering writes to shared mutable state, so although it may
in theory eliminate the defensive copy if it proves that `useFoo`, it is
unlikely to do so in practice. The developer may know that the program will
not attempt to modify the same object or global variable during a call,
and want to suppress this copy. An explicit `borrow` operator could allow for
this:

```swift
var global = Foo()

func useFooWithoutTouchingGlobal(x: borrow Foo) {
  /* global not used here */
}

func callUseFoo() {
  // The programmer knows that `useFooWithoutTouchingGlobal` won't
  // touch `global`, so we'd like to pass it without copying
  useFooWithoutTouchingGlobal(x: borrow global)
}
```

If `useFooWithoutTouchingGlobal` did in fact attempt to mutate `global`
while the caller is borrowing it, an exclusivity failure would be raised.

#### Move-only types, uncopyable values, and related features

The `consume` versus `borrow` distinction becomes much more important and
prominent for values that cannot be implicitly copied. We have plans to
introduce move-only types, whose values are never copyable, as well as
attributes that suppress the compiler's implicit copying behavior selectively
for particular variables or scopes. Operations that borrow
a value allow the same value to continue being used, whereas operations that
consume a value destroy it and prevent its continued use. This makes the
convention used for move-only parameters a much more important part of their
API contract, since it directly affects whether the value is still available
after the operation:

```swift
moveonly struct FileHandle { ... }

// Operations that open a file handle return new FileHandle values
func open(path: FilePath) throws -> FileHandle

// Operations that operate on an open file handle and leave it open
// borrow the FileHandle
func read(from: borrow FileHandle) throws -> Data

// Operations that close the file handle and make it unusable consume
// the FileHandle
func close(file: consume FileHandle)

func hackPasswords() throws -> HackedPasswords {
  let fd = try open(path: "/etc/passwd")
  // `read` borrows fd, so we can continue using it after
  let contents = try read(from: fd)
  // `close` consumes fd, so we can't use it again
  close(fd)

  let moreContents = try read(from: fd) // compiler error: use after consume

  return hackPasswordData(contents)
}
```

When protocol requirements have parameters of move-only type, the ownership
convention of the corresponding parameters in an implementing method will need to
match exactly, since they cannot be implicitly copied in a thunk:

```swift
protocol WritableToFileHandle {
  func write(to fh: borrow FileHandle)
}

extension String: WritableToFileHandle {
  // error: does not satisfy protocol requirement, ownership modifier for
  // parameter of move-only type `FileHandle` does not match
  /*
  func write(to fh: consume FileHandle) {
    ...
  }
   */

  // This is OK:
  func write(to fh: borrow FileHandle) {
    ...
  }
}
```

All generic and existential types in the language today are assumed to be
copyable, but copyability will need to become an optional constraint in order
to allow move-only types to conform to protocols and be used in generic
functions. When that happens, the need for strict matching of ownership
modifiers in protocol requirements would extend also to any generic parameter
types, associated types, and existential types that are not required to be
copyable, as well as the `Self` type in a protocol that does not require
conforming types to be copyable.

### `set`/`out` parameter convention

By making the `borrow` and `consume` conventions explicit, we mostly round out
the set of possibilities for how to handle a parameter. `inout` parameters get
**exclusive access** to their argument, allowing them to mutate or replace the
current value without concern for other code. By contrast, `borrow` parameters
get **shared access** to their argument, allowing multiple pieces of code to
share the same value without copying, so long as none of them mutate the
shared value. A `consume` parameter consumes a value, leaving nothing behind, but
there still isn't a parameter analog to the opposite convention, which would
be to take an uninitialized argument and populate it with a new value. Many
languages, including C# and Objective-C when used with the "Distributed
Objects" feature, have `out` parameter conventions for this, and the Val
programming language calls this `set`.

In Swift up to this point, return values have been the preferred mechanism for
functions to pass values back to their callers. This proposal does not propose
to add some kind of `out` parameter, but a future proposal could.

## Acknowledgments

Thanks to Robert Widmann for the original underscored implementation of
`__owned` and `__shared`: [https://forums.swift.org/t/ownership-annotations/11276](https://forums.swift.org/t/ownership-annotations/11276).

## Revision history

The [first reviewed revision](https://github.com/apple/swift-evolution/blob/3f984e6183ce832307bb73ec72c842f6cb0aab86/proposals/0377-parameter-ownership-modifiers.md)
of this proposal used `take` and `taking` instead of `consume` and `consuming`
as the name of the callee-destroy convention.
