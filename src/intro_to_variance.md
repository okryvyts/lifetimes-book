# Introduction to variance

Suppose we have the following generic structure:

```rust
struct S<T: ?Sized>(RustT);
```

Let's define a function with a lifetime cast:

```rust
#struct S<T: ?Sized>(T);
fn shortener<'a: 'b, 'b>(s: S<&'a str>) -> S<&'b str> {
    s
}
```

As we have learnt from the previous chapter the function compiles because there is a `'a: 'b` bound, so `'a ~> 'b` is allowed.
We can simplify the example a bit. In Rust there is an implicit rule for the `'static` lifetime that `forall 'a | 'static: 'a`,
so the following signature compiles too.

```rust
#struct S<T>(T);
// implicit 'static: 'a
fn shortener<'a>(s: S<&'static str>) -> S<&'a str> {
    s
}
```

Now let's replace our generic struct with a `Cell` struct from the core library which is defined approximately the same way:

```rust
struct Cell<T: ?Sized> {
    value: UnsafeCell<T>
}

struct UnsafeCell<T: ?Sized> {
    value: T
}
```

And we have a compilation error:

```rust
use std::cell::Cell;
fn shortener<'a>(cell: Cell<&'static str>) -> Cell<&'a str> {
    cell
}
```

```Compiling playground v0.0.1 (/playground)
error[E0308]: mismatched types
 --> src/main.rs:6:5
  |
6 |     cell
  |     ^^^^ lifetime mismatch
  |
  = note: expected struct `Cell<&'a str>`
             found struct `Cell<&'static str>`
note: the lifetime `'a` as defined here...
 --> src/main.rs:5:14
  |
5 | fn shortener<'a>(cell: Cell<&'static str>) -> Cell<&'a str> {
  |              ^^
  = note: ...does not necessarily outlive the static lifetime

For more information about this error, try `rustc --explain E0308`.
error: could not compile `playground` due to previous error
```

We get the same compilation error if we omit specifying the `'a: 'b` relationship in our original example:

```rust
#struct S<T: ?Sized>(T);
fn shortener<'a, 'b>(s: S<&'a str>) -> S<&'b str> {
    s
}
```

So it looks like `Cell` is somewhat exceptional and `'a: 'b` doesn't work for it for some reason. Let's 
figure out why we want to have exceptions from the rules we've learnt in the first place.

Assume the following function compiles

```rust
use std::cell::Cell;
fn shortener<'cell, 'a>(cell: &'cell Cell<&'static str>, s: &'a str) -> &'cell Cell<&'a str> {
    cell.set(s);
    cell
}
```

Let's infer the regions in this function:

```rust 
fn uaf() {
    let cell = Cell::new("Static str");
    let s = String::new("UAF");

    let cell2 = shortener(&cell, &s);
    drop(s);
    println!("{}", cell2);
    
}
```

We need to infer 2 regions: `'cell` and `'a` following the last usage rule.

```
fn uaf() {
/--- cell region
|   let cell = Cell::new("Static str");
|   let s = String::new("UAF");
|
|/-- shortener 'cell region
||   let cell2 = shortener(&cell, &s);
||   drop(s);
||   println!("{}", cell2);
|-
-    
}
```

The `'cell` holds a reference to `cell` and a `cell2` output reference. `&cell` is a regular input reference without additional bounds, 
so the region for it is one line long, however `cell2` is used in the print statement, so the region was expanded to be 3 lines long.
`cell region` outlives the `shortener 'cell` region, so we can take a reference and dereference it at any line within the `'shortener cell`
region, there are no errors at this point.

```
fn uaf() {
    let cell = Cell::new("Static str");
/--- s region
|   let s = String::new("UAF");
|
|/-- shortener 'a region
||   let cell2 = shortener(&cell, &s);
||   drop(s);
-|   println!("{}", cell2);
 -
     
}
```
The `shortener 'a` region holds a reference to `s` and an internal reference inside the `cell2`(we can think it holds cell2 itself for simplicity). `&s` is a regular
input reference, so `'a` with respect of `&s` is only one line long, however `cell2` is used in the print statement which makes `shortener 'a` region 3 lines
long. But we can't dereference `&s` at the line with `println!` because the `s` region ends right before the `println!` statement.
There is an error and compiler successfully caught a use after free bug. However, `cell` is still in scope, so instead of using `cell2` we could have used `cell`
in the print statement:

```
fn uaf() {
/--- cell region
|   let cell = Cell::new("Static str");
|/-- s region
||  let s = String::new("UAF");
||
||/- shortener 'cell region & shortener 'a regions
||| let cell2 = shortener(&cell, &s);
||-
||  drop(s);
|-
|   println!("{}", cell);
-
}
```

Now, as we don't use `cell2`, `shortener 'cell` and `shortener 'a` regions are both only one line long, and we can safely take references
to both `cell` and `s` at this line. But internally the shortener function updates the cell pointing the internal reference to 
the allocated on the heap string which is being dropped right before we're printing it out on the screen. That's a use after free
bug and compiler is unable to prevent it because it analyzes functions independetly and can't see that the cell was updated
inside `shortener`. That's why we need to disable the ability to shorten lifetimes for the `Cell` type. However, hardcoding
types for which shortening rules don't apply is a bad solution. We have the same issue for all types with interior mutability and 
programmers may define they own interior mutable types + there may be others kind of types vulnerable to this same issue. To control
whether we allowed to shorten lifetimes or not there is a mechanism called lifetime variance.


## Variance rules

Variance rules are hardcoded in the compiler for the primitive types and are being inferred for the compound types.

Threre are 3 of them:

- A type is covariant if `'a: 'b` implies `T<'a>: T<'b>`. This is what we've used in the previous chapter to cast `Iterator + 'post_urls` into `Iterator + 'blog_url`. 
Covariance implies that the rules we've learnt work as we discussed and lifetime shortenings are allowed.
- A type is invariant if `'a: 'b` implies nothing. That's what we've seen in the example with the `Cell` type. Basically it's a mechanism to disable lifetime casts.
- A type is contravariant if `'a: 'b` implies `T<'b>: T<'a>`. This is a rule that allows to extend lifeime `'b` to lifetime `'a`. 
It works only for the function arguments and it will be your homework to figure it out.

In practice you'll usually deal with covariance and invariance.

Here is a table from the Nomicon with the variance settings for different types.
As a general rule: 
- All const contexts are covariant
- All mutable/interiory mutable contexts are invariant
- Function arguments are contravariant

Type          | 'a            | T         | U
--------------|---------------|-----------|----------
&'a T         | covariant     | covariant |
&'a mut T     | covariant     | invariant |
Box<T>        | covariant     |           |
Vec<T>        | covariant     |           |
UnsafeCell<T> | invariant     |           |
Cell<T>       | invariant     |           |
fn(T) -> U    | contravariant | covariant |
*const T      | covariant     |           |
*mut T        | invariant     |           |

It may be a bit confusing to see that variance is applied to a lifetime and some type T. 
That's because `T` may be a reference itself(like `&'s str`). Let's understand how this rules
work with one more practical example:

```rust 
struct S<'a, T> {
    val: &'a T
}

