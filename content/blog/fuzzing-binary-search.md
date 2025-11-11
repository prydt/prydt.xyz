+++
title = "Fuzzing Binary Search"
date = 2025-11-10
+++


Something close to a holy grail for me in programming is an automated test generator.

Given some function, I would love to be able to throw my code at another program which probes my function and tries all the weird edge cases that I am sure to forget when writing unit tests or defensive asserts. Writing tests manually? That's boring! And oftentimes when something is boring, getting a computer to do it is a solution. 


## Symbolic Execution (Klee)
At first, I wanted to try out symbolic execution with [Klee](https://klee-se.org). Klee is a symbolic execution engine for LLVM which runs programs with symbolic values for the variables, which allows it to effectively simulate running the program on all possible inputs at once! This is extremely cool, although you can probably already guess that this can be quite computationally expensive, even with Klee being extremely clever. But for constrained enough tasks, this absolutely fits my defintiion of a holy grail. The issue is that I wanted to try this with Rust, and it looks like [using Klee with Rust is not currently well supported](https://www.unwoundstack.com/blog/klee-and-rust.html).


Alright... well maybe I'll try to get that working sometime later, but for now let's try something else. Fuzzing!


## Fuzzing
Fuzzing is a technique for finding interesting test cases using randomized inputs. While this by itself is not too ground-breaking, more sophisticated fuzzers like American Fuzzy Lop (AFL) use techniques for directing the random input to maximize code coverage of the test cases. This allows fuzzers to try to take every path within your code and weasel their way into control-flow states you might not have expected.

For my very basic trial of fuzzing, I will be using two libraries: [proptest](https://github.com/proptest-rs/proptest)[^1] and [cargo-fuzz (libfuzzer)](https://github.com/rust-fuzz/cargo-fuzz). 

## The problem

I want to test these libraries on a small but important well-known bug: [integer overflow in binary search](https://research.google/blog/extra-extra-read-all-about-it-nearly-all-binary-searches-and-mergesorts-are-broken/).

```Java
public static int binarySearch(int[] a, int key) {
        int low = 0;
        int high = a.length - 1;

        while (low <= high) {
            int mid = (low + high) / 2; // BUG: low + high can overflow
            int midVal = a[mid];

            if (midVal < key)
                low = mid + 1;
            else if (midVal > key)
                high = mid - 1;
            else
                return mid; // key found
        }
        return -(low + 1);  // key not found.
    }
```

Here the code is in its original Java from the article. I've rewritten it in Rust because I wanted to try fuzzing / property testing in Rust[^2].

```Rust
pub fn binary_search(needle: i32, haystack: &Vec<i32>) -> Option<usize> {
    let mut low = 0;
    let mut high = haystack.len() - 1;

    while low <= high {
        let mid = (low + high) / 2;
        let mid_val = haystack[mid];

        if mid_val < needle {
            low = mid + 1;
        } else if mid_val > needle {
            high = mid - 1;
        } else {
            return Some(mid);
        }
    }
    return None;
}
```
I've modified the signature for simplicity. In the original Java `binarySearch`, a failure to find the item returns a negative insertion index, the index where the item would be inserted into the list. I've replaced that with an `Option` type which makes the failure clearer, although it removes some information. In the Rust standard library, they return the insertion index with a `Result` type.

Now something I didn't realize immediately while translating this code manually: in the Java code, low and high are `int`s (32-bit), while in Rust, they are `usize` (pointer-sized unsigned int). This leads to some issues since the implementation assumes that high can be negative in the case of an empty list. -- This is where proptest and cargo-fuzz come in!

## proptest
```Rust
use proptest::prelude::*;

#[allow(dead_code)]
fn needle_and_haystack() -> impl Strategy<Value = (i32, Vec<i32>)> {
    (
        0..100i32,
        proptest::collection::vec(prop::num::i32::ANY, 0..100),
    )
}

proptest! {
    #[test]
    fn proptest_binary_search((needle, mut haystack) in needle_and_haystack()) {
        haystack.sort_unstable();
        let result = binary_search(needle, &haystack);

        match result {
            Some(index) => {
                prop_assert_eq!(haystack[index], needle);
            },
            None => {
                prop_assert!(!haystack.contains(&needle));
            }
        }
    }
}
```
proptest is a property-testing framework for Rust which automatically shrinks failing test cases.

Here is the result of running the proptest:
```
thread 'proptest_binary_search' (95480) panicked at src/main.rs:36:1:
Test failed: attempt to subtract with overflow.
minimal failing input: (needle, mut haystack) = (
    0,
    [],
)
```

It correctly found that an empty list is an error since high is now a usize!

## cargo-fuzz
```Rust
#![no_main]

use fuzz_binary_search::binary_search;
use libfuzzer_sys::fuzz_target;

fuzz_target!(|input: (i32, Vec<i32>)| {
    let (needle, haystack) = input;
    let result = binary_search(needle, &haystack);
    match result {
        Some(idx) => assert_eq!(haystack[idx], needle),
        None => assert!(!haystack.contains(&needle)),
    }
});
```
cargo-fuzz, which uses LLVM's libfuzzer under the hood, is also able to find several errors with this implementation which has been shoddily translated.

P.S. wanna get your LSP to work in the `fuzz/` directory that cargo-fuzz uses? You need to add this to your Cargo.toml.
```toml
[workspace]
members = [
  ".",
  "fuzz"
]
```


## Some remarks
Although proptest felt a little more clunky when defining input constraints, this does help speed up the search through guidance.

Something that I didn't expect but had fun learning was that actually reproducing this bug from 2006 in Rust would have led me to write some quite awkward Rust[^3]! This is exactly what I want from a programming language. I want to be nudged in the correct direction. Rust makes it difficult to "hold it wrong." Modern programming languages really ought to be [poka-yoke](https://en.wikipedia.org/wiki/Poka-yoke).


This was just an excuse to play around with fuzz testing and it works remarkably well for the amount of effort it takes. I think that if you are writing some data structure or some non-trivial transformation, there's really no reason not to fuzz your program. Fuzzing is of course no cure-all, but it is easy enough to setup that you don't really have an excuse not to try it!


I've made a git repo for these little experiments so if you want to see the complete code for any of these examples, here: [https://github.com/prydt/etudes/tree/main/fuzz-binary-search](https://github.com/prydt/etudes/tree/main/fuzz-binary-search)

## Further reading
- [The Proptest Book](https://proptest-rs.github.io/proptest/)
- [The Rust Fuzz Book](https://rust-fuzz.github.io/book/introduction.html), covering cargo-fuzz and AFL
- [Pulling JPEGs out of thin air](https://lcamtuf.blogspot.com/2014/11/pulling-jpegs-out-of-thin-air.html), a really neat example of the powers of fuzzing



### Footnotes
[^1]: proptest is actually a library for property testing which is slightly different from fuzzing. Property testing has more guidance from the programmer for how the inputs are generated, while fuzzing tends to be more black-box.
[^2]: Why for real? -- For fun!
[^3]: I would have needed to intentionally cast low, high into `i32` and then cast back into `usize` for indexing. 