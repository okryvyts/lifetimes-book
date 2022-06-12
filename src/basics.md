# Basics

_It's recommended to follow the example by retyping it in your favorite editor_

Imagine we're writing a web crawler that collects posts from different blogs.
The crawler's output looks like that:

```rust
struct DiscoveredItem {
    blog_url: String,
    post_url: String,
}
```

Now we want to write a function operating on the web crawler's results
to iterate post URLs from a specific blog.


```rust
struct DiscoveredItem {
    blog_url: String,
    post_url: String,
}

fn post_urls_from_blog(
    items: &[DiscoveredItem],
    blog_url: &str,
) -> impl Iterator<Item = &str> {
    // Creating an iterator from the &[DiscoveredItem] slice
    items.iter().filter_map(move |item| {
        // Filtering items by blog_url
        if item.blog_url == blog_url {
            // Returning a post URL
            Some(item.post_url.as_str())
        } else {
            None
        }
    })
}
```

Our function doesn't compile, the compiler complains it needs some lifetime annotations.

```
error[E0106]: missing lifetime specifier
  --> src/main.rs:12:27
   |
10 |     items: &[DiscoveredItem],
   |            -----------------
11 |     blog_url: &str,
   |               ----
12 | ) -> impl Iterator<Item = &str> {
   |                           ^ expected named lifetime parameter
   |
   = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `items` or `blog_url`
help: consider introducing a named lifetime parameter
   |
9  ~ fn post_urls_from_blog<'a>(
10 ~     items: &'a [DiscoveredItem],
11 ~     blog_url: &'a str,
12 ~ ) -> impl Iterator<Item = &'a str> {
   |

For more information about this error, try `rustc --explain E0106`.
error: could not compile `playground` due to previous error
```

If we read the error carefully we can even find a suggestion how to fix our signature.

```
help: consider introducing a named lifetime parameter
   |
9  ~ fn post_urls_from_blog<'a>(
10 ~     items: &'a [DiscoveredItem],
11 ~     blog_url: &'a str,
12 ~ ) -> impl Iterator<Item = &'a str> {
   |
```

Let's use it!

```rust
struct DiscoveredItem {
    blog_url: String,
    post_url: String,
}

fn post_urls_from_blog<'a>(
    items: &'a [DiscoveredItem],
    blog_url: &'a str,
) -> impl Iterator<Item = &'a str> {
    // Creating an iterator from the &[DiscoveredItem] slice
    items.iter().filter_map(move |item| {
        // Filtering items by blog_url
        if item.blog_url == blog_url {
            // Returning a post URL
            Some(item.post_url.as_str())
        } else {
            None
        }
    })
}
```

Cool, now everything compiles. So that's it, we've won our fight with the borrow checker... right? Well, not quite.
The compiler actually tricked us. It provided a suggestion which is semantically incorrect and which made
our function over restrictive. That means the borrow checker will strike back soon and will make us scream out of pain.
Take a deep breath because in a moment...

## Compiler strikes back

Let's try to use our function in some non-trivial context.


```rust
#struct DiscoveredItem {
#    blog_url: String,
#    post_url: String,
#}
#
#fn post_urls_from_blog<'a>(
#    items: &'a [DiscoveredItem],
#    blog_url: &'a str,
#) -> impl Iterator<Item = &'a str> {
#    // Creating an iterator from the &[DiscoveredItem] slice
#    items.iter().filter_map(move |item| {
#        // Filtering items by blog_url
#        if item.blog_url == blog_url {
#            // Returning a post URL
#            Some(item.post_url.as_str())
#        } else {
#            None
#        }
#    })
#}
fn main() {
    // Assume the crawler returned the following results
    let crawler_results = &[
        DiscoveredItem {
            blog_url: "https://blogs.com/".to_owned().to_owned(),
            post_url: "https://blogs.com/cooking/fried_eggs".to_owned(),
        },
        DiscoveredItem {
            blog_url: "https://blogs.com/".to_owned(),
            post_url: "https://blogs.com/travelling/death_mountain".to_owned(),
        },
        DiscoveredItem {
            blog_url: "https://successfulsam.xyz/".to_owned(),
            post_url: "https://successfulsam.xyz/keys_to_success/Just_do_this_one_thing_every_day".to_owned(),
        },
    ];

    // Reading the blog URL we're interested in from somewhere
    let blog_url = get_blog_url();

    // Collecting post URLs from this blog using our function
    let post_urls: Vec<_> = post_urls_from_blog(crawler_results, &blog_url).collect();

    // Spawning a thread to do some further blog processing
    let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));

    // Processing posts in parallel
    for url in post_urls {
        process_post(url);
    }

    handle.join().expect("Everything will be fine");
}

// Returns a predefined value
fn get_blog_url() -> String {
    "https://blogs.com/".to_owned()
}

// Just prints URL out
fn process_post(url: &str) {
    println!("{}", url);
}

// Actually does nothing
fn calculate_blog_stats(_blog_url: String) {}
```

This code doesn't compile. What's worse is that the compiler error is just absurd:

```
   Compiling playground v0.0.1 (/playground)
error[E0505]: cannot move out of `blog_url` because it is borrowed
  --> src/main.rs:45:37
   |
42 |     let post_urls: Vec<_> = post_urls_from_blog(crawler_results, &blog_url).collect();
   |                                                                  --------- borrow of `blog_url` occurs here
...
45 |     let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
   |                                     ^^^^^^^                      -------- move occurs due to use in closure
   |                                     |
   |                                     move out of `blog_url` occurs here
...
48 |     for url in post_urls {
   |                --------- borrow later used here

For more information about this error, try `rustc --explain E0505`.
error: could not compile `playground` due to previous error
```

How on earth `blog_url` that was taken from some kind of a user input is
related to `post_urls` returned by the crawler?
The most annoying thing is that the code in the `main` function is actually perfectly fine and should compile,
but as you may have already guessed it doesn't because of our malformed function signature.
What's going on here? And what's the error communicating to us?
To answer these questions, we need to understand the way the borrow checker works.
