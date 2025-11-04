# DepositionSort

Depositionsort - An adaptive, O(nlogn), stable, in-place, sorting algorithm


Author: Stew Forster (stew675@gmail.com)

Copyright (C) 2021-2025


# Building and Testing

Depositionsort is written in C.  A Makefile is provided to easily compile and test.
This produces an executable called **ts**   (aka. TestSort)

`ts` can be used to provide a variety of inputs to the sorting algorithms
to test speed, correctness, and sort stability.

```
Usage: ts [options] <sorttype< <num>

[options] is zero or more of the following options
        -a seed     A random number generator seed value to use (default=1)
                    A value of 0 will use a randomly generated seed
        -d <0..100> Disorder the generated set by the percentage given (default=100)
        -f          Data set keys/values range from 0..UINT32_MAX (default)
        -l <num>    Data set keys/values limited in range from 0..(num-1)
        -l n        If the letter 'n' is specified, use the number of elements as the key range
        -o          Use a fully ordered data set (Shorthand for setting disorder factor to 0)
        -r          Reverse the data set order after generating it
        -u          Data set keys/values must all be unique
        -v          Verbose.  Display the data set before sorting it
        -w <num>    Optional workspace size (in elements) to pass to the sorting algorithm

Available Sort Types:
   gq   - GLibc Quick Sort In-Place (Stability Not Guaranteed)
   di   - Simple Deposition Merge Sort In-Place (Stable)
   ds   - Adaptive Deposition Merge Sort In-Place (Stable)
   du   - Adaptive Deposition Merge Sort In-Place (Unstable)
```

For example:

```
./ts -a 5 -d 5 -r rs 1000000
```

will test the Stable Deposition Sort algorithm, with a random seed value of 5, a disordering
factor of 5%, with the data set then reversed.


# Speed

DepositionSort is fast.  Here's a comparison of the time taken to sort 10,000,000
random items on an AMD 9800X3D CPU

```
GLibC Qsort                     1.103s
Deposition Simple*              1.489s
Deposition In-Place Stable      0.927s
Deposition In-Place Unstable    0.827s
Deposition Workspace Stable**   0.815s

*  This is the raw DepositionSort merge algorithm implemented in its most basic manner
   It is sort-stable and in-place, but isn't using any techniques to speed it up.
** This is the Unstable Algorithm, but given a workspace of 25% of the size of the
   data to be sorted, which makes the Unstable algorithm be Sort Stable.
```

What about on mostly sorted data sets?  Here's the speeds when given data that
has been disordered by 5% (ie. 1 in 20 items are out of order)

```
GLibC Qsort                     0.416s
Deposition Simple               0.358s
Deposition In-Place Stable      0.381s
Deposition In-Place Unstable    0.379s
Deposition Workspace Stable     0.365s
```

Somewhat surprising here is the ability of the basic Deposition Sort merge to outpace
the other algorithms

What about reversed data ordering?  Depositionsort doesn't make any explicit checks for
reversed ordering, so it runs at

```
GLibC Qsort                     1.101s
Deposition Simple               1.496s
Deposition In-Place Stable      0.932s
Deposition In-Place Unstable    0.853s
Deposition Workspace Stable     0.813s
```

...and finally when using wholly sorted data, to demonstrate its adaptability.  The
stable in-place variant of DepositionSort takes a little longer than the other 3 due to
it initially scanning to build a working set before "realising", whereas the other
3 variants only take about as long as it takes to do a single pass over the data.

```
GLibC Qsort                     0.212s
Deposition Simple               0.017s
Deposition In-Place Stable      0.021s
Deposition In-Place Unstable    0.018s
Deposition Workspace Stable     0.018s
```

# Discussion

This is my implementation of an O(nlogn) in-place merge-sort algorithm.
There is (almost) nothing new under the sun, and DepositionSort is certainly an
evolution on the work of many others.  It has its roots in the following:

- Merge Sort
- Insertion Sort
- Block Sort
- Grail Sort
- Powermerge - https://koreascience.kr/article/CFKO200311922203087.pdf

This originally started out with me experimenting with sorting algorithms,
and I thought that I had stumbled onto something new, but all I'd done was
independently rediscover Powermerge (see link above)

Here's a link to a StackOverflow answer I gave many years back some time
after I'd found my version of the solution:

https://stackoverflow.com/a/68603446/16534062

Still, Powermerge has a number of glaring flaws, which I suspect is why
it hasn't been widely adopted, and the world has more or less coalesced
around Block Sort and its variants like GrailSort, and so on.  Powermerge's
biggest issue is that the recursion stack depth is unbounded, and it's
rather easy to construct degenerate scenarios where the call stack will
overflow in short order.

I worked to solve those issues, but the code grew in complexity, and then
started to slow down to point of losing all its benefits.  While messing
about with solutions, I created what I call SplitMergeInPlace().  To date
I've not found an algorithm that implements exactly what it does, but it
does have a number of strong similarities to what BlockSort does.

Unlike DepositionMerge(), SplitMerge() doesn't bury itself in the details of
finding the precise optimal place to split a block being merged, but rather
uses a simple division algorithm to choose where to split.  In essence it
takes a "divide and conquer" approach to the problem of merging two arrays
together in place, and deposits fixed sized chunks, saves where that chunk
is on a work stack, and then continues depositing chunks.  When all chunks
are placed, it goes back and splits each one up again in turn into smaller
chunks, and continues.

In doing so, it achieves a stack requirement of 16*log16(N) split points,
where N is the size of the left side array being merged.  The size of the
right-side array doesn't matter to the SplitMerge algorithm.  This stack
growth is very slow.  A stack size of 160 can account for over 10^12 items,
and a stack size of 240 can track over 10^18 items.

SplitMerge() is about 30% slower than DepositionMerge() in practise though, but
it makes for an excellent fallback to the faster DepositionMerge() algorithm
for when DepositionMerge() gets lost in the weeds of chopping up chunks and runs
its work stack out of memory.

I then read about how GrailSort and BlockSort use unique items as a work
space, which is what allows those algorithms to achieve sort stability.  I
didn't look too deeply into how either of those algorithms extract unique
items, preferring the challenge of coming up with my own solution to that
problem.  extract\_uniques() is my solution that also takes a divide and
conquer approach to split an array of items into uniques and duplicates,
and then uses a variant of the Gries-Mills Block Swap algorithm to quickly
move runs of duplicates into place:

Ref: https://en.wikipedia.org/wiki/Block_swap_algorithms

extract\_uniques() moves all duplicates, which are kept in sorted order, to
the left side of the main array, which creates a sorted block that can be
merged in at the end.  When enough unique items are gathered, they are then
used as the scratch work-space to invoke the adaptive merge sort in place
algorithm to efficiently sort that which remains.  This phase appears to
try MUCH harder than either BlockSort or GrailSort do, as it is still
sorting the array as it performs this unique extraction task.

Of course, if an input is provided with less than 0.5% unique items, then
DepositionSort will give up and revert back to using the non-adaptive, but
stable, simple sort.  The thing is, if the data set is THAT degenerate,
then the resulting data is very easy to sort, and the slow simple sort
still runs very quickly.
