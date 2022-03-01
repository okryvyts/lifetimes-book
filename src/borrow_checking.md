# Borrow checking

The most important thing to understand about the borrow checker is it
analyzes each function completely independently from other functions.
That means when we encounter a call to our `post_urls_from_blog` the
borrow checker doesn't look inside it to validate the usage of references.
All it does is it reads the function signature and evaluates its lifetimes.
But what does it mean to evaluate a lifetime? Let's go back to our example
and figure this out.

As a borrow checker we're analyzing our `main` function and encountering a
line of code with another function invocation.

```rust,noplayground
let post_urls: Vec<_> = post_urls_from_blog(crawler_results, &blog_url).collect();
```

First of all, we need to read the function signature:

```rust,noplayground
fn post_urls_from_blog<'a>(
    items: &'a [DiscoveredItem], 
    blog_url: &'a str,
) -> impl Iterator<Item = &'a str> {
    // We're looking at the function from the borrow checker's perspective.
    // It doesn't see the impl :(
}
```

See this `'a` we defined here? This is a generic lifetime. Sort of a placeholder.
We need to come up with a concrete value for it at every place we call the function.
To calculate it we need to adhere the following conditions:

1. The lifetime value must be minimal.
1. References with this resulting lifetime value must stay valid for the whole lifetime value(no dangling pointers!)

Ok, but that still sounds vague. What exactly is a lifetime value? Well, it's nothing more than a continuous[^continuous] region of code. 
Like from line X to line Y. The single line with the function invocation above is a perfect region of code. That's some another perfect region of code:

```rust,noplayground
/---region
|// Reading the blog URL we're interested in from somewhere
|let blog_url = get_blog_url(); 
|
|// Collecting post URLs from this blog using our function
|let post_urls: Vec<_> = post_urls_from_blog(crawler_results, &blog_url).collect();
|
|// Spawning a thread to do some further blog processing
|let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
-
```

Region boundaries basically define where the refences can be used and the duration of the borrow.
__All__ references belonging to the region must be safe to use at __any__ line within the region.
All regions are function scoped. There are no "cross-function" regions. When we encounter
a function call in a function we currently analyze we just calculate a region for it
in the calling function based on the callee signature.

Now when we posses this secret knowledge we can formulate our task more precisely:
_At the function invocation point we need to infer some minimal regions of code that will "hold" our references 
with the guarantee that these references are safe to use at any line within the region they belong to._

## Inferring regions

Let's look at the `post_urls_from_blog` signature once again:

```rust,noplayground
fn post_urls_from_blog<'a>(
    items: &'a [DiscoveredItem], 
    blog_url: &'a str,
) -> impl Iterator<Item = &'a str> {
    // We're looking at the function from the borrow checker's perspective.
    // It doesn't see the impl :(
}
```

We see only one lifetime paramater which means we need to infer only one region for this function(Yes, we're inferring
regions for the whole function, not for each of its arguments).
This region must hold `items`, `blog_url`, `Item` references and... an iterator. The complete function
signature actually looks like that:

```rust,noplayground
fn post_urls_from_blog<'a>(
    items: &'a [DiscoveredItem], 
    blog_url: &'a str,
) -> impl Iterator<Item = &'a str> + 'a {
    // We're looking at the function from the borrow checker's perspective.
    // It doesn't see the impl :(
}
```

The compiler elided the last `'a` according to lifetime elision rules, so it was hidden from us. 
Now we have all information to perform the evaluation. I will evaluate the signature in a backwards order to quickly show the region expansion, 
but in general it's more natural to start from the input arguments.

How wide the region should be? As wide as all references it holds must stay valid. How long
the references must stay valid? As long as they're used. So, basically, a size of a region is determined
by the last reference usage this region holds. Let's apply this rule in practice. 
Our function returns an iterator `impl Iterator<Item = &'a str> + 'a`. How is it used?

