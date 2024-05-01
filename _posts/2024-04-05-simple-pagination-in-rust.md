---
layout: post
title: "Simple pagination in Rust"
description: "In this post we will briefely desribe a simple way to do pagination in Rust, with a bonus wiring for Axum and Askama"
date: 2024-04-05
tags: [programming, rust, askama, axum]
comments: false
share: true
---

Greetings fellow Rustaceans 🦀! 

Today, I'll be diving into how to build a handy pagination system in Rust - a
feature that's essential for any application with a lot of data to display,
whether it's blog posts, product listings or anything in-between. I'll try to
break down step-by-step, keeping things simple and straightforward.

### Pagination Item Enum

First thing that needs to be determined is what types of items our pagination
system will manage. Here is a `PaginatorItem` enum that describes the bare
minimum.

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum PaginatorItem {
    Number { is_current: bool, value: usize },
    Ellipsis,
}
```

Don't mind the `Clone` and `PartialEq` for the moment, but we will need those a
bit later (`PartialEq` will be needed when we write unit tests for the
paginator outcome and we need to compare the expected results with the actual
results).
Here, `Number` carries a page number with a flag indicating if it's the current
page. `Ellipsis` is a special type indicating more pages are available than
currently shown. This helps in not overloading users with too many page numbers
at once.
Please note that you might to include more types, for example `Next`,
`Previous`, `First`, `Last`, but for the sake of simplicity I wanted to keep
just the bare minimum.

### Structuring our Paginator

Next up, we define a `Paginator` struct to keep tabs on our total number of
pages and the current page number:

```rust
#[derive(Debug)]
pub struct Paginator {
    total_pages: usize,
    page: usize,
}
```

Simple, right? This struct will help us navigate through our pages effectively.

### Safe construction with try_new

To get things rolling, we'll need a way to safely create an instance of our
`Paginator`. This is where `try_new` comes into play, performing some essential
checks:

```rust
impl Paginator {
    pub fn try_new(items_total_count: usize, per_page: usize, page: usize) -> Result<Self, String> {
        if page == 0 {
            return Err("Page number must be greater than 0".to_string());
        }

        if per_page == 0 {
            return Err("Items per page must be greater than 0".to_string());
        }

        let total_pages = (items_total_count + per_page - 1) / per_page;

        if page > total_pages && total_pages > 0 {
            return Err(format!("The page {} is greater than the total number of pages", page));
        }

        Ok(Self { total_pages, page })
    }
}
```

It's checking for things like ensuring the page number starts at 1 and that
we're not trying to paginate zero items per page - common sense stuff, really.
Also note that in order to keep things simple for this article the error type
is just a `String`, but you can see how this could be changed to something more
specialized.

### Determining the page range

When it comes to deciding which page numbers to show, we don't want to clutter
the interface. The `get_start_and_end_range` method helps us figure out a neat
range of pages to display:

```rust
impl Paginator {
    // ... rest of impl
    pub fn get_start_and_end_range(
        &self,
        total_pages: usize,
        page_list_size: usize,
    ) -> (usize, usize) {
        let mid = page_list_size / 2;

        let mut start = usize::max(1, self.page.saturating_sub(mid));
        let mut end = usize::min(total_pages, self.page + mid);

        if end - start + 1 < page_list_size {
            if start == 1 {
                end = usize::min(start + page_list_size - 1, total_pages);
            } else if end == total_pages {
                start = usize::max(end - page_list_size + 1, 1);
            }
        }

        (start, end)
    }
}
```

This function keeps our pagination bar neat, showing just enough pages around
the current page to make navigation easy, without overwhelming the users.
Here's a deeper dive into how it works:

- __Balancing the Range__: The method calculates the middle of the window
(`mid`) and uses it to adjust the start and end points. If the calculated range
of pages is less than `page_list_size`, it extends either the start or end to
compensate.
- __Handling Extremes__: Special care is taken if the calculated start is at
the very beginning or the end is at the last page, ensuring that the pagination
display is always logical and user-friendly.

### Bringing it all together: The paginate method

Finally, we put all the pieces together in the `paginate` method. This is where
we build the actual list of pagination items to display, based on the current
state:

```rust
/// Default number of pages to display in the pagination before showing ellipsis
const DEFAULT_PAGE_LIST_SIZE: usize = 3;

