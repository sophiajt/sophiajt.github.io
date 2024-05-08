+++
title = "The search for easier safe systems programming"
[taxonomies]
tags = [ "programming" ]
+++

I've been involved in the Rust project in some form or another since 2016, and it's a language I'm very comfortable using. Many Rust programmers could [say the same](https://blog.rust-lang.org/2024/02/19/2023-Rust-Annual-Survey-2023-results.html). But, if we take a step back and are honest with ourselves, we'd admit that the road to getting to that level of comfort was difficult.

I taught Rust professionally for two years. Watching the faces of people trying to learn Rust for the first time reminded me just how hard this language is to learn.

After two years of that, I wanted to answer a question I wasn't entirely sure had an answer: _Is it possible to make an easy-to-use, easy-to-learn, and easy-to-teach safe systems language?_ Could I put my career working on programming languages (TypeScript, Rust, Nushell, etc) to use and find a solution?

# Enter June

For the last year and a half, I and my recently-added collaborator Jane Losare-Lusby have been working in secret on a safe systems language that could be learned about as quickly as one can learn Go. I think we might have something worth exploring.

## Changing how we think of memory

In Rust, we think of each piece of memory as having its own lifetime. Each of these lifetimes must be tracked, sometimes leading to rather complex code, complex error messages, and/or complex mental models of what is happening. The complexity of course comes with the benefit of being highly precise about each and every piece of memory and its reclamation.

Using a Rust example:

```rust
struct Node {
    data1: &Data
    data2: &Data
    data3: &Data
}
```

Rust developers will spot right away that this is an incomplete example. We need two more things: Lifetime Parameters and Lifetime Annotations. Adding those, we get:

```rust
struct Node<’a, ‘b, ‘c> {
    data1: &’a Data
    data2: &’b Data
    data3: &’c Data
}
```

The concept count for this example ends up being pretty substantial. Counting them off, we get:

- Lifetimes
- Lifetime annotations
- Lifetime parameters
- Ownership and borrowing
- Generics

When I showed examples like this to my class when I taught Rust, I had to walk them through each of those concepts first before I could show the full example.

The question then is: can we make this easier?

## What if memory was grouped?

What if instead of having to track every piece of memory's lifetime separately, we let groups of related allocations share a lifetime?

Effectively, this would mean that a data structure, like a linked list, would have a pointer pointing to the head which has a lifetime, and then every node in the list you can reach from that head has the same lifetime.

There are some benefits to this approach, as well as a few drawbacks. Let's take a look at the benefits first.

## Benefits of grouped allocations

Exploring grouped allocations, we noticed some immediate benefits. The first is that we could treat all user-defined values as pointers, and these pointers could also represent their own lifetimes (without needing lifetime parameters). This makes the code feel quite a bit lighter:

```rust
struct Node {
    data: i64
    next: Node?
}

let n = new Node(data: 123, next: none)
```

Since all user data is pointers, we can use the name of the type to mean "pointer to this structured data".

The next thing we noticed is that both lifetimes and inference for lifetimes becomes significantly simpler.

Let's take a variation of the example above:

```rust
struct Node {
    data: i64
    next: Node?
}

fun do_this() {
    let n = new Node(data: 123, next: none)
}
```

We can infer that the allocation that creates `new Node(...)` has a lifetime and what it is. Because this allocation never "escapes" the function - that is, it never leaves the function in any way - then we can call its lifetime "Local".

As we'll find out, each of the lifetime possibities is a readable name that we can show the user in error messages. It also makes things significantly easier to teach.

Let's look at another example to see a different lifetime.

```rust
struct Stats {
    age: i64
}

struct Employee {
    name: c_string,
    stats: Stats,
}

fun set_stats(mut employee: Employee) {
    employee.stats = new Stats(age: 33)
}

fun main() {
    mut employee = new Employee(name: c"Soph", stats: new Stats(age: 22))
    set_stats(employee)
    println(employee.stats.age)
}
```

A bit of a longer example this time, but let's focus on this function:

```rust
fun set_stats(mut employee: Employee) {
    employee.stats = new Stats(age: 33)
}
```

