QuickCheck for Rust with shrinking. QuickCheck is a way to do property based 
testing using randomly generated input. This crate comes with the ability to 
randomly generate and shrink integers, floats, tuples, booleans, lists, 
strings, options and results.

The shrinking strategies for lists and numbers use a binary search to cover 
the input space quickly. (It should be the same strategy used in
[Koen Claessen's QuickCheck for 
Haskell](http://hackage.haskell.org/package/QuickCheck).)

[![Build status](https://api.travis-ci.org/BurntSushi/quickcheck.png)](https://travis-ci.org/BurntSushi/quickcheck)


### Documentation

The API is comprehensively documented (with examples):
[http://burntsushi.net/rustdoc/quickcheck/](http://burntsushi.net/rustdoc/quickcheck/).


### Simple example

Here's a complete working program that tests a function that reverses a vector:

```rust
extern crate quickcheck;

use quickcheck::quickcheck;

fn reverse<T: Clone>(xs: &[T]) -> ~[T] {
    let mut rev = ~[];
    for x in xs.iter() {
        rev.unshift(x.clone())
    }
    rev
}

fn main() {
    quickcheck(|xs: ~[int]| xs == reverse(reverse(xs)));

    // You can also use regular `fn` types.
    fn prop(xs: ~[int]) -> bool { xs == reverse(reverse(xs)) }
    quickcheck(prop);
}
```


### Installation

Given that Rust hasn't hit `1.0` yet---and the recent deprecation of 
`rustpkg`---installing Rust libraries is pretty grim at the moment.
More than that, I am keeping this crate in sync with Rust's master branch (as 
enforced by `travis-ci`), so you'll need to build Rust from source first.

If you have an up-to-date Rust, then the easiest way to get going is to just 
clone this repo and build it:

```bash
git clone git://github.com/BurntSushi/quickcheck
cd quickcheck
rustc --crate-type lib ./src/lib.rs # makes libquickcheck-{version}.rlib in CWD
rustc -L ./ ./examples/reverse.rs
RUST_LOG=quickcheck ./reverse
```

Also, `quickcheck` has a `cargo-lite.conf` that seems to work. With a Python 2 
`pip` binary, install `cargo-lite` with `pip2 install cargo-lite` and then
install `quickcheck` with
`cargo-lite install git://github.com/BurntSushi/quickcheck`.
It looks like the library ends up in
`~/.rust/lib/{arch-os}/libquickcheck-{version}.rlib`.
Even better, it looks like `rustc` knows to look there, so you don't have to 
use `-L`. So for example, to run the `reverse` example if you used `cargo-lite` 
to install:

```bash
rustc ~/.rust/src/quickcheck/examples/reverse.rs
RUST_LOG=quickcheck ./reverse
```

N.B. The `RUST_LOG=quickcheck` enables `debug!` so that it shows useful output 
(like the number of tests passed). This is **not** needed to show witnesses for 
failures.


### Shrinking

Shrinking is a crucial part of QuickCheck that simplifies counter-examples for 
your properties automatically. For example, if you erroneously defined a 
function for reversing vectors as: (my apologies for the contrived example)

```rust
fn reverse<T: Clone>(xs: &[T]) -> ~[T] {
    let mut rev = ~[];
    for i in iter::range(1, xs.len()) {
        rev.unshift(xs[i].clone())
    }
    rev
}
```

And a property to test that `xs == reverse(reverse(xs))`:

```rust
quickcheck(|xs: ~[int]| xs == reverse(reverse(xs)));
```

Then without shrinking, you might get a counter-example like:

```
[quickcheck] TEST FAILED. Arguments: ([-17, 13, -12, 17, -8, -10, 15, -19, 
-19, -9, 11, -5, 1, 19, -16, 6])
```

Which is pretty mysterious. But with shrinking enabled, you're nearly 
guaranteed to get this counter-example every time:

```
[quickcheck] TEST FAILED. Arguments: ([0])
```

Which is going to be much easier to debug.


### Case study: The Sieve of Eratosthenes

The [Sieve of Eratosthenes](http://en.wikipedia.org/wiki/Sieve_of_Eratosthenes)
is a simple and elegant way to find all primes less than or equal to `N`.
Briefly, the algorithm works by allocating an array with `N` slots containing
booleans. Slots marked with `false` correspond to prime numbers (or numbers
not known to be prime while building the sieve) and slots marked with `true`
are known to not be prime. For each `n`, all of its multiples in this array
are marked as true. When all `n` have been checked, the numbers marked `false`
are returned as the primes.

As you might imagine, there's a lot of potential for off-by-one errors, which 
makes it ideal for randomized testing. So let's take a look at my 
implementation and see if we can spot the bug:

```rust
fn sieve(n: uint) -> ~[uint] {
    if n <= 1 {
        return ~[]
    }

    let mut marked = vec::from_fn(n+1, |_| false);
    marked[0] = true; marked[1] = true; marked[2] = false;
    for p in iter::range(2, n) {
        for i in iter::range_step(2 * p, n, p) {
            marked[i] = true;
        }
    }
    let mut primes = ~[];
    for (i, m) in marked.iter().enumerate() {
        if !m { primes.push(i) }
    }
    primes
}
```

Let's try it on a few inputs by hand:

```
sieve(3) => [2, 3]
sieve(5) => [2, 3, 5]
sieve(8) => [2, 3, 5, 7, 8] # !!!
```

Something has gone wrong! But where? The bug is rather subtle, but it's an 
easy one to make. It's OK if you can't spot it, because we're going to use
QuickCheck to help us track it down.

Even before looking at some example outputs, it's good to try and come up with 
some *properties* that are always satisfiable by the output of the function. An 
obvious one for the prime number sieve is to check if all numbers returned are 
prime. For that, we'll need an `is_prime` function:

```rust
fn is_prime(n: uint) -> bool {
    if n == 0 || n == 1 {
        return false
    } else if n == 2 {
        return true
    }

    let max_possible = (n as f64).sqrt().ceil() as uint;
    for i in iter::range_inclusive(2, max_possible) {
        if n % i == 0 {
            return false
        }
    }
    return true
}
```

All this is doing is checking to see if any number in `[2, sqrt(n)]` divides
`n` with a few base cases for `0`, `1` and `2`.

Now we can write our QuickCheck property:

```rust
fn prop_all_prime(n: uint) -> bool {
    let primes = sieve(n);
    primes.iter().all(|&i| is_prime(i))
}
```

And finally, we need to invoke `quickcheck` with our property:

```rust
fn main() {
    quickcheck(prop_all_prime);
}
```

A fully working source file with this code is in `examples/sieve.rs`.

The output of running this program has this message:

```
[quickcheck] TEST FAILED. Arguments: (4)
```

Which says that `sieve` failed the `prop_all_prime` test when given `n = 4`.
Because of shrinking, it was able to find a (hopefully) minimal counter-example 
for our property.

With such a short counter-example, it's hopefully a bit easier to narrow down 
where the bug is. Since `4` is returned, it's likely never marked as being not 
prime. Since `4` is a multiple of `2`, its slot should be marked as `true` when 
`p = 2` on line 23.

Ah! But does the `range_step` function include `n`? Its documentation says

> Return an iterator over the range [start, stop) by step. It handles overflow 
> by stopping.

Shucks. The `range_step` function will never yield `4` when `n = 4`. We could 
use `n + 1`, but the `std::iter` crate also has a
[`range_step_inclusive`](http://static.rust-lang.org/doc/master/std/iter/fn.range_step_inclusive.html)
which seems clearer.

Changing the call to `range_step_inclusive` results in `sieve` passing all 
tests for the `prop_all_prime` property.


### What's not in this port of QuickCheck?

I think I've captured the key features, but there are still things missing:

* As of now, only functions with 3 or fewer parameters can be quickchecked.
This limitation can be lifted to some `N`, but requires an implementation
for each `n` of the `Testable` trait.
* Functions that fail because of a runtime error (e.g., out-of-bounds) are not
caught by QuickCheck. Therefore, such failures will not have a witness attached
to them. (I'd like to fix this, but I don't know how.)
* `Coarbitrary` does not exist in any form in this package. I think it's 
possible; I just haven't gotten around to it yet.

Please let me know if I've missed anything else.


### Laziness

A key aspect for writing good shrinkers is a good lazy abstraction. For this,
I chose iterators. My insistence on this point has resulted in the use of an
existential type, which I think I'm willing to live with.

Note though that the shrinkers for lists and integers are not lazy. Their
algorithms are more complex, so it will take a bit more work to get them to
use iterators like the rest of the shrinking strategies.


### Request for review

This is my first Rust project, so I've undoubtedly written unidiomatic code. In 
fact, it would be fair to say that the code in this project just happened to be 
what I could manage to get by the compiler.

I think my primary concern is whether I'm using the region types correctly.
I'd also like to trap runtime failures in functions so that they can be
reported as a test failure from within QuickCheck. (In typical use, the test
will fail anyway, but if QuickCheck can catch it, then we can attach a witness
to the failure.)

Also, I would like to avoid using macros for building abstractions. (I'm not 
opposed to using them to generating trait implementations---as is done in the 
standard library---but I haven't learned them yet.)
