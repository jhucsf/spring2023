---
layout: mathjax
title: "Assignment 1: Big integers"
---

*Note: this assignment has not been finalized yet. This assignment description
is incomplete, and details could change.*

Milestone 1: due Monday Jan 30th by 11pm

Milestone 2: due Monday Feb 6th by 11pm

Assignment type: **Pair**, you may work with one partner

# Overview

In this assignment, you will implement a simple C library implementation operations
on a 256-bit unsigned integer data type.

This is a substantial assignment! We strongly recommend that you start working
on it as early as possible, and plan to make steady progress rather than waiting
until the last minute to complete it.

To complete the assignment, you will implement a number of C functions,
and you will also write unit tests to test these functions.

## Milestones, grading criteria

The grading breakdown is as follows:

Milestone 1 (15% of the assignment grade):

* Implementation of functions (15%)
    - `uint256_create_from_u64` 
    - `uint256_create` 
    - `uint256_get_bits` 

Milestone 2 (85% of the assignment grade):

* Implementation of functions (65%)
    - `uint256_create_from_hex`
    - `uint256_format_as_hex`
    - `uint256_add`
    - `uint256_sub`
    - `uint256_mul` (note that this function is worth at most 8% of the overall assignment grade)
* Comprehensiveness and quality of your unit tests (10%)
* Design and coding style (10%)

**Important!** Milestone 1 is intended as a warm-up, since you might not have
written C code in a while. For that reason, it is a very lightweight milestone.
Milestone 2 will require significantly more work.

## Getting started

To get started, download [csf\_assign01.zip](csf_assign01.zip), which contains the
skeleton code for the assignment, and then unzip it.

Note that you can download the zipfile from the command line using the `curl` program:

```bash
curl -O https://jhucsf.github.io/spring2023/assign/csf_assign01.zip
```

Note that in the `-O` option, it is the letter "O", not the numeral "0".

# Unsigned integers

You should be familiar with unsigned integer data types from your previous
experience programming in C and C++. (And, of course, we are covering the properties
of machine-level integer data types in the course.)

In C, the `uint64_t` data type is typically the largest unsigned integer data type
available, and because its values consist of 64 bits, it can represent integer values
in the range $$0$$ to $$2^{64}-1$$, inclusive.

In this assignment, you will implement functions implementing operations on
a data type called `UInt256`, which is a 256-bit unsigned integer data type.
Its internal representation contains an array of 4 `uint64_t` values:

```c
typedef struct {
  uint64_t data[4];
} UInt256;
```

The elements of the `data` array are in order from least significant to
most significant. So, element 0 contains bits 0–63,
element 1 contains bits 64–127, etc.

The functions you must implement are declared in the header file `uint256.h`
and defined in the source file `uint256.c`.

## Implementing the functions, testing

Each function has a detailed comment describing the expected behavior.

A unit test program is provided.  Its code is in the source file
`uint256_tests.c`. You can compile the test program by running the
`make` command. So, the following commands will build the test program
and execute it:

```bash
make
./uint256_tests
```

Note that while the unit test program contains some basic tests that you should
find useful, they are by no means comprehensive. So, you should understand
that *you are expected to add your own tests.*  For Milestone 2, part of the
grading criteria includes the comprehensiveness of the tests you write.

## Hints and tips

**Addition**

To add two `UInt256` values, you should use the "grade school" algorithm
for addition. Treat the elements of the `data` arrays of the values
being added as the "columns" of the addition, starting with the elements
at index 0 (the least significant "column".) Each column of the sum is
the sum of the corresponding column values of the values being added.
*However*, you will need to handle the possibility that the sum of column
values could exceed the range of values that can be represented by
`uint64_t`, in which case you will need to carry a 1 into the next
(more significant) column.

You can determine if the addition of two `uint64_t` values overflows
as follows:

```c
uint64_t leftval = /* some value */,
         rightval = /* some value */,
         sum;

sum = leftval + rightval;
if (sum < leftval) {
  // the addition overflowed
}
```