What's the lifetime of this `new Stats(..)` allocation? In this example, we do see the new pointer escape the function via a parameter. We can also give this a readable lifetime: `Param(employee)`

In all, we have three lifetimes an allocation can have:

- Local
- Param(xxxx)
- Return

## Any data structure you want

Another big advantage of grouping our allocations is that we no longer have to worry about a drop order. This means we can think of the whole thing as dropping all at once. For large structures, that can be a speed-up over languages with a required drop order.

Additionally, we get another major benefit. We can now create arbitrary data structures.

```rust
struct Node {
    data: i64
    next: Node?
}

mut node3 = new Node(data: 3, next: none)
let node2 = new Node(data: 2, next: node3)
let node1 = new Node(data: 1, next: node2)

node3.next = node1
```

And just like that, we've made a circular linked list. Creating a similar example in Rust is certainly more of a challenge.

But, something fishy is going on here.

To make the above work, we're using shared, mutable pointers. This is explicitly forbidden in Rust. Why is it okay here?

## The dangers of shared, mutable pointers

Rust disallows holding two mutable references to the same memory location and for good reason. Well, multiple reasons actually.

First, having two copies of a mutable pointer where two separate threads each hold a copy means we have the possibility for a race condition. This can leave us with incoherent data that's difficult to debug.

Second, even if these two multiple pointers are limited to the same thread, we get what we might call "spooky action at a distance". The modification of one pointer is then visible to the holder of the other pointer, which might be far away from the source of the mutation.

For us to reasonably use shared, mutable pointers, we need to tame both of these. The first issue, the race condition, is easy enough: we prevent sending shared, mutable pointers between threads. This limits them to a single thread.

The second issue is decidedly harder. There have been many attempts at ways of handling this through rules enforced by the type system.

In June, we're trying something a bit different. We'll let developers use shared, mutable pointers, but then offer a "carrot" to opt-in to restrictions around using them. The carrot ends up pulling from a classic technique of software engineering: encapsulation.

## The full power of encapsulation

In traditional encapsulation, programmers make a kind of "best effort" to hide implementation details from the world around them. Keeping private state private grants the benefits of better code reuse, ease of updating implementation details, and more.

But as often is the case, if that kind of rule isn't enforced, over time APIs get designed where internal implementation details leak out.

Something very interesting happens if we don't allow this to happen. If an encapsulation can be checked by the compiler, and the compiler enforces that no private details leak, we have what you might call "full encapsulation".

These kinds of encapsulations wouldn't allow any aliasing of pointers into them. They'd have their internal pointers fully isolated from the rest of the program.

Once we have this, some new capabilities start opening up:

- We can "fence off" our shared, mutable pointers, making it possible to create single-owner encapsulations that can be sent safely between threads.
- We can lean people in the direction of cleaner API design, as now we have a way to truly keep private implementation details private.
- We can handle some of the drawbacks of grouped allocations.

What kind of drawbacks, you might ask? It's high time we talked about them.

## Drawbacks of grouped allocations

If we go back to our earlier example and look carefully, we'll notice something:

```rust
fun set_stats(mut employee: Employee) {
    employee.stats = new Stats(age: 33)
}

fun main() {
    mut employee = new Employee(name: c"Soph", stats: new Stats(age: 22))
    set_stats(employee)
    println(employee.stats.age)
}
```

The question is: what happened to the `new Stats(age: 22)` allocation?

Remembering that June is a systems language, we can't say "the garbage collector handled it" because we have no garbage collector. Nor can we say "the refcount hit zero, so we reclaimed it" as we don't use refcounting. As a systems language, we can't allow hidden or difficult-to-predict overhead to happen.

It's not actually leaked either, as even the memory it occupies will be reclaimed once the entire group is reclaimed. For all intents and purposes, though, it's lost to the user until the group is no longer live. It's a kind of "memory bloat" that happens if we group our allocations.

To handle this, we'll need some way of recycling that memory. I say "recycling" specifically because in June we can't free the memory, as the group is treated together as a single entity where all the allocations in the group are freed at once. If we instead recycle the memory, we can reuse that same memory while the group is live.

