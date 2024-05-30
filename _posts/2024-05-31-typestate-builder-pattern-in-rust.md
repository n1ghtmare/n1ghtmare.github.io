---
layout: post
title: "Typestate builder pattern in Rust"
description: "In this post we will what the typestate builder pattern is and how to use it in Rust"
date: 2024-05-31
tags: [programming, rust]
comments: false
share: true
---

Greetings fellow Rustaceans 🦀! 

Today, I'll try to describe how you can leverage the typestate pattern
alongside the well-known (and beloved) builder pattern to enforce compile-time
checks, ensuring that the builder is used correctly. I'll try to keep it as
straightforward as possible.

### The Builder Pattern

If you've used Rust for a while, chances are you've come accross the builder
pattern. It's a staple in many popular crates such as, for example,
[reqwest](https://docs.rs/reqwest/latest/reqwest/index.html),
[serde](https://docs.rs/serde/latest/serde/trait.Serialize.html) and
[clap](https://docs.rs/clap/latest/clap/index.html) to name a few.

Essentially the builder pattern is a design pattern that provides a flexible
and clear way to construct complex objects. This pattern is especially useful
when an object must be created with a specific configuration of many possible
parameters, some of which may be optional. In the builder pattern, instead of
using numerous constructors, the object to be built is constructed using a
separate __builder object__. It's typical for a builder object to have
__chaining methods__ (these methods set up configuration parameters and return
the same builder object to allow for method chaning) and a __build method__
(after all the desired parameters are set, this method is called to construct
the final object).

Consider this simple example struct that we intend to construct with a
dedicated builder. Keep in mind that this is just a concise demonstration;
typically, a builder is more suited to constructing larger structs where its
benefits become more apparent.

```rust
struct ServiceEntry {
    name: String,
    enabled: bool,
    logs_max_size: u128,
}

impl ServiceEntry {
    pub fn report(&self) {
        println!("Service details:");
        println!("name: {}", self.name);
        println!("enabled: {}", self.enabled);
        println!("logs max size: {}", self.logs_max_size);
    }
}
```

We have a `ServiceEntry` with a few properties and a single `report` method that just prints them.

Now let's define a builder for it.

```rust
struct ServiceEntryBuilder {
    name: Option<String>,
    enabled: bool,
    logs_max_size: u128,
}
```

Typically a builder will have a constructor with some default values, then we have the chaining methods and finally the build method.

```rust
impl ServiceEntryBuilder {
    pub fn new() -> Self {
        Self {
            name: None,
            enabled: true,
            logs_max_size: 10 * 1024,
        }
    }

    pub fn name(mut self, name: impl Into<String>) -> Self {
        self.name = Some(name.into());
        self
    }

    pub fn enabled(mut self, enabled: bool) -> Self {
        self.enabled = enabled;
        self
    }

    pub fn logs_max_size(mut self, logs_max_size: u128) -> Self {
        self.logs_max_size = logs_max_size;
        self
    }

    pub fn build(self) -> ServiceEntry {
        ServiceEntry {
            name: self.name.unwrap(),
            enabled: self.enabled,
            logs_max_size: self.logs_max_size,
        }
    }
}
```

We want to consume (or take ownership of) `self` in the `build` method in order
to ensure that it cannot be used once the method has been called. Also notice
that for `name` we take "anything that implements `Into<String>`" (isn't Rust
cool?!).

So far so good - pretty simple. 

Here is how our builder is used:

```rust
fn main() {
    let service = ServiceEntryBuilder::new()
        .enabled(false)
        .logs_max_size(20 * 1024)
        .name("my-service")
        .build();

    service.report();
}
```

Prints:

```sh
$ cargo run -q
Service details:
name: my-service
enabled: false
logs max size: 20480
```


Now, if you're a keen observer you've already spotted a pretty substantial
issue in the `build` method. Namely the `.unwrap()` on `name`.

```rust
impl ServiceEntryBuilder {
    // ...

    pub fn build(self) -> ServiceEntry {
        ServiceEntry {
            name: self.name.unwrap(), // Can panic if it's not set. Not good!
            enabled: self.enabled,
            logs_max_size: self.logs_max_size,
        }
    }
}
```
    
If we omit the name and try to build again our application will panic.

```rust
fn main() {
    let service = ServiceEntryBuilder::new()
        .enabled(false)
        .logs_max_size(20 * 1024)
        // .name("my-service")
        .build();

    service.report();
}
```

Prints:

```sh
$ cargo run -q
thread 'main' panicked at src/main.rs:50:29:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Wouldn't it be cool if we can force `name` to be provided on compile time
instead of panicking during runtime?

We can do that using the type system.

### The Typestate Pattern

We need a way to prevent build to be called unless a name has been provided. So
let's create a new type to represent this state for us:

```rust
struct NoName;
```

Next we will make our builder to be generic over the type we want to enforce.


```rust
struct ServiceEntryBuilder<T> {
    name: T,
    enabled: bool,
    logs_max_size: u128,
}
```

Next, let's move the implementation with the methods for `enabled` and `logs`
to work with any type of `name`. Essentially we don't want to change them, so
we move them to the most generic implementation of the builder.

```rust
impl<T> ServiceEntryBuilder<T> {
    pub fn enabled(mut self, enabled: bool) -> Self {
        self.enabled = enabled;
        self
    }

    pub fn logs_max_size(mut self, logs_max_size: u128) -> Self {
        self.logs_max_size = logs_max_size;
        self
    }
}
```

Now, we move our constructor to a concrete implementation that has `name` be of
type `NoName` (think of it as _starting_ with the `NoName` type "state").

```rust
impl ServiceEntryBuilder<NoName> {
    pub fn new() -> Self {
        Self {
            name: NoName,
            enabled: true,
            logs_max_size: 10 * 1024,
        }
    }
}
```

We want to express (using our type system), that after a user has provided us
with a name we "transition" to a different type "state" (you can think of it as
a sort of a state machine) that now has a `name`. Let's see how this can be
done.


```rust
impl ServiceEntryBuilder<NoName> {
    pub fn new() -> Self {
        Self {
            name: NoName,
            enabled: true,
            logs_max_size: 10 * 1024,
        }
    }

    pub fn name(self, name: impl Into<String>) -> ServiceEntryBuilder<String> {
        ServiceEntryBuilder {
            name: name.into(),
            enabled: self.enabled,
            logs_max_size: self.logs_max_size,
        }
    }
}
```

Notice that we now don't return `Self` from `name()`, but instead return a new
concrete implementation of `ServiceEntryBuilder` over `String`, which looks
like:

```rust
impl ServiceEntryBuilder<String> {
    pub fn build(self) -> ServiceEntry {
        ServiceEntry {
            name: self.name,
            enabled: self.enabled,
            logs_max_size: self.logs_max_size,
        }
    }

    pub fn name(mut self, name: impl Into<String>) -> Self {
        self.name = name.into();
        self
    }
}
```

In this new implementation we can now guarantee that `name` is set and is of
type `String`, so we got rid of the ugly `.unwrap()`. Awesome!

On the usage side everything still works as expected:

```rust
fn main() {
    let service = ServiceEntryBuilder::new()
        .enabled(false)
        .logs_max_size(256 * 1024)
        .name("my-service")
        .build();

    service.report();
}
```

Here is the output:

```sh
$ cargo run -q
Service details:
name: my-service
enabled: false
logs max size: 262144
```

However, let's see the cool part. If we omit name we now get a compiler error:

```rust
fn main() {
    let service = ServiceEntryBuilder::new()
        .enabled(false)
        .logs_max_size(256 * 1024)
        // .name("my-service")
        .build();

    service.report();
}
```

If we run `cargo run` again we get the following error:

```sh
$ cargo run -q
error[E0599]: no method named `build` found for struct `ServiceEntryBuilder<NoName>` in the current scope
  --> src/main.rs:76:10
   |
20 |   struct ServiceEntryBuilder<T> {
   |   ----------------------------- method `build` not found for this struct
...
72 |       let service = ServiceEntryBuilder::new()
   |                     --------------------------
   |                     |
   |  ___________________method `build` is available on `ServiceEntryBuilder<NoName>`
   | |
73 | |         .enabled(false)
   | |          -------------- method `build` is available on `ServiceEntryBuilder<NoName>`
74 | |         .logs_max_size(256 * 1024)
75 | |         // .name("my-service")
76 | |         .build();
   | |         -^^^^^ method not found in `ServiceEntryBuilder<NoName>`
   | |_________|
   |
   |
   = note: the method was found for
           - `ServiceEntryBuilder<String>`

For more information about this error, try `rustc --explain E0599`.
error: could not compile `builder-typestate` (bin "builder-typestate") due to 1 previous error
```

The Rust compiler has the best error messages, it even suggests that we can
find the `build` method in the implementation of the builder over `String`. The
reason is of course, because, indeed we don't have a build method on the
implementation over `NoName`, which is exactly what we want.
Isn't that cool?!

Furthermore we didn't lose the flexibility of our builder, we can still have
the chaining methods in any order we want and we can re-use them to our liking.

```rust
fn main() {
    let service = ServiceEntryBuilder::new()
        .enabled(false)
        .name("my-service")
        .logs_max_size(20 * 1024)
        .name("me-again")
        .enabled(true)
        .logs_max_size(10 * 1024)
        .build();

    service.report();
}
```

Works and outputs:

```sh
$ cargo run -q
Service details:
name: me-again
enabled: true
logs max size: 10240
```

### Conclusion

To wrap things up, the builder pattern is a real game changer when you're
dealing with complex object setups in Rust. It not only simplifies your code
but also keeps it clean and easy to manage. When you throw the typestate
pattern into the mix, things get even better. This combo enforces an order to
the building process, catching slip-ups during compile time instead of leaving
them for runtime surprises.

Happy coding!
