+++
title = "A Few Good Ideas in Programming Languages"
date = 2026-05-10
+++

Here are just a few programming language features I love:

# 1. Flow Typing

A great idea that I first encountered in [Crystal](https://crystal-lang.org), is the idea of *flow typing*.

Crystal is a compiled programming langauge with static type-checking with syntax very similar to the [Ruby programming language](https://www.ruby-lang.org/en/). Something that keeps Crystal feeling like a dynamically typed language is how a variable can be assigned to multiple types throughout the course of its lifetime, unlike most statically typed languages I've used. Here is an example:

```Crystal
my_var = 5

# my_var's type here is Int32
assert(my_var.is_a?(Int32))

if some_complex_condition()
  my_var = "hello!"

  # my_var's type here is String
  assert(my_var.is_a?(String))
end

# What is my_var's type here?
#
# It's Int32 | String
assert(my_var.is_a?(Int32 | String))
```

What's most interesting about this is that there is some time when `my_var` is just a `Int32`, there is a region where it is guaranteed to be a `String` and then there is a region where the compiler cannot actually guarantee one or the other... so its type is the union of all the possibilities. Now if you try to run a `String` method on `my_var`, it'll fail because its not a `String`, its a `Int32 | String` and the compiler will force you to add a check like `if my_var.is_a?(String)` which *narrows* the possible types to just a `String`.

This is a great example of using fancy type inference to make a compiled language *feel* dynamic without paying much of a runtime penalty. 

Typescript also has [flow typing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#control-flow-analysis) and [type narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)!

# 2. Borrow Checking

[Rust](https://rust-lang.org/) is a systems programming language which guarantees memory safety without a garbage collector. 

One large class of memory safety bugs in concurrent programs is the loathsome data race: when multiple threads read and write the same memory location simultaneously without synchronization.

The borrow checker is how Rust is able to statically prevent data races at compile-time. It enforces the following [rules](https://doc.rust-lang.org/1.8.0/book/references-and-borrowing.html):

- Any borrow must not out-live the scope of the owner
- You can EITHER have:
  - exactly 1 mutable reference (`&mut T`)
  - OR 1 or more immutable references (`&T`)

This might remind you of a [readers-writer lock](https://en.wikipedia.org/wiki/Readers%E2%80%93writer_lock) which is a lock which allows many readers OR a singular writer. This is because to prevent data races, we only need to synchronize reads with respect to writes. The point of concurrent synchronization is to serialize the writes to a given memory location.

I love the borrow checker because it is such an elegant solution to the problem of data races and are a zero-cost abstraction which only comes at the cost of compile-time checks and added complexity. Added complexity is a real tradeoff but if you are writing concurrent programs, this complexity is inherent to the subject matter.

# 3. Contract Programming

If you've programmed much, you've probably come across the humble assert statement, which is used to error / report if a given invariant isn't upheld. Every program has invariants that it needs to uphold. Things like "this date is always past this other date" or "this tree is always balanced."

The [D programming language](https://dlang.org/) (such an underrated language) supports [contract programming](https://dlang.org/spec/contracts.html) where it provides syntactic support for describing more complex invariants which can be at the function-level or object-level.

D has standard assert statements:
```D
double always_positive = 100;
assert(always_positive > 0);
```

But D also has `enforce` which is used to mark a difference in semantics. Asserts are used for program invariant violations. If an assert is triggered, this should indicate a correctness bug in our program. `enforce` instead throws an exception due to some external issue: something like a user input out of bounds or environment issue.

```D
enforce(length >= 7, "Must be at least 7.");
```

Additionally, D has syntactic support for pre- and post-conditions for functions.  Here's an example taken from [Programming in D](https://ddili.org/ders/d.en/contracts.html):

```D
int daysInFebruary(int year)
out (result) {
    assert((result == 28) || (result == 29));

} do {
    return isLeapYear(year) ? 29 : 28;
}
```

Here the `daysInFebruary` function has a post-condition which is that it should only ever return 28 or 29; everything else is definitely a logical error in the function. 

And finally, D has class-level invariants which are used to guarantee that object data is is always consistent. Here is a slightly more complex example tying everything together:

```D
class BankAccount {
    private double balance;

    invariant() {
        balance >= 0; // object-level invariant, always checked
    }

    this(double initialBalance)
    in (initialBalance >= 0, "Initial balance cannot be negative")
    {
        balance = initialBalance;
    }

    void deposit(double amount)
    in  (amount > 0, "Deposit amount must be positive")
    out (; balance == balance + amount) // checked after method returns
    {
        balance += amount;
    }

    void withdraw(double amount)
    in  (amount > 0,           "Withdrawal amount must be positive")
    in  (amount <= balance,    "Insufficient funds")
    out (; balance >= 0,       "Balance must remain non-negative")
    {
        balance -= amount;
    }

    double getBalance()
    out (result; result >= 0, "Balance returned must be non-negative")
    {
        return balance;
    }
}
```

This `invariant()` block is far cleaner and easier to maintain than having some consistency check function run at the beginning and end of every class method, and it has idomatic meaning.

-- Pranoy