Techniques to do this have been around for decades, and often people use "free lists" to keep a list of nodes that have been recycled, so they can be reused when the next allocation happens.

The problem with free lists is that they aren't safe. If you're not careful, you'll create a security vulnerability and/or an incredibly hard bug to find.

Instead, we need to build in a safe way of recycling memory into the language.

## Safe memory recycling

Using the idea of full encapsulation from earlier, we can create "fenced in" sets of pointers that we know aren't shared with the rest of the world. Once we have them, it's possible to track the pointers inside. These pointers can get a "copy count", so we know how many copies are live at any point in time (not dissimilar from a refcount, though this has no automatic reclamation).

Once we have a copy count for each internal pointer, we can give developers a built-in `recycle` command.

```
let x = new Foo()
recycle x
```

Recycling would start at the given pointer and would check the pointers reachable from it. Each pointer it finds that it can recycle would go into the safe free list.

You might wonder "why not do this automatically?". There are a couple reasons:

- The operation is linear time based on your transitively-reachable pointers. This means you may incur a noticeable overhead when recycling
- Because of the first point, it's important to make places where this occurs visible

If this sounds like a kind of manual garbage collection, you're right. My collaborator Jane calls this "semi-automatic" memory reclamation. You ask once, and when you ask you get a kind of highly focused mark and sweep for that single pointer and the pointers reachable from it.

_Note: this feature is not yet in the reference compiler. We're hoping to implement it in the coming weeks._

## More work ahead

We have a way of simplifying lifetimes, making for readable code that people from various languages should be able to understand and use. We can also give clear, easy-to-understand lifetime errors when they arise.

Having safe memory recycling gives us a way to keep groups and still offer things like `delete` in a linked list abstraction. It's convenient but not so automatic that we lose the visibility into the costs of memory management.

That said, there are still some challenge ahead that will need to be solved in the language design and tooling. For example, how do you know when the program is bloating memory? We'll need some way of doing a memory trace when the program is running to detect this and warn the developer.

I see this in a way as a more incremental/prototype-friendly way of development. June is always memory safe, but the first version of a program may not be as efficient as it could be in terms of memory usage. That's a process we often go through as developers. First, we "make it work" before we "make it good".

In June, we keep it lightweight as we keep your programs memory safe, and then we provide tools and support for incrementally improving code.

## Future possibilities

### Relationship to Rust

June has a real opportunity to be a good complement to Rust. Rust's focus on embedded and system's development is a core strength. June, on the other hand, has a lean towards application development with a system's approach. This lets both co-exist and offer safe systems programming to a larger audience.

An even better end state requires Rust to have a stable ABI. Once it does, June will be able to call into Rust crates to get the benefits of Rust's substantial crate ecosystem. We're looking forward to collaborating on this in the future.

### Going beyond OOP

OOP has for decades been the way many applications are written, but it's not without its flaws. Many OOP languages allow programmers to freely break good rules of thumb, like the Liskov substitution principle, or to create a mess of interwoven code between parent and child classes that's difficult to maintain.

We're currently investigating other ways of making code reuse easier, more modular, and more composible. We're not quite ready to talk about this, though we hope to soon.

### Exploring related work

Over the years, there have been a [number of memory management techniques tried](https://verdagon.dev/grimoire/grimoire), including many that lie outside of the ones commonly found in languages today. We'd like to explore these more deeply to see which, if any, may help June.

## Acknowledgements

We had the help of dozens of experts in various fields as we brainstormed the initial design for June, and for their contributions, we're thankful. We'd especially like to thank the collaborators who went above and beyond with their time across multiple brainstorming sessions to help June grow to where it is.

- Andreas Kling
- Doug Gregor
- Jason Turner
- Mads Torgersen
- Mae Milano
- Steve Francia

Also, special thanks to our private beta testers for testing out June and giving us feedback.

## Checking it out

Documentation on the June language and the June reference compiler are now available via the [June repo](https://github.com/sophiajt/june).

Please note: the reference compiler is pre-alpha quality.