impl Paginator {
    // ... rest of impl
    pub fn paginate(&mut self) -> Vec<PaginatorItem> {
        if self.total_pages < 1 {
            // If there's only one page, there's no need to generate pagination.
            return Vec::new();
        }

        let mut items: Vec<PaginatorItem> = Vec::new();
        if self.total_pages <= DEFAULT_PAGE_LIST_SIZE + 2 {
            // If total pages are within the display limit (+2 because of the
            // ellipsis and last page), show all without ellipsis
            for value in 1..=self.total_pages {
                items.push(PaginatorItem::Number {
                    is_current: value == self.page,
                    value,
                });
            }

            return items;
        }

        let (start, end) = self.get_start_and_end_range(self.total_pages, DEFAULT_PAGE_LIST_SIZE);

        // Paginate start
        if start > 1 {
            items.push(PaginatorItem::Number {
                is_current: false,
                value: 1,
            });
            if start > 2 {
                // Show ellipsis if there's a gap
                items.push(PaginatorItem::Ellipsis);
            }
        }

        // Paginate middle
        for value in start..=end {
            items.push(PaginatorItem::Number {
                is_current: value == self.page,
                value,
            });
        }

        // Paginate end
        if end < self.total_pages {
            if end < self.total_pages - 1 {
                // Show ellipsis if there's a gap
                items.push(PaginatorItem::Ellipsis);
            }

            items.push(PaginatorItem::Number {
                is_current: false,
                value: self.total_pages,
            });
        }

        items
    }
}
```

As you can see this method is pretty straightforward. It dynamically builds a
user-friendly pagination display:

- __Empty__: If the total number of pages is zero we return early since there
is nothing to paginate.
- __Small set of pages__: If the total number of pages is small, every page
number is displayed. -__Larger sets__: For larger total number of pages, it
ensures that navigation is simple by showing the first page, an ellipsis if
needed, a central block of pages around the current page, possibly another
ellipsis, and the last page.

### Testing

Now, normally in a blog post such as this one, unit tests are omitted for
brevity, but I decided to include them. These are tests for all possible
scenarios. I find them extremely helpful, maybe you would too.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // Helper function to create a Paginator and run pagination
    fn setup_and_paginate(
        total_count: usize,
        per_page: usize,
        current_page: usize,
    ) -> Vec<PaginatorItem> {
        let mut paginator = Paginator::try_new(total_count, per_page, current_page)
            .expect("Failed to create paginator");
        paginator.paginate()
    }

    mod paginate {

        use super::*;

        #[test]
        fn first_page() {
            let items = setup_and_paginate(105, 10, 1);
            let expected_items = vec![
                PaginatorItem::Number {
                    is_current: true,
                    value: 1,
                },
                PaginatorItem::Number {
                    is_current: false,
                    value: 2,
                },
                PaginatorItem::Number {
                    is_current: false,
                    value: 3,
                },
                PaginatorItem::Ellipsis,
                PaginatorItem::Number {
                    is_current: false,
                    value: 11,
                },
            ];

            // Result should be: [1] 2 3 ... 11
            assert_eq!(items, expected_items);
        }

        #[test]
        fn second_page() {
            let items = setup_and_paginate(105, 10, 2);
            let expected_items = vec![
                PaginatorItem::Number {
                    is_current: false,
                    value: 1,
                },
                PaginatorItem::Number {
                    is_current: true,
                    value: 2,
                },
                PaginatorItem::Number {
                    is_current: false,
                    value: 3,
                },
                PaginatorItem::Ellipsis,
                PaginatorItem::Number {
                    is_current: false,
                    value: 11,
                },
            ];

            // Result should be: 1 [2] 3 ... 11
            assert_eq!(items, expected_items);
        }

        #[test]
        fn page_right_before_start_ellipsis_threshold() {
            let items = setup_and_paginate(105, 10, 3);
            let expected_items = vec![
                PaginatorItem::Number {
                    is_current: false,
                    value: 1,
                },
                PaginatorItem::Number {
                    is_current: false,
                    value: 2,
                },
                PaginatorItem::Number {
                    is_current: true,
                    value: 3,
                },
                PaginatorItem::Number {
                    is_current: false,
                    value: 4,
                },
                PaginatorItem::Ellipsis,
                PaginatorItem::Number {
                    is_current: false,
                    value: 11,
                },
            ];

            // Result should be: 1 2 [3] 4 ... 11
            assert_eq!(items, expected_items);
        }

        #[test]
        fn page_right_after_start_ellipsis_threshold() {
            let items = setup_and_paginate(105, 10, 4);
            let expected_items = vec![
                PaginatorItem::Number {
                    is_current: false,
                    value: 1,
                },
                PaginatorItem::Ellipsis,
                PaginatorItem::Number {
                    is_current: false,
                    value: 3,
                },
                PaginatorItem::Number {
                    is_current: true,
                    value: 4,
                },
                PaginatorItem::Number {
                    is_current: false,
                    value: 5,
                },
                PaginatorItem::Ellipsis,
                PaginatorItem::Number {
                    is_current: false,
                    value: 11,
                },
            ];

            // Result should be: 1 ... 3 [4] 5 ... 11
            assert_eq!(items, expected_items);
        }

        #[test]
        fn pagination_middle_page() {
            let items = setup_and_paginate(105, 10, 5);
            let expected_items = vec![
                PaginatorItem::Number {
                    is_current: false,
                    value: 1,
                },
                PaginatorItem::Ellipsis,
                PaginatorItem::Number {
                    is_current: false,
                    value: 4,
                },
                PaginatorItem::Number {
                    is_current: true,
                    value: 5,
                },
                PaginatorItem::Number {
                    is_current: false,
                    value: 6,
                },
                PaginatorItem::Ellipsis,
                PaginatorItem::Number {
                    is_current: false,
                    value: 11,
                },
            ];

            // Result should be: 1 ... 4 [5] 6 ... 11
            assert_eq!(items, expected_items);
        }

        #[test]
        fn page_right_before_end_ellipsis_threshold() {
            let items = setup_and_paginate(105, 10, 8);
            let expected_items = vec![
                PaginatorItem::Number {
                    is_current: false,
                    value: 1,
                },
                PaginatorItem::Ellipsis,
                PaginatorItem::Number {
                    is_current: false,
                    value: 7,
                },
                PaginatorItem::Number {
                    is_current: true,
                    value: 8,
                },
                PaginatorItem::Number {
                    is_current: false,
                    value: 9,
                },
                PaginatorItem::Ellipsis,
                PaginatorItem::Number {
                    is_current: false,
                    value: 11,
                },
            ];

            // Result should be: 1 ... 7 [8] 9 ... 11
            assert_eq!(items, expected_items);
        }

        #[test]
        fn page_right_after_end_ellipsis_threshold() {
            let items = setup_and_paginate(105, 10, 9);
            let expected_items = vec![
                PaginatorItem::Number {
                    is_current: false,
                    value: 1,
                },
                PaginatorItem::Ellipsis,
                PaginatorItem::Number {
                    is_current: false,
                    value: 8,
                },
                PaginatorItem::Number {
                    is_current: true,
                    value: 9,
                },
                PaginatorItem::Number {
                    is_current: false,
                    value: 10,
                },
                PaginatorItem::Number {
                    is_current: false,
                    value: 11,
                },
            ];

            // Result should be: 1 ... 8 [9] 10 11
            assert_eq!(items, expected_items);
        }

        #[test]
        fn last_page() {
            let items = setup_and_paginate(105, 10, 11);
            let expected_items = vec![
                PaginatorItem::Number {
                    is_current: false,
                    value: 1,
                },
                PaginatorItem::Ellipsis,
                PaginatorItem::Number {
                    is_current: false,
                    value: 9,
                },
                PaginatorItem::Number {
                    is_current: false,
                    value: 10,
                },
                PaginatorItem::Number {
                    is_current: true,
                    value: 11,
                },
            ];

            // Result should be: 1 ... 9 10 [11]
            assert_eq!(items, expected_items);
        }

        #[test]
        fn no_of_pages_below_all_ellipsis_threshold() {
            let items = setup_and_paginate(50, 10, 4);
            let expected_items = vec![
                PaginatorItem::Number {
                    is_current: false,
                    value: 1,
                },
                PaginatorItem::Number {
                    is_current: false,
                    value: 2,
                },
                PaginatorItem::Number {
                    is_current: false,
                    value: 3,
                },
                PaginatorItem::Number {
                    is_current: true,
                    value: 4,
                },
                PaginatorItem::Number {
                    is_current: false,
                    value: 5,
                },
            ];

            // Result should be: 1 2 3 [4] 5
            assert_eq!(items, expected_items);
        }

        #[test]
        fn large_number_of_pages() {
            let items = setup_and_paginate(1577, 10, 1);
            let expected_items = vec![
                PaginatorItem::Number {
                    is_current: true,
                    value: 1,
                },
                PaginatorItem::Number {
                    is_current: false,
                    value: 2,
                },
                PaginatorItem::Number {
                    is_current: false,
                    value: 3,
                },
                PaginatorItem::Ellipsis,
                PaginatorItem::Number {
                    is_current: false,
                    value: 158,
                },
            ];

            // Result should be: [1] 2 3 ... 158
            assert_eq!(items, expected_items);
        }

        #[test]
        fn page_is_greater_than_last_page() {
            // Page is greater than the total number of pages
            // We expect an error to be returned
            let err = Paginator::try_new(100, 10, 11).unwrap_err();

            assert_eq!(err, "The page 11 is greater than the total number pages");
        }

        #[test]
        fn empty() {
            let items = setup_and_paginate(0, 10, 1);

            // No pagination needed for an empty list
            assert!(items.is_empty());
        }

        #[test]
        fn single_page() {
            let items = setup_and_paginate(5, 10, 1);

            let expected_items = vec![PaginatorItem::Number {
                is_current: true,
                value: 1,
            }];

            // Result should be: [1]
            assert_eq!(items, expected_items);
        }
    }
}
```