Note that it is possible that the sum of two `UInt256` values could
overflow the range of values which can be represented. This will happen
if a 1 is carried out of the most significant column. You do not need
to do anything special to handle this possibility. So, for example,
if you add 1 to the maximum `UInt256` value that can be represented,
the result should be 0:

```c
UInt256 max;
for (int i = 0; i < 4; ++i) { max.data[i] = ~(0UL); }

UInt256 one = uint256_create_from_u64(1UL);

UInt256 sum = uint256_add(max, one);

// these assertions should succeed
ASSERT(sum.data[3] == 0UL);
ASSERT(sum.data[2] == 0UL);
ASSERT(sum.data[1] == 0UL);
ASSERT(sum.data[0] == 0UL);
```

**Subtraction**

You should implement subtraction as follows. Let's say that `uint256_sub`
needs to compute the difference $$a-b$$. It should implement this subtraction
as $$a + (-b)$$, where $$-b$$ is the two's complement negation of $$b$$.
Recall that the two's complement negation of a bitstring is found by
inverting all of its bits, and then adding 1.

As with addition, you do not need to do anything special to handle the
case where a difference would produce a value that can't be represented
(because it would be negative.)  So, for example, subtracting 1 from 0
should yield a difference that is equal to the maximum value that can
be represented.

**Multiplication**

Note that multiplication is the most complicated operation you will be
implementing, so you should probably implement `uint256_mul` last, after
getting all of the other functions to work.

To think about how multiplication should work, let's start by thinking about
how unsigned integers are represented.

For example, the integer 37 is represented in binary as 100101, because

$$37 = 1 \times 2^{5} + 0 \times 2^{4} + 0 \times 2^{3} + 1 \times 2^{2} + 0 \times 2^{1} + 1 \times 2^{0}$$

Let's say we want to compute the product $$37 \times m$$, where $$m$$ is some other integer value.
We do this by computing the sum of multiplying $$m$$ by the powers of 2 that make up 37:

$$37 \times m = (2^{5} \times m) + (2^{2} \times m) + (2^{0} \times m)$$

As we know, left shifting an integer's binary representation by $$n$$ bits
is equivalent to multiplying it by $$2^{n}$$.  This suggests
an algorithm for multiplying `UInt256` values $$a$$ and $$b$$:

* Start with the result set to 0
* For each bit in the binary representation of $$a$$ that is set to 1:
    * Compute a term by left shifting $$b$$ by the bit position of the bit in $$a$$
    * Update the result by adding the term to its current value

When the algorithm finishes, the result should be the product $$a \times b$$.

As with addition and subtraction, the product could be too large to represent,
but you do not need to do anything special to handle this case. You can
just allow the left-shift operations and additions to overflow.
The autograder tests will only test multiplications where the product can be
represented exactly.

You will probably want to implement helper functions for checking whether a
particular bit is set to 1, and for left-shifting a value by a specified
number of positions:

```c
int uint256_bit_is_set(UInt256 val, unsigned index);
UInt256 uint256_leftshift(UInt256 val, unsigned shift);
```

You can add declarations for these helper functions to `uint256.h`, and add
unit tests for them to `uint256_tests.c`.

You will need to think carefully about how to implement the left shift
operation. Drawing a diagram on paper will probably be helpful.
Note that when left-shifting a `uint64_t` value, shifting by 64 or more
positions is undefined behavior.  Also note that literal integer values are,
by default, 32 bit values which can't be shifted by more than 31 positions.
Adding the suffix "`UL`" on an integer literal will force its type to be
`unsigned long`, which is a 64 bit type in the version of `gcc` we are
using.  So, for example,

```c
// this shift is undefined behavior
uint64_t val1 = (1 << 33);

// this shift is well defined
uint64_t val2 = (1UL << 33);
```

**Conversion from hex**

TODO

**Conversion to hex**

TODO
