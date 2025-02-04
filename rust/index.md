---
layout: page
title: A Swift guide to Rust
---

I needed to onboard to Rust recently for a project.  Here is the guide that I wanted.

# Why Rust

Mostly because I needed a high-performance language that was cross-platform.  By high-performance really I mean "not GCed".  By cross-platform I mean it works on both iOS and 64-bit Linux.

Within those constraints you basically have C11, C++11, and Rust.  (Swift is not cross-platform.  ObjC is sort of, but it's not well-supported.)  I hate C++ on principle, so that was out.  I know C11 quite well, but my problem was such that a more "modern" language would be a help.  So I picked Rust.

## How is it?

It's okay.  Really what most people want to know is "is it really safer?" and the answer is "yes, but the cost is high."  Using Rust has caught, perhaps, 10s of bugs that even very well-unit-tested C11 (or Swift) would not have caught.  The trouble is that I spend roughly 25% of my development time wrestling with the compiler to achieve that safety.  Is it worth it?  Maybe, for some programs.

Some people tell me that "you get better with time" and such, and there is some truth to that, but I'm now 3 months in and the 25% number is still about right.  (It used to be, like 75%, so it did get better with time).  From the perspective of a daily Swift user, Rust is just fundamentally not as productive as Swift, although it does produce faster-running and safer programs.  More about this in the Design Philosophy section.

To me, Rust is sort of a coincidence.  What I really want is a C++ that doesn't suck.  But you can produce a better C++ while drunk.  Rust has, IMHO, produced it.  But it is not better because it is safer, it is better because it's not mind-numbingly stupid.  Insofar as a lot of attention in the language design is focused on safety at the expense of everything else, I find it annoying.  But not as annoying as C++.

# Methodology

I'm an advanced Swift developer but I'm still learning Rust.  So some things might be wrong.  Send me a [PR on GitHub](https://github.com/drewcrawford/drewcrawford.github.io/pulls).

Generally I believe that things can be taught best by people who still remember what it's like not to know them.  A lot of the Rust tutorials out there are written by people who have used the language for years.  So they gloss over some details that I think are important.

Other tutorials cover things I already know, because I know Swift (another high-performance systems language) relatively well.  And also C.  So I don't need yet another pointer tutorial, or 'what is a string'.  I need to know the new stuff.

If you are trying to go from Rust to Swift, this guide might also be useful, since it explains one in terms of the other, but I make no guarantees.

I'm not going to cover basic things like `func` => `fn`, `!` => `unwrap()` and so on.  For simple things like that, try

* [the book](http://doc.rust-lang.org/book/)
* [rust by example](http://rustbyexample.com/index.html)
* The [standard library reference](http://doc.rust-lang.org/std/)
* [An alternate introduction to Rust](http://words.steveklabnik.com/a-new-introduction-to-rust), for a basic lifetime guide

Instead I'm going to cover the "hard stuff".  The stuff that I spent hours banging my head against the wall about, and I could not find any good resources for.

* How to fight the borrow checker and win
* Closures and the memory model, in depth
* OO programming without objects
* Understanding the "selfish" functions (`self`, `&mut self`, `&self`), and their semantics, in depth
* Multithreading
* Grab bag of "medium-sized" topics that may be of interest to Swift programmers: generics, semicolons, visibility, static dispatch

Let's start with the type system.

# Value Types

In Swift, we have value types (Structs/Enums) and reference types (Classes).  They are "on the same level" in the sense that, they are both fundamental types.

In Rust, value types are fundamental, and reference types are secondary.  They are not on the same level.  There is not even any "class" in Rust.  If you want a reference type, you must derive it from some underlying value type.  We'll see how to do this once we cover the value types.

## Enums

Enums are practically the same in both languages.  One difference is that Rust uses `match` where Swift uses `select`.  And the syntax is a little different, and Rust can do a little more.  But that's nothing to write home about.

## Structs

Rust's structs are also very similar to Swift structs.  The differences are, as best as I can tell:

1.  The `struct` itself contains only the *fields*.  The functions are contained in a separate block called `impl StructName`.  This is a lot like ObjC's `@interface` and `@implementation` separation, but unlike ObjC they are generally contained in the same file.
2.  Right before the struct declaration you can include a set of lines like `#[derive(Clone)]`.  The presence of this line magically writes an implementation for the `Clone` trait.  More info on this later.

# Reference Types

In Swift a reference type is declared with `class`.  Rust does not really have a notion of this.  Instead, there are three "reference wrappers" that turn a value type into a reference type.  These "wrappers" are part of the standard library, not part of the language itself.

Note that in all cases, you get reference semantics *by wrapper*, so this is something that the *user* of the type does, not specified by the *type itself*.  Whereas in Swift `class` causes the *type itself* to be a reference.  In Rust, over here we can use something as a reference type with a wrapper, and somewhere else we can use it as a value type without a wrapper.  Value types are fundamental; reference types are glued on the top.

You can, technically, declare a *type alias* that works out to e.g. `Box<T>` but this is not idiomatic.  People expect the caller to decide how to use the type, and for it not to be a property of the type itself.

## Box<T>

`Box<T>` is the simplest reference wrapper.  It literally just contains a pointer to the thing.  So if I say `Box::new(5);` I have a pointer to 5.

So what are these other reference wrappers?  What is there more to do?  Well, it has to do with Rust's idea of pointers.

Rust gets serious about pointer safety, much more serious than even Swift is.  For example, Rust does not want you to have 2 mutable references to the same memory, because what if somehow two threads write data at the same time.  

A `Box` only has one *owner*.  A Rust *owner* is like a Swift strong pointer.  In Swift you can do

```swift
let o1 = NSObject()
let o2 = o1 //second strong pointer to o1
println("\(o1)")
println("\(o2)")
```

However the Rust equivalent is not allowed:

```rust
fn main() {
    let a = Box::new([5]);
    let b = a;
    println!("{:?}",a);
}
```

```
test.rs:4:21: 4:22 error: use of moved value: `a`
test.rs:4     println!("{:?}",a);
                              ^
```

The Rust semantics in this code are sort of novel.  The line

```rust
    let b = a;
```

causes a *move*.  Semantically, the strong pointer of our `Box` *moves* from `a` to `b`.  `a` is no longer valid after the assignment.  `b` becomes valid.  You can in fact assign it as many times as you like, but only the last one is a valid strong pointer.  In Rust we say the Box *has one owner*.

# Rc

To get closer to Swift semantics, we can use `Rc`.  `Rc` is like `Box` but it can have more than one owner.  Here is an updated sample:

```rust
use std::rc::Rc;
fn main() {
    let a = Rc::new([5]);
    let b = a.clone();
    println!("{:?}",a);
}
```

This compiles.  Important changes include the use of `clone()`.  `clone()` returns a 'new' strong pointer.  If you don't use `clone()` you will get the same error as before.  Rust does not allow implicit multiple strong pointers like Swift does, you have to opt into it.

There is another difference here, that has to do with memory management.  In Swift's case, whether reference counting is used is an implementation detail.  That is, the compiler inserts the retain/release calls.  If the number of strong pointers is small, it may omit reference counting calls, or include them, as it decides.  In Rust, `retain` is explicit:  you call `clone` on an `Rc`.  So you control when this happens, not the compiler.  (`release` is what happens when your variable goes out of scope, so that is implicit, like Swift.)

Finally, `Rc` (and `Arc`) give you access to an `&T`, whereas `Box` can also give you access to an `&mut T`.  This has something to do with Rust's memory safety; there can only be one mutable pointer.  If you need more than one mutable pointer in an `Rc`, you can try `Rc<Cell<T>>`.

# Arc

Rust has a final reference wrapper type, called `Arc`.  It is better to assume that this is nothing like the Swift technology ARC.

It is basically the same as `Rc`, except `clone` is implemented with an atomic increment instruction (and thus is threadsafe).    Swift's retain/release system on the other hand is always threadsafe, and you cannot get a non-atomic version.

You may wonder why Rust has all these types for what is essentially the same thing in Swift, and what it comes down to is performance and control.  Swift takes the view that whether or not something has multiple strong pointers is something that the compiler should figure out during an optimization pass.  And so Swift does not make you worry about such things, although there are cases where it is overly cautious and you spend a little time waiting for an atomic instruction that is not strictly necessary.

Rust takes the view that this is the programmer's responsibility.  And so you are actually opting into multiple strong pointers, and atomics.  That can produce faster code in some cases, although it is a lot more work.

Finally, `Arc` (and `Rc`) give you access to an `&T`, whereas `Box` can also give you access to an `&mut T`.  This has something to do with Rust's memory safety; there can only be one mutable pointer.  If you need more than one mutable pointer in an `Arc`, you can try `Arc<Mutex<T>>`.

## A word on thread safety

"Forgetting to use atomics" may scare some Swift programmers.  The reality however is that in Rust you cannot "just" forget an atomic and so get into a thread bug situation.  Instead, your program won't compile.  So it is not really that scary.

## A word on performance of reference and value types

It's important to distinguish between reference and value types *semantically* vs reference and value types *performance-wise*.

In C for example, when we say "pass-by-value" what we really mean is "pass-with-copy".  Whereas pass-by-reference is "pass-without-copy".  And so people pass references around in C, not necessarily because they have to for some semantic reason, but because it is fast.

However, this view does not hold true for Swift and Rust.  "pass-by-value" can be zero-copy (if it's immutable), or it can be copy-on-write (if it's mutable), or something like that.  There is not always a copy.  Pass-by-reference is not necessarily faster than pass-by-value.  Use the semantics that make sense for your program, and do not work on the presumption that one is faster than the other.

# Traits

Traits in Rust serve the purpose of several Swift features: extensions, protocols, and inheritance.

## Traits as protocol:

```rust
pub trait MyProtocol : SuperTrait {
    fn my_func(&mut self);
}
```

You see here that we have something like a protocol here.  Structs that *conform* to our *protocol* (*implement* our *trait*) must have this function.

Also like protocols, traits can have a supertrait.  This is not very commonly done in Swift, but it is indeed possible, check out `NSObjectProtocol`.  

## Traits as extensions:

When you actually go to implement a trait, you do it in a special block:

```rust
impl MyProtocol for Struct {
    fn my_func(&mut self) {

    }
}
```

This separation between the main body code and the trait code is similar to `extension` in Swift.  It can be in different files, etc.

Also like `extension`, you can provide a trait impelementation for a system class.

```rust
impl<T> MyProtocol for Box<T> {
    fn my_func(&mut self) {

    }
}
```

Unlike `extension` however (and possibly a better idea), you have to opt-in to a trait at the place that it is used.  For example

```rust
fn main() {
    use MyProtocol; //opt-in to MyProtocol on all objects that have an impelementation
    Box::new(5).my_func();
}
```

This avoids some of the "monkeypatching scariness" of traditional class extensions, since you have to opt into them with a fully-qualified name at the place where they are used.

If you don't do this you will get an error, and there will be a note suggesting one or more `use` lines that you need to add to make it work.

## Traits as inheritance

Finally, traits can provide a default implementation of the function.

```rust
pub trait MyProtocol : SuperTrait {
    fn my_func(&mut self) {
        println!("Hello from trait!")
    }
}
```

(Contrawise to a Swift protocol, which cannot provide a function body.)

A struct by default will inherit this implementation, but it may instead override it.

```rust
impl<T> MyProtocol for Box<T> {
    fn my_func(&mut self) {
        println!("Hello from Box<T>");
    }
}
```

(Contrawise to a Swift class extension, which cannot be overriden.)

## Traits as *not* multi-inheritance

Sadly, there is no clear path to multi-inheritance with traits.  If we have

```rust
trait Circle: Shape {
    
}
```

Then we are saying that all `Circle`s are `Shape`s.  We are *not* saying however that all `Circle`s are `Shape`s *automatically*.  Instead we are explaining to the compiler that for every `struct` which implements `Circle` we will also implement `Shape`.

Generally for this problem, one of several approaches are taken:

1.  Preprocessor macros to supply an implementation of `Shape`
2.  Use [associated items](http://stackoverflow.com/a/29292642/116834)
3.  Use structs instead.  `struct Circle` with a pointer to a `Shape` struct.
4.  Comment on [this issue](https://github.com/rust-lang/rfcs/issues/1024)

Overall traits are an interesting idea, and unify a lot of concepts that in Swift would be separate into a single tool.

# A word on design philosophy

The fact that `Circle`  isn't automatically `Shape` illustrates an important, but subtle difference in design philosophy.  In Rust, you tell the compiler what you are going to do and then you do it.  So in essence, the compiler is told twice: once in a high-level shorthand way, that acts as a "promise". And another time in a finely detailed, more roundabout way, that acts as a "fulfillment" of the earlier promise.  The idea is that the compiler checks your work, but doesn't lift a finger to help you.

Meanwhile in Swift you tell the compiler something once and then it does it.  As a result Swift can be more concise, at the cost of losing some control about exactly how the promise will be fulfilled.

In many ways it's like manual vs auto transmission in a car.  In an automatic transmission car you just tell the car what gear to use, but in a manual transmission you're involved in the mechanics of switching the gears.  At many times in Rust you are quite involved in the mechanics of various language features.  If you thought Swift was writing to-the-metal, I've got news for you, son.  Rust is to-the-particle.

Examples of this philosophical debate on both large and small scales include:

* Expecting callers to understand, research, and select some bolt-on wrapper to achieve reference semantics rather than a built-in `class` keyword
* Move semantics and opt-in `clone` (strong pointers), instead of Swift's implicit strong pointers
* Manual inheritance (as `Circle/Shape`) vs more traditional automatic inheritance
* Verbosity of Rust's `impl` (as we will see in the next section)
* Swift's automatic pointer classes vs Rust's more manual lifetimes (as we cover later on in this guide)
* Structs with fields have a different syntax than structs without fields, and the caller is expected to know which kind of struct it's using
* Swift has an emphasis on "automatic" properties (e.g. `private (set) var whatever` emits a getter and setter that meet certain constraints) while Rust never writes functions for you

I'm tempted to say Rust is 'lower-level' than Swift, but that conjures up the chasm between say Python and C, and this is nowhere near that drastic.  It's fair to say however that the Swift developer will have the distinct sensation that they are repeating themselves a few extra times.  They're not wrong; that's what Rust asks of you.

Another way to say this is that if there are two ways of doing something, Swift will try to pick a sensible default, while Rust will do neither and write you a long error message explaining how you can do it yourself.  In Rust's defense, I've found that doing things myself can produce much faster code, because I'm optimizing for some particular case instead of using a language-wide default implementation.  On the other hand, Swift's defaults tend to be sensible and they produce the program a lot faster.  But that is the tradeoff.

# `impl` syntax

One puzzling bit to new programmers is the verbosity of `impl`:

```rust
impl<'a, K, V> for Struct<'a, K, V> { }
```

`K` and `V` are generic parameters, so that is not so bad.  `'a` is a *lifetime*, which is a Rust-specific concept we will talk about shortly, but syntactically it is similar to a generic type.

The puzzling thing about this is that the list `<'a, K, V>` appears twice instead of once.   The reason is that `impl` can do more than just implement a struct.  It can implement a trait for a struct:

```rust
impl<'a, K, V, A, B> MyTrait<A,B> for Struct<'a, K, V> { }
```

...for only certain kinds of parameters:

```rust
impl<'a, K, V, A, B> MyTrait<A,B> for Struct<'a, K, V>  where A: Foo, B: Bar + 'a, K: Baz<A>, V: Qux<quux=Corge> { ... }
```

The meaning of this is that the trait is implemented when these constraints are satisfied, and is not implemented in other cases.  Note that it is currently difficult to provide many implementations for the same trait because it is hard to ensure that the conditions for which they are implemented do not overlap.  This problem is called the [negative trait bounds problem](https://github.com/rust-lang/rfcs/pull/586).

This last case deserves some explaining.  Note that we can say either `impl<K: Foo>` or `impl<K> ... where K: Foo`.  I think the `where` syntax is newer and designed to be more extensible.

When we say `K: Baz<A>` we mean `K` implements a trait, where the generic parameter of that trait is A.  

When we say `V: Quz<quux=Corge>` we mean `Q` implements a trait, where the trait's *associated item `quux`* takes the value `Corge`.  [Associated items](https://github.com/rust-lang/rfcs/blob/master/text/0195-associated-items.md) are basically identical to Swift's [associated types](http://www.russbishop.net/swift-associated-types) so I will not say any more about them, other than to point out the syntactical differences.

# Lifetimes

You should read [An alternate introduction to Rust](http://words.steveklabnik.com/a-new-introduction-to-rust) to get the "scopes" understanding of lifetimes, but I'm going to give a complementary understanding that explains some of the practical implications.

In Swift, we say

```swift
struct MyStruct {
    foo: NSObject()
}
```

In this case `MyStruct` has a strong pointer to `foo`.  Pointers are something of an implementation detail in Swift, but since `NSObject` comes from ObjC we know that `NSObject` is heap-allocated and therefore `foo` is a pointer, not embedded in the struct itself.  

(Objects that are defined in Swift, on the other hand, are not defined to be heap-allocated nor stack-allocated, so in that case whether or not a pointer is used is an implementation detail.)

For a literal Rust translation there is no ambiguity on this question; a pointer is not used.

```rust
struct MyStruct {
    foo: Foo,  
};
```

here `foo` is an actual field in our struct, that is, the entire `foo` is placed in the struct's memory.  The implication is, in this Rust code the size of `MyStruct` grows with the size of `Foo`, whereas in the Swift code, the pointer to `NSObject` is always constant size because pointers are the same size everywhere on your computer.  

As a corollary, the Rust compiler must be able to figure out the size of `Foo` to declare `MyStruct`.  If `Foo` is a trait (which could be implemented by structs of various sizes) then we cannot work out the memory layout of `MyStruct` and thus it cannot be defined.

A more idiomatic translation is to tell Rust to use a pointer to `Foo`, which in Rust is an explicit thing:

```rust
struct MyStruct {
    foo: &Foo,  
};
```

This behaves as you would expect, and as a corollary `Foo` can now be a trait (although there are some interesting implications, see the section on Dispatch below).

One puzzle though is what it means to have

```rust
let m : MyStruct = ...
(*m.foo).doFoo();
```

From a C perspective there are *in theory* 3 possibilities for what this can mean:

1.  if `m.foo` is unitialized then the behavior is undefined.  However both Swift and Rust require elements of structs (classes) to be initialized in all cases (well, unless you have "initialized" it to `mem::unitialized`)
2.  `m.foo` points to a valid value, whether on the heap or the stack
3.  `m.foo` points to a formerly valid value, but it has gotten blown away.  Like the artist formerly known as Prince.

The languages handle the third case very differently, which I will now explain in some detail.

## Swift ARC and pointer memory classes

In Swift, pointers have *memory classes* that explain how we handle this case.

By default pointers are `strong`.  This means that as long as the pointer exists, the value it points to will also.

It is important to understand how this is implemented.  When we say:

```swift
{
    let j = NSObject()
    //stuff
}
```

it is equivalent to

```swift
{
    let j = NSObject().retain()
    //stuff
    j.release()
}
```

When I say "is equivalent to", it is important to understand that the compiler has broad latitude in how to optimize these calls.  It may omit them if it can prove it is safe.  But you shouldn't notice.

`retain` and `release` here increment and decrement the *reference count* of our object.  It is incremented when a strong pointer is created and it is decremented when the strong pointer goes away.  When all strong pointers go away, the object is deallocated.

Swift also has a `weak` pointer class with semantics that A) we do not increment the *reference count* when a weak pointer is created and B) if the reference count becomes zero, the pointer takes on the value `nil` (`None`).  As a consequence, the `weak` class can only be used for optional types.

Finally, Swift has an `unowned` pointer class with semantics that A) we do not increment the *reference count* when an unowned pointer is created and B) if the reference count becomes zero and you try to access it, your program will deterministically crash.

In summary, there are two very important bits to understand here:

1.  That the `retain` and `release` calls are inserted by the compiler, that is, ARC is a compile-time technology
2.  That the actual reference counting behavior is a runtime behavior
3.  Although there is some runtime stuff going on, it is *deterministic*.  We can be certain that an object will be deallocated when all strong pointers blow away (with very minor transpositions for compiler optimizations) (not batched up later like garbage collection)

## Rust and lifetimes

Rust lifetimes exist to solve the same problem that Swift pointer memory classes do: ensure that you do not have pointers to objects that have been blown away.  In practice, however, the implementation is different.

In Swift, this is enforced through a mix of compiler-time and run-time behavior.  Function calls are inserted automatically at compile-time; they are executed at runtime.

In Rust on the other hand, you simply state what should happen, and the compiler checks to see if you did it.  For example:

```rust
struct MyStruct<'a> {
    foo: &'a Foo
}
```

Says `MyStruct.foo` will always be valid, that is, the `foo` will not vanish out from under us.  *But it does not explain how this is enforced.*  That is because you must enforce it yourself.  The compiler will check your work, but it will not lift a finger to help you.

To enforce it, you generally have to demonstrate to the compiler that the constraint is met.  For example in this code:


```rust
let f = Foo;
let j = MyStruct{ foo: &f};
```

We can see right away that f will not become a dangling pointer.  However if we did something like this:

```rust
let mut j: MyStruct;
{
    let f = Foo;
    j = MyStruct{ foo: &f}
} // f goes poof
j.foo();
```

Then the compiler will say "now wait a minute, this constraint isn't satisfied."

It is important to emphasize one other fact.  All pointers have lifetimes in Rust, whether there is a little `'a` there or not.  The difference is, some lifetimes are figured out automatically (called *elided*) and other times you explain what they are.  Generally then, battling the borrow checker is an exercise in overriding elided lifetimes with explicit ones.

There are cases when the compiler is dumb and it does not know the constraint is met even though it is.  Generally in those cases, the secret is to explain to the compiler about lifetimes, so that it understands what we are doing.  If we have `box: Box<Foo>` and `MyStruct` then the compiler doesn't know if `MyStruct.foo = box;` is legal, because what if the box blows away before the `MyStruct`.  But if we have `box: 'd Box<Foo>` and `&'d MyStruct` then we know the assignment is okay, because we have asserted the box cannot blow away before the `MyStruct`.

One last point.  lifetimes "flow" from a struct to its members:

```rust
struct MyStruct<'a> {
    foo: &'a Foo,
}
```

They also "flow" from a function to its return values:

```rust
fn foo(&'a Foo) -> &'a Foo;
```

Here we mean that the return value will not outlast the parameter.  This might be useful for returning a "view" of the input.  For example, we have a string, and we return a substring, where the substring is something like `&string[2]` in C, that is, a pointer to some element.

Unlike C, it is safe, because back at the callsite somebody checks to see if the return value lasts longer than the parameters.  

# Closures

*Update: one of the Rust maintainers wrote a good guide to closures "under the hood" [here](http://huonw.github.io/blog/2015/05/finding-closure-in-rust/)*

The Swift and Rust conception of a closure is similar, but subtly different.

Let's take a Swift example. Somewhat surprisingly, this is a legal Swift program.

```swift
var write : (() -> ())? = nil
var read: (()->())? = nil

func test() {
    var str = "Test"
    write = {
        str = "Whatever"
        return
    }
    read = {
        println("\(str)")
        return
    }
}
test()
read!() //"Test"
write!() 
read!() //"Whatever"
```

Author's note: this is both bad Swift and bad Rust, but it uses a surprising number of language features, so we'll roll with it.

It is sort of odd to see that our `read` and `write` closures can still access `str` even though that variable is out of scope.  If this was C, our `str` would no longer be on the stack frame--it would be gone somewhere.  In Swift however, `str` continues to exist as long as some closure still exists that needs it.  It is not tied to some particular stack frame.

Now that example in Rust:

```rust
fn main() {

    let mut write  = None;
    let mut read = None;

    let test = ||{
        let mut str = "Test";
        write = Some(||{
            str = "Whatever";
            return
        });
        read = Some(||{
            println!("{:?}",str);
            return
        });
    };
    test();
    read.unwrap()();
    write.unwrap()();
    read.unwrap()();
}
```

Notable changes include:

1.  We don't specify a type for `read` and `write`, the compiler figures it out
2.  test is now a closure instead of a function.  In Swift, functions are simply named closures.  In Rust, they are different.  Only closures can capture the environment.
3.  Rust wants a little more hand-holding around Optionals.  In Swift you can assign to an optional directly, in Rust, you must assign it to `Some(the thing)`.

Unfortunately, the Rust compiler takes a dim view of this code, producing a full 85 lines of errors:

```
test.rs:17:5: 17:9 error: cannot borrow immutable local variable `test` as mutable
test.rs:17     test();
               ^~~~
test.rs:18:5: 18:9 error: cannot move out of `read` because it is borrowed
test.rs:18     read.unwrap()();
               ^~~~
test.rs:6:16: 16:6 note: borrow of `read` occurs here
test.rs:6     let test = ||{
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(||{
test.rs:9             str = "Whatever";
test.rs:10             return
test.rs:11         });
           ...
test.rs:19:5: 19:10 error: cannot move out of `write` because it is borrowed
test.rs:19     write.unwrap()();
               ^~~~~
test.rs:6:16: 16:6 note: borrow of `write` occurs here
test.rs:6     let test = ||{
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(||{
test.rs:9             str = "Whatever";
test.rs:10             return
test.rs:11         });
           ...
test.rs:20:5: 20:9 error: cannot move out of `read` because it is borrowed
test.rs:20     read.unwrap()();
               ^~~~
test.rs:6:16: 16:6 note: borrow of `read` occurs here
test.rs:6     let test = ||{
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(||{
test.rs:9             str = "Whatever";
test.rs:10             return
test.rs:11         });
           ...
test.rs:20:5: 20:9 error: use of moved value: `read`
test.rs:20     read.unwrap()();
               ^~~~
test.rs:18:5: 18:9 note: `read` moved here because it has type `core::option::Option<[closure(())]>`, which is non-copyable
test.rs:18     read.unwrap()();
               ^~~~
test.rs:8:22: 11:10 error: `str` does not live long enough
test.rs:8         write = Some(||{
test.rs:9             str = "Whatever";
test.rs:10             return
test.rs:11         });
test.rs:3:26: 21:2 note: reference must be valid for the block suffix following statement 0 at 3:25...
test.rs:3     let mut write  = None;
test.rs:4     let mut read = None;
test.rs:5
test.rs:6     let test = ||{
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(||{
          ...
test.rs:7:29: 16:6 note: ...but borrowed value is only valid for the block suffix following statement 0 at 7:28
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(||{
test.rs:9             str = "Whatever";
test.rs:10             return
test.rs:11         });
test.rs:12         read = Some(||{
           ...
test.rs:12:21: 15:10 error: `str` does not live long enough
test.rs:12         read = Some(||{
test.rs:13             println!("{:?}",str);
test.rs:14             return
test.rs:15         });
test.rs:4:24: 21:2 note: reference must be valid for the block suffix following statement 1 at 4:23...
test.rs:4     let mut read = None;
test.rs:5
test.rs:6     let test = ||{
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(||{
test.rs:9             str = "Whatever";
          ...
test.rs:7:29: 16:6 note: ...but borrowed value is only valid for the block suffix following statement 0 at 7:28
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(||{
test.rs:9             str = "Whatever";
test.rs:10             return
test.rs:11         });
test.rs:12         read = Some(||{
           ...
error: aborting due to 7 previous errors
```

I am sorry to report, one of the big annoyances of Rust is dealing with these long impenetrable novels the compiler writes us about what is wrong with our code.  Where do we even start?

Well, the first place we start is making test mutable, to avoid the error `test.rs:17:5: 17:9 error: cannot borrow immutable local variable 'test' as mutable`.

The second thing we do is to solve this 'str does not live long enough' business.

When a closure captures something, it can either do it "by borrow" or "by move".

1.  "by borrow" means essentially that the closure gets a reference to the enclosing stack frame.  However, the `read` and `write` functions are used outside the stack frame of `test`, so that's no good.  This is essentially what the `str does not live long enough` is about.  
2.  "by move" means that the values that are captured are *moved into* the closure, and thus may exist independently of the stack frame.

With these changes in hand, we have a new attempt:

```rust
fn main() {

    let mut write  = None;
    let mut read = None;

    let mut test = move ||{
        let mut str = "Test";
        write = Some(move ||{
            str = "Whatever";
            return
        });
        read = Some(move ||{
            println!("{:?}",str);
            return
        });
    };
    test();
    read.unwrap()();
    write.unwrap()();
    read.unwrap()();
}
```

This also fails, but with less errors.  We're making progress:

```
test.rs:18:5: 18:9 error: use of moved value: `read`
test.rs:18     read.unwrap()();
               ^~~~
test.rs:6:25: 16:6 note: `read` moved into closure environment here because it has type `[closure(())]`, which is non-copyable
test.rs:6     let mut test = move ||{
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(move ||{
test.rs:9             str = "Whatever";
test.rs:10             return
test.rs:11         });
           ...
test.rs:16:6: 16:6 help: perhaps you meant to use `clone()`?
test.rs:19:5: 19:10 error: use of moved value: `write`
test.rs:19     write.unwrap()();
               ^~~~~
test.rs:6:25: 16:6 note: `write` moved into closure environment here because it has type `[closure(())]`, which is non-copyable
test.rs:6     let mut test = move ||{
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(move ||{
test.rs:9             str = "Whatever";
test.rs:10             return
test.rs:11         });
           ...
test.rs:16:6: 16:6 help: perhaps you meant to use `clone()`?
test.rs:20:5: 20:9 error: use of moved value: `read`
test.rs:20     read.unwrap()();
               ^~~~
test.rs:6:25: 16:6 note: `read` moved into closure environment here because it has type `[closure(())]`, which is non-copyable
test.rs:6     let mut test = move ||{
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(move ||{
test.rs:9             str = "Whatever";
test.rs:10             return
test.rs:11         });
           ...
test.rs:16:6: 16:6 help: perhaps you meant to use `clone()`?
error: aborting due to 3 previous errors
```

You will recall earlier that *moving* means that the old references become invalid.  That is also true here.

The difficulty here is that with `read` and `write` now `move`d into our `test` closure, from outside the references become invalid.  So that's no good.

So we'll move our usage of `read` and `write` inside.  It doesn't make a semantic difference:

```rust
fn main() {

    let mut write  = None;
    let mut read = None;

    let mut test = move ||{
        let mut str = "Test";
        write = Some(move ||{
            str = "Whatever";
            return
        });
        read = Some(move ||{
            println!("{:?}",str);
            return
        });
        read.unwrap()();
        write.unwrap()();
        read.unwrap()();
    };
    test();

}
```

```
test.rs:18:9: 18:13 error: use of moved value: `read`
test.rs:18         read.unwrap()();
                   ^~~~
test.rs:16:9: 16:13 note: `read` moved here because it has type `core::option::Option<[closure(())]>`, which is non-copyable
test.rs:16         read.unwrap()();
                   ^~~~
error: aborting due to previous error
```

Now we're down to just one error, although this requires some explaining.  For now, understand that calling `.unwrap()` on an `Option` incurs a `move`.  (This is because it's declared `self`, see the section on Functions below.) So we cannot unwrap twice, because that would require us to `move` twice.

However we can unwrap into a temporary variable, once.  And use that unwrapped value both times.  And now it's fine:

```rust
fn main() {

    let mut write  = None;
    let mut read = None;

    let mut test = move ||{
        let mut str = "Test";
        write = Some(move ||{
            str = "Whatever";
            return
        });
        read = Some(move ||{
            println!("{:?}",str);
            return
        });

        let r = read.unwrap();
        r();
        write.unwrap()();
        r();
    };
    test();

}
```

Although the output may be unexpected:

```
"Test"
"Test"
```

This is because **moving closures in Rust have different environments**.  A moving closure (e.g. `move || {...}`) essentially works from a cloned environment, frozen in time exactly as it was when the closure was declared.

Swift closures, in contrast, have shared environments.  Making changes to the environment in one closure may affect another closure that shares the environment.

Rust's non-moving closures (e.g. `|| {...}`) technically have a shared environment, but it is hard to notice, because the borrow checker makes it difficult for 2 closures to read and write to the same parts of the environment.  As we saw in this example.

If you wanted to match the original semantics for some reason, here's a working solution:

```rust
fn main() {
    let mut write  = None;
    let mut read = None;
    {
        let mut test = || {
            let str = Rc::new(Cell::new("Test"));
            write = {
                let str = str.clone();
                Some(move || {
                    str.set("Whatever");
                })
            };
            read = Some(move || {
                println!("{:?}", str.get());
            });
        };
        test();
    }
    let read = read.unwrap();
    read();
    write.unwrap()();
    read();
}
```

Note that the Rust code is a lot more explicit about what is going to happen than the Swift code is.  The Swift behavior surprises advanced developers, and for that reason makes a good interview question.  This Rust code's output is predictable by more people.

# Arrays

Rust's arrays are of fixed size, and the size is in fact part of the array's type: `[i32; 2]` means 2, 32-bit ints.  A mutable array in Rust means that the elements can be swapped.  But elements cannot be appended, because the array size is fixed.

Rust's `Vec` type is dynamically-sized and therefore is closer to the Swift array.

# Preprocessor

Rust has a preprocessor that is stupidly powerful (*way* more powerful than C's), whereas Swift has none.  Read the Rust book for more details on `macro_rules!`, the preprocessor function.

# Functions (self, &, &mut)

A function can be defined in 3 ways.

## self

```rust
fn moving(self);
```

is a *moving* function.  This means that the ownership of the thing is transferred into the function.  (So a lot like a moving closure)  e.g.

```rust
#[derive(Debug)]
struct SomeType;

impl SomeType {
    fn moving(self) {

    }
}

fn main() {
    let f = SomeType;
    f.moving();
    println!("{:?}",f); //error: use of moved value 'f'.
}
```

On the plus side however, the function can assume nobody else has a reference to it.  This is very useful in some circumstances.  

### Examples

1.  If you want to ensure a function cannot be called twice (for example: `free` or `release`).  In Rust, this can be forced at compile time by using a moving function.
2.  If you want to take something that is not entirely threadsafe and use it on a different thread.  Since you know there are no other references, you can move it onto a thread without worry that somebody else might accidentally use it.
3.  You can also use this as a trick to ensure that the value "goes out of scope".

For example if we have a

```rust
fn encrypt(&mut T) { }
fn add_more_data(&mut T) { }

let t = T;
encrypt(&mut t);
//many screenlengths
//of code
//later
add_more_data(&mut t);
```

This code is "unsafe" in a way that the Rust compiler does not understand; it goes to show that "Rust safety" is not the same as *real* safety.  The trouble here is that we encrypted our `T`, but then later (perhaps much later, perhaps many versions of the program later) we added more data to it.  If we expected at the end of this program to have an encrypted `T` and instead we only have a "sorta" encrypted `T`, then that is bad news.  "sorta" encryption is the worst kind of encryption.

This is because Rust does not understand what an "encrypt" function is, it's just a function same as `add_more_data` and you can call them in any order and Rust doesn't care.

However we could tell Rust that `encrypt` is special and not just any old function by making it a *moving* function. This indicates that once we encrypt the `T`, nobody should touch it:

```rust
fn encrypt(T) { }
fn add_more_data(&mut T) { }

let t = T;
encrypt(t); //move
//many screenlengths
//of code
//later
add_more_data(&mut t); //error
```

Now the nightmare of disclosing a huge security issue collapses into a compile error.  So that's progress.

The insight here is that "Rust safety" is a term of art and does not mean your program does not have serious issues.  However very often you can make a small change (here, making `encrypt` a *moving function*) and suddenly Rust is auditing your programs much more thoroughly.  But you, the programmer, have to know how to make that happen.  It doesn't happen on its own.

On the plus side, once we've clued Rust in about `encrypt`, it does all the tedious work of making sure that we don't add plaintext to our encrypted `T`, which is a very tedious and error-prone thing to audit by hand.  So in the right hands Rust really can run circles around the safety of other languages.

Most of the time however the moving function is too restrictive.  Instead you probably want:

## Borrow

We also write a function that `borrows` self:

```rust
#[derive(Debug)]
struct SomeType;

impl SomeType {
    fn borrowing(&self) {

    }
}

fn main() {
    let f = SomeType;
    f.borrowing();
    println!("{:?}",f);
}
```

This is perfectly fine.  However, a borrowed reference has some limitations:

1.  You cannot upgrade it to a move later on.  You are only borrowing the thing, not owning it.  (You can, however, make a copy, a copy that you then own, if that's your thing.)
2.  Since you are only borrowing the thing, you must "return" it at some point, generally when the function is done executing.  Importantly, Rust will not let you squirrel away a borrowed reference into a global variable or somewhere that will potentially outlive the thing you are borrowing.
    1.  You can try and convince Rust that your location will not *really* outlive the borrowed reference by specifying [lifetimes](http://doc.rust-lang.org/book/ownership.html#lifetimes) for both.  All Rust variables have a *lifetime*, but sometimes being explicit can help the compiler understand what you are doing when you are squirreling away borrowed pointers.

## Mutable borrow

Finally, there is a mutable borrow:


```rust
fn mutable_borrow(&mut self);
```

This is exactly like borrow, except the function can now mutate self, whereas earlier it was immutable.

## Overall

Generally speaking, if you're writing a function:

1.  First try borrow (e.g. `&self`)
2.  If you need mutation, try mutable borrow (`&mut self`)
3.  Failing everything else, try a moving function (`self`).  This is mostly useful when you need to call other moving functions, or you are going to do weird things with threads.

## Interactions

You may ask, for example, what would happen if you tried to call a moving function when somebody had *borrowed* `f`?  The answer is we cannot move it, if `f` is still being borrowed:

```rust
#[derive(Debug)]
struct SomeType;

impl SomeType {
    fn moving(self) {

    }
}

fn main() {
    let f = SomeType;
    let g = &f; //note: borrow of `f` occurs here
    f.moving(); //error: cannot move out of `f` because it is borrowed
}
```

However, if our borrow goes out of scope, we can:

```rust
#[derive(Debug)]
struct SomeType;

impl SomeType {
    fn moving(self) {

    }
}

fn main() {
    let f = SomeType;
    {
        let g = &f;
    }
    f.moving();
}
```

## Don't copy that floppy

You may be puzzled to find out that if `SomeType` is an array instead of a struct, the error caused by "borrow and move" goes away:

```rust
type SomeType = [u8; 10];

fn moving(kinda_like_self: SomeType) {
    println!("{:?}",kinda_like_self);
}

fn main() {
    let f = [0; 10];
    let g = &f; 
    moving(f); //no error here
}
```

At first it seems like the big change here is that we have a different kind of function than earlier because instead of saying `f.moving()` we say `moving(f)`.  This actually has little to do with it.  To say `f.moving()` we would have to insert the `moving` function inside the `Array` type, which you can do via Traits as we talked about earlier but that is just more syntax.

What is actually happening here is that `SomeType` now implements the `Copy` trait.  Arrays derive (all of the?) traits their underlying elements have, and since `u8` has `Copy`, an array of `u8` has `Copy` as well.

And traits that have `Copy` do not have move semantics.  They instead have "copy semantics" which is basically ordinary semantics if you're used to Swift.  It lifts this move restriction that we've been talking about.

In the [API documentation](http://doc.rust-lang.org/std/marker/trait.Copy.html) they have a ridiculously circular and contradictory explanation of how you should use this feature, e.g., that all traits should be `Copy` except the ones that shouldn't, and only implement `Copy` when the thing should be copied.

A much better explanation is this.  If your type has a pointer inside it (or a thing that has a thing that has a pointer) then it can't be copied and the compiler will stop you from implementing `Copy` on it.  This is because you'd end up with two strong references to that thing.

Meanwhile you may find in some cases the additional discipline of move semantics is useful.  We've talked about some cases like with functions that "consume" `self` and/or their arguments, to make them deliberately unavailable later in the program as a way to avoid over-release and similar.

Between those two cases, you have a lot of latitude about whether the type should use move semantics or copy semantics.  Personally, I tend to use move semantics until I hit a compile error, and then I make more of a firm decision.

Another performance note.  `Copy` is poorly named, it simply means that the type *can* be copied, i.e., as necessary.  Creating a new reference does not actually incur a copy; the semantics here are copy-on-write.


# Visibility

Swift's namespaces are kind of a joke.  You say `import Framework` and all of a sudden you get all of `Framework`'s classes into your current scope.  You can technically reference them with `Framework.Class`, but nobody does.

Rust on the other hand works on the basis of *modules*.  A file is a module.  Modules can contain other modules. Rust's *crate* (like a Swift *target*) is a module.  It's modules all the way down.

This means that stuff in Rust is much more hierarchically organized, even within a crate (target).  You are not just dumping every class from some framework into your current scope.  You are grabbing some module in some module in the framework, and using only that.  Even within your program, you are importing only a little bit of the rest of the program; not all of it.

Swift has three visibilitiy modifiers:

* `private`, which is *file-visible*
* `internal`, which is *target*-visible, exporting your thing everywhere in the target, and no imports are required to use it anywhere in the target.  This is the default.
* `public`, which is *globally*-visible.  This is `internal`, *plus*, it exports the symbol, so that it is useable by anyone linking to the library.  Of course, this requires external programs to explicitly import your library.

As a consequence of its hierarchical namespacing, Rust has only two visibility modifiers:

* the default, which is visible *within the current module*
* `pub`, which is exported to the parent module

There is no way to declare a function "target-visible" (crate-visible) or "globally-visible" as such.  Instead you declare it `pub`, and then you declare the module containing it `pub` in its parent module, and then that module `pub` in its parent module, and so on, until it's visible enough.  There has to be an unbroken link of `pub` between the user and where it's declared, for it to work.

I don't know why you would want to, but you can approximate Swift's flat namespacing by using `pub use`.  It's like regular `use` (which is like Swift's `import`), but it also exports the symbol.  So

```rust
//in library::module
pub fn foo() { ... }
```

```rust
//in library
pub use module::foo; //re-export foo in the library namespace
```

```rust
//in application
use library;
library::foo(); //now part of library namespace
```

# Properties

Rust has no concept of Swift properties.  You have fields and functions, and that's it.

You could probably roll your own with the preprocessor, if that's your cuppa tea.

# Dispatch

Swift's dispatch is largely an implementation detail, but is mostly static, unless you opt-in with `@dynamic` or `@objc`.

Rust's dispatch on the other hand is defined to be static in most cases.  This guarantees fast performance, but has some gotchas.

One case where it is dynamic is when you have something with the type of `MyTrait` but you don't know the real type, beyond the fact that it implements the trait.  For example

```rust
#[derive(Debug)]
struct SomeType;

trait MyTrait { 
    fn borrow(&self) {

    }
}

impl MyTrait for SomeType { }

fn main() {
    let st = SomeType;
    let st_as_trait : &MyTrait = &st;
    st_as_trait.borrow(); //dynamic dispatch
}
```

This has the consequence that *you cannot call a moving method on a variable that is known only to implement a trait*, which is somewhat nonintuitive.  It's because you need to know the size of the underlying type in order to call the moving function.  But we do not know the size, we know only that it implements some trait.

# Generics

Swift's generics and Rust's generics are very very similar.

One difference is that Swift's generic specialization is an implementation detail.  Whereas Rust is guaranteed to use monomorphization to completely specialize the generic.

In layman's terms, this means that the Rust compiler emits different code to implement `Vec<i32>.append(2)` than for `Vec<i64>.append(2)`.  This increases the size of your program, *but* ensures that these functions are always *statically* dispatched.  That provides some performance guarantees that Swift does not.

After pondering that for 10 minutes, you may wonder: how is that possible?  You could have a library with some generic function `MyFunc<T>(t: T)` and then you call that library from a completely different program, maybe on types that the library did not know about.  How then does the library contain a specialization for a type that it doesn't know anything about?

The answer is that the rust `rlib` format, in addition to native code, contains a kind of bytecode for generic functions.  So when you go to compile your program, that calls `MyFunc` on special types, the compiler can emit new specializations of the function in that already-compiled library.  Pretty neat, huh?

Actually, Swift does this too, although the only place you'll see that documented is this guide.

# Semicolons

with Swift, semicolons are optional, except if you have multiple statements on a line.

In Rust, semicolons are generally required, but actually there is a trick.  

A line that ends in a semicolon evaluates to `()`, which is pronounced *unit* and is basically the equivalent of C's `void`.  Meanwhile, a line without a semicolon evaluates to something else, generally.  So

```rust
fn test() -> i32 {
    3
}
```

returns 3 whereas

```rust
fn test() -> i32 {
    3;
}
```

does not compile.

You can also use `return 3;` but people will think you are a lamer if you end your function that way.  It is a little weird at first, but if you are doing a lot of `return function_call()` it is actually pretty handy just to write `function_call()` instead.  (On the other hand if later want to add more stuff to the end of your function it is not as fun.)

Note that Swift does have some concept of implicit returns, for short closures.  For example this:

```swift
let sema = dispatch_semaphore_create(0);
dispatch_async(dispatch_get_main_queue()) {
    dispatch_semaphore_wait(sema, DISPATCH_TIME_FOREVER)
}
```

actually fails to compile.  Because secretly `dispatch_semaphore_wait` returns `Int`, and in Swift 1-line closures automatically return whatever the line evalutes to, but `dispatch_async` wants a closure that returns `void`.  Generally you fix this by adding return:

```swift
let sema = dispatch_semaphore_create(0);
dispatch_async(dispatch_get_main_queue()) {
    dispatch_semaphore_wait(sema, DISPATCH_TIME_FOREVER)
    return
}
```

to force the closure to return `void`.

Rust however does not change the rules depending on length like Swift does.  **All functions return the last line regardless of length**.  If the last line ends with a semicolon your function will return `()` and if not it will return whatever that line was.

Since `if` is an expression in Rust, not a statement, clever semicolon placement allow its use as an rvalue:

```rust
let y = if (x > 0) { 10 } else { compute(x) };
```


# Threading

If you are used to Swift, you are probably doing a lot of `dispatch_async`, often.  Unfortunately... Rust takes a dim view of this.

## Scoped

The simplest way to do threading in Rust is to use `thread::scoped`:

```rust
use std::thread;

fn main() {
    let guard = thread::scoped(|| {
        println!("Hello from a thread!");
    });

    // guard goes out of scope here
}
```

This construction is actually somewhat clever.  When the guard 'goes out of scope', its destructor is called, and its destructor waits for the thread to finish running.  That way we can be sure that the stack frame of `main` will be around for the full lifetime of the thread.  The function signature of `thread::scoped` makes this clear:

```rust
pub fn scoped<'a, T, F>(f: F) -> JoinGuard<'a, T> where T: Send + 'a, F: FnOnce() -> T, F: Send + 'a
```

Just for fun, I'll break this down:

`fn scoped<'a, T, F>` a function that will use generic parameters `T` and `F`, and generic lifetime parameter `'a`.  We are making up names here; they do not mean anything yet.

`(f: F) -> JoinGuard<'a, T>` taking one parameter `f` (of type `F`) and returning a `JoinGuard` with lifetime `'a` and of type `T`.  The `T` will be the return type of the closure.


`where T: Send + 'a`  Where `T` has the `Send` trait (indicating that we can move it onto a new thread.  Nearly all types have this, but some C types do not.) and lifetime `'a`.  Importantly, we are saying here that the `JoinGuard` and `T` have the same lifetime, that is, that the `T` is not going to vanish while it is inside some `JoinGuard`.

`F: FnOnce() -> T`, Here we say that `F` is a closure that returns type `T`.  `FnOnce` means that the closure will only be called once, and after it is called, it is no longer valid.  This is to avoid the situation where a closure calls foo.move() once, but then *the closure itself is run twice*, thereby causing a double move.  `FnOnce` can call moving functions on captured variables, whereas alternate closure types `Fn` and `FnMut` cannot.

`F: Send + 'a` `F` (our closure) has the `Send` trait, indicating that it can be sent to a new thread, and it also has the `'a` lifetime.  In the context of a closure, a lifetime means *the environment's* lifetime.  So now we are saying that the *environment* (a.k.a. the *stack frame*) cannot vanish out from under our `JoinGuard`.

As a result you cannot capture anything in the closure that does not definitely exist until the JoinGuard is destroyed, and the JoinGuard will wait for the thread to complete, meaning that our function will block until the thread is done.

If you come from a Swift background, that is probably not what you expected.  You expected to start a thread and then the function would return.  You can in fact do that, if you return *the `JoinGuard`*.  But at some point you have to wait on the `JoinGuard`.

## Spawn

Another thing you can do is `thread::spawn`.

```rust
use std::thread;
use std::old_io::timer;
use std::time::Duration;

fn main() {
    thread::spawn(move || {
        println!("Hello from a thread!");
    });

    timer::sleep(Duration::milliseconds(50)); //we have to sleep here or otherwise our program will exit before printing.
}
``` 

This is closer to idiomatic threading in Swift.  However, you can pretty much only use it with *moving* closures (since the environment may not outlive the thread and must be moved in).

The function signature of `spawn` is also a lot simpler:

```rust
pub fn spawn<F>(f: F) -> JoinHandle where F: FnOnce(), F: Send + 'static
```

The difference here is that `F` gets the special lifetime `'static`.  (Again in the context of closures, the lifetime applies to the environment.)  Here we are saying that the environment must contain only static data (like global constants), or data moved into the closure.  It cannot contain for example a borrowed reference, unless that borrowed reference is to something `'static`, because what if the borrowed reference goes out of scope.

And that's the real trick.  Swift programmers are used to putting all kinds of things in closures, but Rust takes a dim view.  Generally speaking, there are a few solutions:

### Clone

You could `clone` the borrowed reference, and then move in the clone.  This works pretty well for immutable data.

One gotcha is that the type itself must implement `clone`.  By default, calling `clone()` does something [kind of stupid](http://stackoverflow.com/questions/28816426/copying-a-struct-for-use-on-another-thread/28816987?noredirect=1#comment45906109_28816987), so unless someone has manually derived the `Clone` trait, or unless it's in a place where you can easily do it, it kind of sucks.  

### Arc and a Mutex:

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::old_io::timer;
use std::time::Duration;

fn main() {
    let data = Arc::new(Mutex::new(vec![1u32, 2, 3]));

    for i in 0..2 {
        let data = data.clone();
        thread::spawn(move || {
            let mut data = data.lock().unwrap();
            data[i] += 1;
        }); //unlocked here
    }

    timer::sleep(Duration::milliseconds(50));
}
```

By wrapping our data in an `Arc` we can move a `clone()` of the `Arc` into the closure.  (Only the `Arc` is `clone`d, not the underlying data.  Recall `clone()` on `Rc`/`Arc` is like getting a strong pointer in Swift.)

Since the `Arc` is reference-counted, the underlying data will not disappear until all the strong pointers go away, so we are fine on that front.

The `Mutex` here is used to satisfy the compiler that we will not be violating some thread safety issue inside `Vec`, which the compiler does not know to be threadsafe.  The `Mutex` is `lock`ed explicitly, and it is unlocked when `lock()`'s return value goes out of scope.  As such, only one thread can be accessing the `Vec` at a time.

### Channels

Also, we can replace our `sleep` with a `channel`.  `channel` is a bit like `dispatch_semaphore_t` in Swift, except it also sends data.  (There is also a 'real' semaphore implementation in the standard library, although it is not widely used.)

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::sync::mpsc;

fn main() {
    let data = Arc::new(Mutex::new(0u32));

    let (tx, rx) = mpsc::channel();

    for _ in 0..10 {
        let (data, tx) = (data.clone(), tx.clone()); //these clones will both be moved into our closure

        thread::spawn(move || {
            let mut data = data.lock().unwrap();
            *data += 1;

            tx.send(()); //send a union ()
        });
    }

    for _ in 0..10 {
        rx.recv(); //receive 10 values before exiting
    }
}
```

