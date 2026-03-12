+++
title = "On Tony Hoare"
date = 2026-03-12
+++

Every so often something happens that makes me remember that computer programming is a new thing.

Something that makes me remember that we are still in the infancy of computing. That we don't really know what we are doing and we are still figuring it out. We can't boast the long histories of fields like math, medicine, or mechanics.



Sir Tony Hoare's recent passing made me remember that. 

Computing is such a young field that many of our pioneers are still around.[^2] Hoare is an inspiration to me and I thought it would be nice to write a little about his many contributions to our young field.

While Hoare was most known for **Quicksort**[^1], I want to cover some of his other work:
- Record Handling (structs, enums, null)
- Monitor Synchronization
- Floyd-Hoare Logic
- Communicating Sequential Processes

# Record Handling (1966)

Hoare introduced the idea of "record structures" which are now just called structs. In this same essay, he introduces "record subclasses" which are akin to discriminated unions (think Rust enums). Finally, if that wasn't enough, he introduces the notion of a "null reference" which he calls his "billion dollar mistake." Anytime you get a NullPointerException? That's him too.

# Monitor Synchronization

Hoare elaborated and popularized the Monitor concurrency primitive invented by Per Brinch Hansen. Monitors solve a common issue in concurrent programs: making threads (efficiently) wait for a condition to become true before continuing. 

# Floyd-Hoare Logic

In "An Axiomatic Basis for Computer Programming" (1969), Hoare develops a systematic method for reasoning about the meaning of computer programs. One of the key ideas of this logic is what is now known as a "Hoare triple": `P {Q} R` where `P` is a precondition, `Q` is a program, and `R` is a postcondition. If `P` holds, then running `Q` will lead to `R` being true. With axioms and inference rules for imperative programming, you can write a proof about the behavior of programs which is exactly what modern verification languages like [Dafny](https://dafny.org/) and [Verus](https://github.com/verus-lang/verus) do!

# Communicating Sequential Processes (CSP)

The core idea of CSP is modeling concurrent systems as a set of independent processes which do not share memory but instead communicate through "channels." Processes can send messages cross these channels, and other processes can recieve messages through them.

You can see the echos of CSP's influence in languages like Erlang and Go.

These are just some of Tony Hoare's most influencial ideas but I need to stop somewhere (and it's 1:40AM here so I should logoff...).

Rest in peace.

[^1] [Rust](https://doc.rust-lang.org/std/primitive.slice.html#method.sort_unstable) and [C++](https://en.cppreference.com/w/cpp/algorithm/sort.html#Notes) both include a "sort" algorithm which is a hybrid sorting algorithm which uses Quicksort for large arrays, and fallsback to heapsort and insertion sort for smaller arrays. The details vary from language to language but Quicksort is still state of the art!

[^2] Alan Turing could've probably made it to the 1990s if the Brits weren't as homophobic. He would be 114 today.