fn shortener<'a, 'b, 'c, 'd>(s: S<'a, &'b str>) -> S<'c, &'d str> 
where
    'a: 'c,
    'b: 'd,
{
    s
}
```

We have a struct definiton corresponding to that row of the table

Type          | 'a            | T         | U
--------------|---------------|-----------|----------
&'a T         | covariant     | covariant |

The tables shows that `&'a T` is covariant over `'a`. That means that `'a: 'b` implies `'a ~> 'b`.
We show it in our shortener funciton by shortening `'a` to `'c`. Also, `&'a T` is covariant over T.
That means if `T` is a reference the lifetime of the reference is covariant. We show it in `shortener`
by shortening `&'b str` to `&'d str`.


Let's modify our example a bit

```rust 
struct S<'a, T> {
    val: &'a mut T
}

fn shortener<'a, 'b, 'c, 'd>(s: S<'a, &'b str>) -> S<'c, &'d str> 
where
    'a: 'c,
    'b: 'd,
{
    s
}
```
Not type `S` corresponds to this row of the table

Type          | 'a            | T         | U
--------------|---------------|-----------|----------
&'a mut T     | covariant     | invariant |

`shortener` no longer compiles. `T` is invrariant meaning `'b: 'd` doesn't allow `'b ~> 'd`. However `S` is still
covariant over `'a`, so `'a: 'c` should work. And indeed, if we remove `'b ~> 'd` cast from the signature it compiles:


```rust 
struct S<'a, T> {
    val: &'a mut T
}

fn shortener<'a, 'b, 'c>(s: S<'a, &'b str>) -> S<'c, &'b str> 
where
    'a: 'c,
{
    s
}
```

This should be enough material to give you basic undestanding of lifetime variance. In general you want to prefer
covariant contexts, because they're the most flexible ones. Invariant contexts usually lead to some hard 
to grasp lifetime errors because references are tightly bounded to their regions and its very hard to move them into another regions.
We'll see such errors and learn how to deal with them in another chapters. For now you need to get comfortable with the variance concept.

## Chapter exercises

The chapter is called `Introduction to variance` because it only gives you a reasoning why it's needed and a brief overview of how it works.
There is already an awesome [Practical variance tutorial](https://lifetime-variance.sunshowers.io) on the Internet.
Complete it to master the variance concept.
