---
layout: post
title: "Dynamic vs Static Dispatch in Rust"
description: "Dynamic vs Static Dispatch in Rust - A brief explanation and comparison"
date: 2023-08-29
tags: [programming, rust]
comments: false
share: true
---

This is going to be a brief explanation and comparison about dynamic and static dispatch in Rust.

Dispatch refers to the process of determining which function or method to call at runtime when the code involves polymorphism. In Rust, this choice can impact both the design and performance of your code.

Rust supports dynamic dispatch by utilizing traits.

Before we proceed, let's present some code which we will use as a base for the rest of this post.

We'll start with the following trait:

```rust
trait Renderable {
    fn render(&self);
}
```

We will proceed by implementing this trait:

```rust
struct HTML {
    content: String,
}

impl Renderable for HTML {
    fn render(&self) {
        println!("Rendering as HTML:\n{}", self.content);
    }
}

struct Markdown {
    content: String,
}

impl Renderable for Markdown {
    fn render(&self) {
        println!("Rendering as Markdown:\n{}", self.content);
    }
}
```


## Static Dispatch

Static dispatch (which happens through [monomorphization](https://en.wikipedia.org/wiki/Monomorphization)), occurs when the function or method to be called is determined at compile time. In Rust, this happens when the type information is known during compilation. It's efficient since the compiler can optimize the generated code for specific types. It can be done with trait bounds:

```rust
fn static_render<T: Renderable>(x: &T) {
    // Do some work then render...
    x.render();
}

fn main() {
    let html = HTML {
        content: String::from("<h1>Hello, world!</h1>"),
    };

    static_render(&html);

    let markdown = Markdown {
        content: String::from("# Hello, world!"),
    };

    static_render(&markdown);
}
```

In the `static_render()` function, the type of the `x` parameter is known at compile time. The compiler generates specialized code for each type that implements the `Renderable` trait. A mental model that might help here, is to think Rust generates something like:

```rust
fn static_render_for_html(x: &HTML) {
    // we know it's HTML
    x.render();
}

fn static_render_for_markdown(x: &Markdown) {
    // we know it's Markdown
    x.render();
}
```

This results in efficient code execution and eliminates the runtime overhead associated with dynamic dispatch.

## Dynamic Dispatch

Dynamic dispatch, on the other hand, happens at runtime and is based on a feature called "trait objects". Trait objects allow you to write more flexible code that can work with multiple types implementing the same trait. However, the come with a performance cost due to the runtime overhead of dynamic method resolution.

```rust
fn dynamic_render(x: &dyn Renderable) {
    // Do some work then render...
    x.render();
}

fn main() {
    let html = HTML {
        content: String::from("<h1>Hello, world!</h1>"),
    };

    dynamic_render(&html);

    let markdown = Markdown {
        content: String::from("# Hello, world!"),
    };

    dynamic_render(&markdown);
}
```

In the `dynamic_render` function, the `x` parameter is of type `&dyn Renderable` which is a trait object. The method calls are resolved at runtime using a virtual function table ([vtable](https://en.wikipedia.org/wiki/Virtual_method_table)).

In scenarios where you don't know the exact types at compile time or want to work with a variety of types through a common interface, dynamic dispatch using trait objects becomes necessary. For instance, if you're dealing with different types that implement a trait and are in a collection, dynamic dispatch is needed.

Here is an example, using the types from earlier:

```rust
fn main() {
    let html_renderer = Box::new(HTML {
        content: String::from("<h1>Hello</h1>"),
    });

    let markdown_renderer = Box::new(Markdown {
        content: String::from("# Hi"),
    });

    let renderers: Vec<Box<dyn Renderable>> = vec![html_renderer, markdown_renderer];

    for r in renderers {
        r.render();
    }
}
```

When we're iterating in the `for` and calling `r.renderer()` the compiler can't know whether we're calling `HTML::render()` or `Markdown::render()`, they both have different addresses and bodies. `HTML::render()` and `Markdown::render()` are therefore totally different functions, but you can store an `HTML` and a `Markdown` together in the same collection. A vtable (and dynamic dispatch) is necessary in this case in order to determine which method to call based on wheter it's an `HTML` or a `Markdown`.