```rust,noplayground
let post_urls: Vec<_> = post_urls_from_blog(crawler_results, &blog_url).collect();
```

Well, we just collect it immediately into a vector therefore, our region with respect of iterator
is a single line of code(the iterator is consumed and can't be used anywhere else):

```rust,noplayground
fn main() {
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

    let blog_url = get_blog_url(); 

/---post_urls_from_blog 'a region
|   let post_urls: Vec<_> = post_urls_from_blog(crawler_results, &blog_url).collect();
-
    let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));

    for url in post_urls {
        process_post(url);
    }

    handle.join().expect("Everything will be fine");
}
```

The consumed iterator yielded references `Item=&'a str` that also belong to our region.
We store them in the `post_urls` vector. Now we need to find the last usage of those references.
It's here:

```rust,noplayground
    for url in post_urls {
        process_post(url);
    }
```

So `post_urls` references must be valid at least till the end of this loop. Expanding the region accordingly:

```rust,noplayground
fn main() {
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

    let blog_url = get_blog_url(); 

/---post_urls_from_blog 'a region
|   let post_urls: Vec<_> = post_urls_from_blog(crawler_results, &blog_url).collect();
|
|   let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
|
|   for url in post_urls {
|       process_post(url);
|   }
-
    handle.join().expect("Everything will be fine");
}
```

As for input arguments, usually they don't affect the region expansion because
they must be valid only for the duration of a function call, but later we will study some cases when they do.
Let's evaluate `items: &'a [DiscoveredItem]` and `blog_url: &'a str` together. They're just
regular input references without any quirks, so they must be valid only at the line with the function invocation.
If we had started our analysis from input arguments, our region would look like that:

```rust,noplayground
fn main() {
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

    let blog_url = get_blog_url(); 

/---post_urls_from_blog 'a region
|   let post_urls: Vec<_> = post_urls_from_blog(crawler_results, &blog_url).collect();
-
    let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));

    for url in post_urls {
        process_post(url);
    }

    handle.join().expect("Everything will be fine");
}
```

But we've already analyzed the outputs and know that our region must be wider, hence the resulting
`'a` region of the `post_urls_from_blog` function looks like that:

```rust,noplayground
fn main() {
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

    let blog_url = get_blog_url(); 

/---post_urls_from_blog 'a region
|   let post_urls: Vec<_> = post_urls_from_blog(crawler_results, &blog_url).collect();
|
|   let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
|
|   for url in post_urls {
|       process_post(url);
|   }
-
    handle.join().expect("Everything will be fine");
}
```

The region holds: a copy of the `crawler_results` reference, a reference to `blog_url`, a vector of `post_urls` references,
and the consumed iterator(yes, it's consumed at the first line of our region, but it still belongs to it).
Note that we didn't analyze any relationships between references. We don't understand
how inputs and outputs are connected and where the references point to. All we did is
we inferred a region for them where derefencing __any__ of those references must
be safe. 

We're done with our function. Regions for regular variables can be trivially inferred by following Rust scoping
rules. For example, this is the region for the `handle` variable:

```rust,noplayground
fn main() {
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

    let blog_url = get_blog_url(); 

    let post_urls: Vec<_> = post_urls_from_blog(crawler_results, &blog_url).collect();
/---handle region
|   let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
|
|   for url in post_urls {
|       process_post(url);
|   }
|
|   handle.join().expect("Everything will be fine");
-
}
```

Try to infer regions for the remaining variables yourself.

## Validating regions
It's time to ensure the safety. After we inferred all regions in the
analyzed function we need to explore relationships between __the regions__(not variables) looking
for potential conflicts. Let's start at the point where the `crawler_results` reference is copied to be passed as an argument to the `post_urls_from_blog`
function. Can we create this copy? 

```rust,noplayground
fn main() {
/---crawler results region
|   let crawler_results = &[
|       DiscoveredItem {
|           blog_url: "https://blogs.com/".to_owned().to_owned(),
|           post_url: "https://blogs.com/cooking/fried_eggs".to_owned(),
|       },
|       DiscoveredItem {
|           blog_url: "https://blogs.com/".to_owned(),
|           post_url: "https://blogs.com/travelling/death_mountain".to_owned(),
|       },
|       DiscoveredItem {
|           blog_url: "https://successfulsam.xyz/".to_owned(),
|           post_url: "https://successfulsam.xyz/keys_to_success/Just_do_this_one_thing_every_day".to_owned(),
|       },
|   ];
|
|   let blog_url = get_blog_url(); 
|
|/--post_urls_from_blog 'a region
||  let post_urls: Vec<_> = post_urls_from_blog(crawler_results, &blog_url).collect();
||
||  let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
||
||  for url in post_urls {
||      process_post(url);
||  }
|-
|   handle.join().expect("Everything will be fine");
-
}
```

`crawler_resutls` region fills the whole `main` body clearly outliving our `post_urls_from_blog 'a` region meaning
we can dereference the copy of `crawler_results` reference at any line within the `post_urls_from_blog 'a` region
(note that we use `crawler_results` borrow only at the line with the function call, but it lasts till the end of the `post_urls_from_blog 'a` region anyway).

Then we're taking a reference to `blog_url`. Can we do that?

```rust,noplayground
fn main() {
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
/---blog_url region 
|   let blog_url = get_blog_url(); 
|
|/--post_urls_from_blog 'a region
||  let post_urls: Vec<_> = post_urls_from_blog(crawler_results, &blog_url).collect();
||
||  let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
-|
 |  for url in post_urls {
 |      process_post(url);
 |  }
 -
    handle.join().expect("Everything will be fine");
 
}
```

No, we can't. We must be able to derefence this `blog_url` reference at any place within the `post_urls_from_blog 'a` region,
but there is no way of doing that around the for loop because `blog_url` region ends(variable moves out of scope) right before the loop.

Now we should be able to decipher the error message from the previous chapter:

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

Effectively it tells us that at line 42 compiler tries to take a reference to `blog_url` into a region that ends at line 48, but `blog_url` region
ends at line 45, so compiler can't do that. How can we fix this error? 
One way is to put the for loop before `std::thread::spawn`. This way the regions will be aligned and
`blog_url` will be safe to use at any line of the `post_urls_from_blog 'a` region. But our code is not executing in parallel this way. 


```rust,noplayground
fn main() {
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
/---blog_url region 
|   let blog_url = get_blog_url(); 
|
|/--post_urls_from_blog 'a region
||  let post_urls: Vec<_> = post_urls_from_blog(crawler_results, &blog_url).collect();
||
||  for url in post_urls {
||      process_post(url);
||  }
|-
|   let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
-
    handle.join().expect("Everything will be fine");
}
```

Another way is to get away with clones. But let's look at our region carefully:

```rust,noplayground
/---blog_url region 
|   let blog_url = get_blog_url(); 
|
|/--post_urls_from_blog 'a region
||  let post_urls: Vec<_> = post_urls_from_blog(crawler_results, &blog_url).collect();
||
||  let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
-|
 |  for url in post_urls {
 |      process_post(url);
 |  }
 -
```

We don't really need the `blog_url` reference to be valid inside the for loop, we only care about `post_urls` there.
This `post_urls_from_blog 'a` region is essentially a region for our `post_urls`, the `blog_url` region could be much
smaller, but the function signature asks the compiler to infer only a single region, so the `blog_url` reference ends up coupled
with the `post_urls` references. What we actually want is to ask the compiler to infer 2 regions for this function: the one for `post_urls`
and the one for `blog_url`, so regions in `main` would look like that.

```rust,noplayground
/---blog_url region 
|   let blog_url = get_blog_url(); 
|
|/--post_urls_from_blog 'post_urls region
||/-post_urls_from_blog 'blog_url region
||| let post_urls: Vec<_> = post_urls_from_blog(crawler_results, &blog_url).collect();
||-
||  let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
-|
 |  for url in post_urls {
 |      process_post(url);
 |  }
 -
```

This way `post_urls_from_blog 'blog_url` region is only 1-line long and borrowing `blog_url` for this line is fine, when `post_urls_from_blog 'post_urls` region
holds only `post_urls` references and doesn't care about the `blog_url` region at all. Let's try to split this `'a` region!

## Splitting `post_urls_from_blog 'a` region
To ask the compiler to infer 2 regions instead of 1 we just need to introduce a second lifetime parameter to the function signature:

```rust,noplayground
fn post_urls_from_blog<'post_urls, 'blog_url>(
    items: &'post_urls [DiscoveredItem], 
    blog_url: &'blog_url str,
) -> impl Iterator<Item = &'post_urls str> {
    // ...
}
```

We're taking post urls from input items, so `items` clearly belong to `post_urls` region. We use `blog_url` only for filtering, so it belongs to its own 
`blog_url` region. Iterator returns post urls from the input `items`, so `Item = &str` must belong to `post_urls` region. But what about an `Iterator` itself?
We're iterating items, so let's assign it to `post_urls` region.

```rust,noplayground
fn post_urls_from_blog<'post_urls, 'blog_url>(
    items: &'post_urls [DiscoveredItem], 
    blog_url: &'blog_url str,
) -> impl Iterator<Item = &'post_urls str> + 'post_urls {
    // ...
}
```

And now I want to go through one more example to emphasize that borrow checker analyzes each function completely independetly:

```rust,noplayground
fn uaf(options: &CrawlerOptions) {
    let items = crawler::run(options);

    let blog_url = get_blog_url();
    let iterator = post_urls_from_blog(&items, &blog_url);
    drop(blog_url);

    for url in iterator {
        do_stuff(url);
    }
}
```

Let's infer regions for this function:

```rust,noplayground
fn uaf(options: &CrawlerOptions) {
/---items region
|   let items = crawler::run(options);
|
|   let blog_url = get_blog_url();
|/--post_urls_from_blog 'post_urls region
||  let iterator = post_urls_from_blog(&items, &blog_url);
||  drop(blog_url);
||
||  for url in iterator {
||      do_stuff(url);
||  }
--
}
```

`post_urls_from_blog 'post_urls` holds an iterator, and a reference to the `items` variable and it can dereference them at any line of this region
because `items` region outlives the `'post_urls` region.

```rust,noplayground
fn uaf(options: &CrawlerOptions) {
    let items = crawler::run(options);
/---blog_url region
|   let blog_url = get_blog_url();
|/--post_urls_from_blog 'blog_url region
||  let iterator = post_urls_from_blog(&items, &blog_url);
|-  drop(blog_url);
-
    for url in iterator {
        do_stuff(url);
    }
  
}
```

`post_urls_from_blog 'blog_url` region holds just a reference to the `blog_url` variable. It's an input argument, therefore the reference should be valid
only for the time of the function call, so the region is 1-line long. `blog_url` region clearly outlives this 1-line region, so it's safe to create a borrow
there. As the result the `uaf` function passes the borrow checking just perfectly, but if we think about what's going on, we quickly realize that the iterator
holds a reference to `blog_url` internally to do the comparisons, so in fact we have a use after free memory bug here. `post_urls_from_blog` function
signature doesn't tell anything about this internal borrow, so borrow checker can't spot any issue while analyzing the `uaf` function. Luckily for us
it can spot the issue during the analysis of the `post_urls_from_blog` function body which is done only once and independently from the `uaf` function.

```rust
#struct DiscoveredItem {
#    blog_url: String,
#    post_url: String,
#}
fn post_urls_from_blog<'post_urls, 'blog_url>(
    items: &'post_urls [DiscoveredItem], 
    blog_url: &'blog_url str,
) -> impl Iterator<Item = &'post_urls str> + 'post_urls {
    items.iter().filter_map(move |item| {
        if item.blog_url == blog_url {
            Some(item.post_url.as_str())
        } else {
            None
        }
    })
}
```

The borrow checker emits the following error for this implementation:

```
error[E0623]: lifetime mismatch
  --> src/main.rs:11:6
   |
10 |     blog_url: &'blog_url str,
   |               -------------- this parameter and the return type are declared with different lifetimes...
11 | ) -> impl Iterator<Item = &'post_urls str> + 'post_urls {
   |      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |      |
   |      ...but data from `blog_url` is returned here

For more information about this error, try `rustc --explain E0623`.
error: could not compile `playground` due to previous error
```

It was able to spot that the iterator borrows `blog_url` from the `'blog_url` region, but the signature
suggests that the iterator borrows only from the `'post_urls` region, so the borrow checker threw a `lifetime mismatch` error.
Let's reflect this `blog_url` borrow in our signature by assigning the iterator to the `'blog_url` region.

```rust
#struct DiscoveredItem {
#    blog_url: String,
#    post_url: String,
#}
fn post_urls_from_blog<'post_urls, 'blog_url>(
    items: &'post_urls [DiscoveredItem], 
    blog_url: &'blog_url str,
) -> impl Iterator<Item = &'post_urls str> + 'blog_url {
    items.iter().filter_map(move |item| {
        if item.blog_url == blog_url {
            Some(item.post_url.as_str())
        } else {
            None
        }
    })
}
```

```
   Compiling playground v0.0.1 (/playground)
error[E0623]: lifetime mismatch
  --> src/main.rs:11:6
   |
10 |     blog_url: &'blog_url str,
   |               -------------- this parameter and the return type are declared with different lifetimes...
11 | ) -> impl Iterator<Item = &'post_urls str> + 'blog_url {
   |      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |      |
   |      ...but data from `items` is returned here

For more information about this error, try `rustc --explain E0623`.
error: could not compile `playground` due to previous error
```

Compilation failed with the same error. Hmm... it's time to resort to magic!

```rust
#struct DiscoveredItem {
#    blog_url: String,
#    post_url: String,
#}
fn post_urls_from_blog<'post_urls, 'blog_url>(
    items: &'post_urls [DiscoveredItem], 
    blog_url: &'blog_url str,
) -> impl Iterator<Item = &'post_urls str> + 'blog_url
where
    'post_urls: 'blog_url
{
    items.iter().filter_map(move |item| {
        if item.blog_url == blog_url {
            Some(item.post_url.as_str())
        } else {
            None
        }
    })
}
```

And now everything compiles, including the example from the previous chapter.
Check it out:

```rust
#struct DiscoveredItem {
#    blog_url: String,
#    post_url: String,
#}
#fn post_urls_from_blog<'post_urls, 'blog_url>(
#    items: &'post_urls [DiscoveredItem], 
#    blog_url: &'blog_url str,
#) -> impl Iterator<Item = &'post_urls str> + 'blog_url
#where
#    'post_urls: 'blog_url
#{
#    items.iter().filter_map(move |item| {
#        if item.blog_url == blog_url {
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

We will demistify the added `where` clause and will understand the last compilation error in the next chapter,
but before going further make sure you understood the material from this chapter.

## Chapter exercises
1. The chapter says when we encounter a function call we need to infer minimal regions for it at the invocation point.
Why do we want these regions to be minimal?
1. Assume this signature compiles:
```rust,noplayground
fn post_urls_from_blog<'post_urls, 'blog_url>(
    items: &'post_urls [DiscoveredItem], 
    blog_url: &'blog_url str,
) -> impl Iterator<Item = &'post_urls str> + 'blog_url {
    // ...
}
```

Go back to the `uaf` example. Infer and validate regions for the `uaf` using this `post_urls_from_blog` signature.
Does `uaf` compile?


[^continuous]: The book is written in the NLL era.
