---
layout: post
title: "Pattern matching in Rust"
description: "In this brief post we explore pattern matching in Rust with the aim to impress you with its coolness. Also a showcase of elegance."
date: 2023-11-18
tags: [programming, rust]
comments: false
share: true
---

Greetings fellow Rustaceans 🦀! 

In this short post, I'll try to present a series of succint yet hopefully comprehensive examples of pattern matching in Rust.

Rust has many cool features and pattern matching is one of them.

I would like to keep this short, so let's dive right into it.

### Example 1: Basics of the `match` keyword

Our exploration of pattern matching starts with the `match` keyword. It is a versatile conductor orchestrating decisions based on data. Whether dealing with numbers, strings, or custom structs, the match construct ensures a fluid and stylistically aligned code execution. 

Here is a very basic and contrived example, to start with:


```rust
fn main() {
    let n = 7;

    match n {
        1 => println!("Too low."), // not matched
        7 => println!("Lucky number 7!"), // **matched!**
        42 => println!("The answer to life, the universe, and everything..."), // not matched
        _ => println!("A numerical insignificance."), // not matched
    }
}
// prints: Lucky number 7!
```

### Example 2: Structs

Here is how we can do pattern matching on structs:

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let my_point = Point { x: 42, y: 24 };

    match my_point {
        Point { x, y } => {
            println!("X-coordinate: {}, Y-coordinate: {}", x, y);
        }
    }
}
// prints: X-coordinate: 42, Y-coordinate: 24
```

### Example 3: Enums

In pattern matching, enums shine by providing a structured way to handle the different cases. The example showcases the handling of different variants of the `Message` enum:

```rust
enum Message {
    Greeting(String),
    Farewell,
    Custom(String, i32),
}

fn main() {
    let my_message = Message::Custom(String::from("Salutations"), 42);

    match my_message {
        Message::Greeting(msg) => {
            println!("Received a salutation: {}", msg);
        }
        Message::Farewell => {
            println!("Bidding adieu!");
        }
        Message::Custom(text, number) => {
            println!("Custom message: {} - {}", text, number);
        }
    }
}
// prints: Custom message: Salutations - 42
```

### Example 4: The `@` operator

The `@` opeartor creates a binding to a variable.

```rust
fn main() {
    let point = (3, 7);

    match point {
        (x_within_range @ 1..=5, y_within_range @ 1..=10) => {
            println!(
                "Point within specified bounds: ({}, {})",
                x_within_range, y_within_range
            );
        }
        (x, y) => {
            println!("Point outside specified bounds: ({}, {})", x, y);
        }
    }
}
// prints: Point within specified bounds: (3, 7)
```

### Example 5: More of the `@` operator

Here is another example where, when matched, we bind the name "Alice" to the variable `special_name`:

```rust
fn main() {
    let name = "Alice";

    match name {
        special_name @ "Alice" => {
            println!(
                "Special greeting for Alice: Hello, {}!",
                special_name.to_uppercase()
            );
        }
        x => {
            println!("Generic greeting: Hi, {}!", x);
        }
    }
}
// prints: Special greeting for Alice: Hello, ALICE!
```

### Example 6: Mixing things

We can mix match with `|` and `if` in order to create much more expressive and flexible matches, where we chain patterns together , as shown here:

```rust
fn main() {
    let my_favorite_fruit = ("banana", 3);

    match my_favorite_fruit {
        ("banana", quantity) | ("apple", quantity) if quantity > 2 => {
            println!("An abundance of fruit!");
        }
        ("orange", _) => {
            println!("Evoking citrusy vibes!");
        }
        _ => {
            println!("Merely a snack.");
        }
    }
}
// prints: An abundance of fruit!
```

### Example 7: Ranges and guards

We already saw guards and conditional expressions within patterns, but here is a bit more involved example to illustrate the point:

```rust
fn main() {
    let number = 42;

    match number {
        n if n < 0 => println!("Manifesting negative connotations!"),
        n if (0..=10).contains(&n) => println!("Diminutive yet potent!"),
        n if n > 10 && n % 2 == 0 => println!("Balancing the odds!"),
        _ => println!("A numerical insignificance."),
    }
}
// prints: Balncing the odds!
```

### Example 8: The `@` operator once again

Here we revisit the `@` to ephasize its utility in capturing specific values within patterns providing a refined and precise approach to matching:

```rust
struct Person {
    name: String,
    age: u32,
}

fn main() {
    let my_person = Person {
        name: String::from("Alice"),
        age: 30,
    };

    match my_person {
        Person {
            name,
            age: age_within_range @ 20..=35,
        } => {
            println!(
                "{} falls within the age range of 20-35 ({} years old).",
                name, age_within_range
            );
        }
        Person { name, age } => {
            println!(
                "{} does not fall within the specified age range ({} years old).",
                name, age
            );
        }
    }
}
// prints: Alice falls within the age range of 20-35 (30 years old).
```

Hopefully someone will find this post useful, but at the very least I know that future Self (see what I did there?) will use it as a reference. Happy coding!