### Bonus: wiring it up with Askama

In the project that I work on, we use [axum](https://github.com/tokio-rs/axum)
with [askama](https://github.com/djc/askama). Please, note that I won't go into
details of how axum or askama work in this blog post, since it's outside of the
scope. I use this paginator in askama, this is how it is wired up. 

First we have a paginate the data in the backend in an axum handler and pass it
down to an askama template. It would look something like (contrived example):

```rust
#[derive(Template)]
#[template(path = "views/posts.html")]
pub struct PostsView {
    posts: Page<Post>,
    paginator_items: Vec<PaginatorItem>,

    // We need this for the paginator links
    current_relative_url_path: String,
}
```

This is the axum handler for posts:

```rust
/// Query parameters for the paginator
#[derive(Deserialize)]
pub struct PaginatorQuery {
    pub page: Option<usize>,
}

// Number of items per page, this could also be dynamic as a query param for example
const PER_PAGE: usize = 10;

pub async fn show_posts(
    Query(query): Query<PaginatorQuery>,
) -> Result<impl IntoResponse, StatusError> {
    let page = query.page.unwrap_or(1);

    let posts: Page<Post> = get_posts_paginated(page, PER_PAGE).await?;

    let mut paginator = Paginator::try_new(
        posts.total_count.unwrap_or(0),
        PER_PAGE,
        page,
    )
    .map_err(|error| {
        error!(?error, "Failed to create paginator");
        StatusError::internal_error()
    })?;

    let paginator_items = paginator.paginate();

    Ok(HtmlTemplate(PostsView {
        posts,
        paginator_items,
        current_relative_url_path: "/posts".to_string(),
    })
    .into_response())
}
```

Then in our `posts` askama template:

```html
{% raw %}
{% extends "layout.html" %}

{% block title %}
  Posts
{% endblock %}

{% block content %}
  <h2>Posts</h2>
  {% if posts.items.is_empty() %}
    <div>
        There are currently no posts to show at this time.
    </div>
  {% else %}
    <ul>
      {% for post in posts.items %}
        <li>
            {{ post.title }}
        </li>
      {% endfor %}
    </ul>

    {% include "partials/paginator.html" %}
  {% endif %}
{% endblock %}
{% endraw %}
```

As you can see at the bottoom we have a `paginator` partial template that is
re-used on every page that needs pagination. Here is the `paginator.html`
askama partial template:

```html
{% raw %}
{% if !paginator_items.is_empty() %}
  <div class="flex flex-wrap text-sm md:text-base justify-center items-center space-x-4 mt-6">
    {% for item in paginator_items %}
      {% match item %}
        {% when PaginatorItem::Number with {is_current, value } %}
          {% if is_current %}
            <span class="border px-4 py-2">{{ value }}</span>
          {% else %}
            <a href="{{current_relative_url_path}}?page={{ value }}" class="px-4 py-2">{{ value }}</a>
          {% endif %}
        {% when PaginatorItem::Ellipsis %}
          <span class="px-4 py-2">...</span>
      {% endmatch %}
    {% endfor %}
  </div>
{% endif %}
{% endraw %}
```

As you can imagine the wiring was pretty simple, we just handle the case for
each of the different variants of `PaginatorItem` and we add some styling (in
this case `tailwindcss`).
    

### Conclusion

And there you have it - a neat little pagination system built from scratch in
Rust! As a bonus I tried to show you how to wire this up in a typical Rust web
app. Whether you're working on a new web app or just looking to understand Rust
a bit better, I hope this guide was useful and helps you in your journey.

Happy coding!
