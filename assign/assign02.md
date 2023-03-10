---
layout: default
title: "Assignment 2: Hex dump"
---

Milestone 1: due Friday, Feb 17th by 11pm

Milestone 2: due <strike>Monday, Feb 27th</strike> Tuesday, Feb 28th by 11pm

Assignment type: **Pair**, you may work with one partner

*Update 2/9*: Updated example runs so that input is read from a file
rather than from a terminal.

# Overview

In this assignment you will implement a hex dump program using both C
and assembly language. The submission of this assignment will be broken
up to two milestones as listed below.

Note that your assembly-code *must* strictly adhere to all requirements
of the x86-64 Linux register use and procedure calling conventions.
You will receive a deduction for violations of the register use and
procedure calling conventions, *even if your code passes the
automated tests.*

**Acknowledgment:** The idea for this assignment comes from the [Fall
2018 HW5](https://www.cs.jhu.edu/~phf/2018/fall/cs229/simple-x86_64.html)
developed by Peter Froehlich.

## Milestone 1 requirements

For this submission, all C language function implementations must be
working with unit tests written. In addition, _at least_ the Assembly
language functions of `hex_to_printable` and `hex_format_byte_as_hex`
must be working with unit tests written.

To summarize this milestone, the `c_hexdump` program should be 100% functional, and
`c_hextests` should pass all of the implemented unit tests. (You will need
to write your own unit tests so that each of the functions declared
in `hexfuncs.h` is thoroughly tested.)

## Milestone 2 requirements

The rest of the Assembly language functions must be written with
thorough unit tests. Uploads for this submission should include the C
implementation and unit tests submitted for part 1 as well.

For this milestone, the `asm_hexdump` program should be 100% functional,
and `asm_hextests` should pass all of the implemented unit tests.
(You should not need to add any additional unit tests, although you may
if you wish.)

## Late Days

If you will be using more than 2 late days on this assignment (for
either milestone), please make a private post (to instructors) on Courselore
letting us know.

## Grading breakdown

**Milestone 1** (30 points)

* C implementation: 10%
* Assembly implementation: 20%

**Milestone 2** (70 points)

* Assembly implementation: 35%
* Unit tests: 20%
* Packaging, style, and design: 15%

## Getting started

Download [csf\_assign02.zip](csf_assign02.zip), which contains the skeleton code for the assignment.

You can download this file from a Linux command prompt using the `curl` command:

```bash
curl -O https://jhucsf.github.io/spring2023/assign/csf_assign02.zip
```

Note that in the `-O` option, it is the letter "O", not the numeral "0".

# Hex dump

Start by [reading up](http://en.wikipedia.org/wiki/Hex_dump) on what
hexdumps are. For this assignment, you will write a program in C and x86-64 assembly that produces a hexdump on standard output for data read from standard input.

Let???s start with an example:

```
$ echo "Hello" > input.txt
$ ./c_hexdump < input.txt 
00000000: 48 65 6c 6c 6f 0a                                Hello.
```

First, we echoed the text "Hello" followed by a newline (`\n`) character
to the file `input.txt`. Then, we invoked the `c_hexdump` program to read from
`input.txt`.  The result shows the *ASCII code* for each character
(in *hexadecimal*, so it???s guaranteed to be two digits wide for each
character), including the newline character. The formatting may look a
bit strange, but the purpose of the ???large gap??? becomes apparent if
we examine a longer input:

```
$ echo "This is a longer example of a hexdump. Marvel at its magnificence." > input.txt
$ ./c_hexdump < input.txt 
00000000: 54 68 69 73 20 69 73 20 61 20 6c 6f 6e 67 65 72  This is a longer
00000010: 20 65 78 61 6d 70 6c 65 20 6f 66 20 61 20 68 65   example of a he
00000020: 78 64 75 6d 70 2e 20 4d 61 72 76 65 6c 20 61 74  xdump. Marvel at
00000030: 20 69 74 73 20 6d 61 67 6e 69 66 69 63 65 6e 63   its magnificenc
00000040: 65 2e 0a                                         e..
```

This time we echoed a longer sequence of characters consisting of two
sentences (followed implicitly by a newline) to `input.txt`
and again invoked `c_hexdump` to read the data.
Again, we see the ASCII code for each character (including
spaces and newlines). The formatting is set up so that regardless of the
number of characters, we always have three ???columns??? of output:

1.  First the overall ???position??? in the input. Note that this is also a
    hexadecimal number, **formatted to 8 digits**.
2.  Then the ASCII values for each character in hexadecimal, at most 16
    to a line.
3.  Finally a "string-like" representation of the data, with printable
    characters shown but non-printable characters (like newline or tab)
    replaced with a dot.

Note that there???s a single space between the colon after the offset and
the ASCII values, but there are *two* spaces between the ASCII values
and the string-like representation.

The behavior of your program should be identical to the command `xxd -g
1`. Take note of how the program will only print a row if it either has
a full row of sixteen characters, or if CTRL-D is pressed.

Note that because the purpose of this assignment is to give
you an opportunity to learn how to write x86-64 assembly
language code, there are some very important [non-functional
requirements](#non-functional-requirements) that you will need to satisfy.
(Please read that section of the assignment description carefully.)

**Important**: For testing the functional correctness of your hexdump programs, it
is only important that it behave identically to `xxd -g 1` *when reading from
a file*, using input redirection.  So, you will want to test your hexdump
programs using a command like

```
./c_hexdump < myinput
```

where `myinput` is an input file you want to test. If the input to the
program is read from a terminal, then when the program calls `read`
there is a significant possibility that it could return fewer than
16 bytes of data, even the end of input has not yet been reached.

# Functional requirements

## Functions

The header file `hexfuncs.h` declares the following functions:

```c
// Read up to 16 bytes from standard input into data_buf.
// Returns the number of characters read.
unsigned hex_read(char data_buf[]);

// Write given nul-terminated string to standard output.
void hex_write_string(const char s[]);

// Format an unsigned value as an offset string consisting of exactly 8
// hex digits.  The formatted offset is stored in sbuf, which must
// have enough room for a string of length 8.
void hex_format_offset(unsigned offset, char sbuf[]);

// Format a byte value (in the range 0-255) as string consisting
// of two hex digits.  The string is stored in sbuf.
void hex_format_byte_as_hex(unsigned char byteval, char sbuf[]);

// Convert a byte value (in the range 0-255) to a printable character
// value.  If byteval is already a printable character, it is returned
// unmodified.  If byteval is not a printable character, then the
// ASCII code for '.' should be returned.
char hex_to_printable(unsigned char byteval);
```

In both your C and assembly language implementations, you are required
to implement these functions exactly as specified.

Per the [non-functional requirements](#non-functional-requirements),
the `hex_read` and `hex_write_string` functions must be implemented using
the `read` and `write` system calls, rather than using stdio functions.

## Main functions

In `c_hexmain.c` and `asm_hexmain.S`, you will develop C and
assembly-language `main` functions which call the functions shown above
in order to implement the functionality of the hexdump program.

Note that your main function (either version) may *only* call these
functions and (optionally) helper functions that you create.

The `c_hexdump` and `asm_hexdump` Makefile targets build executable
programs using these `main` modules.  When reading data from standard
input, their output should be identical to the command `xxd -g 1`.

The `casm_hexdump` Makefile target builds an executable program which
uses the C version of the `main` function, but the assembly-language
version of the hex functions.  This is a handy way to test your assembly
language function implementations before you have fully implemented
the assembly language version of the `main` function.  Its behavior
(reading from standard input) should also be identical to `xxd -g 1`.

## Unit tests

The source file `hextests.c` contains unit tests for the required
functions.  The provided version is very minimal, so you should add
additional tests so that your implementations of the functions are
thoroughly tested.  Part of your grade will be based on the thoroughness
of your unit tests.

Note that it will not be straightforward to write unit tests for the
`hex_read` and `hex_write_string` functions, since they do I/O.  So,
you are not required to write unit tests for them.

**Important advice**: Writing complete programs in
assembly language is challenging.  Using unit tests, you can adopt a
test-driven approach where you implement one assembly language
function at a time, and test each one to ensure correct operation.
Using this approach will make developing the `asm_hexdump` program vastly
easier.

## Program-level testing

In addition to unit testing individual functions, you should test the
program as a whole. In general, for any input file (text, binary, etc.),
the commands

```
./c_hexdump < inputfile
```

and


```
./asm_hexdump < inputfile
```

should produce exactly the same output as

```
xxd -g 1 < inputfile
```

We encourage you to test your program with a variety of inputs, including (but not limited to):

* empty file
* small files
* large files
* files with sizes that are a multiple of 16
* files with sizes that aren't a multiple of 16
* text files
* binary files

# Non-functional requirements

Calling C library functions is *not* allowed, with the following exceptions:

1. `c_hexfuncs.c` may `#include <unistd.h>` and call the `read`
   and `write` functions (which are wrappers for the `read` and `write`
   system calls).
2. `asm_hexfuncs.S` may call `read` and `write`
3. Your unit tests (in `hextests.c`) may call any C library functions
   (for example, you could use `strcmp` to verify that the
   `hex_format_offset` function generated the correct string value)

Outside of these exceptions, any call to C
library functions will result in an **automatic zero** on the entire
assignment. Please don't do it!

All assembly language code must be 100% written by hand and extensively
commented. No credit will be given otherwise. Any undocumented assembly
code **will** cause you to lose points. You don???t have to comment
**every** line, but at least every ???coherent chunk??? of assembly
should have a comment or two. In particular you **must** describe where
you get what data from, especially when it comes to functions and their
parameters/results. **You have been warned\!**

# Hints and tips

## Adding "stub" functions

As distributed, the starter code for the assignment doesn't include
any function definitions in the `asm_hexfuncs.S` or `c_hexfuncs.c`
source files. Before the programs depending on them will compile and
link successfully, you will need to add "stub" implementations of
all of the functions.

For example, the C stub function for `hex_format_offset`:

```c
void hex_format_offset(unsigned offset, char sbuf[]) {
  // TODO: implement
}
```

The assembly language stub function for `hex_format_offset`:

```
	.globl hex_format_offset
hex_format_offset:
	/* TODO: implement */
	ret
```

Make sure that all of the assembly language functions are in the
`.text` section.


## x86-64 tips and tricks

Here are some specific tips and tricks in no particular order.

Don't forget that you need to prefix constant values with `$`.  For example,
if you want to set register `%r10` to 16, the instruction is

```
movq $16, %r10
```

and not

```
movq 16, %r10
```

If you want to use a label as a pointer (address), prefix it with
`$`.  For example,

```
movq $sHexDigits, %r10
```

would put the address that `sHexDigits` refers to in `%r10`.

When calling a function, the stack pointer (`%rsp`) must contain an address
which is a multiple of 16.  However, because the `callq` instruction
pushes an 8 byte return address on the stack, on entry to a function,
the stack pointer will be "off" by 8 bytes.  You can subtract 8 from
`%rsp` when a function begins and add 8 bytes to `%rsp` before returning
to compensate.  (See the example `addLongs` function.)  Pushing an
odd number of callee-saved registers also works, and has the benefit
that you can then use the callee-saved registers freely in your function.

If you want to define read-only string constants, the `.rodata` section
is the right place for them.  For example:

```
        .section .rodata
sHexDigits: .string "0123456789abcdef"
```

The `.equ` assembler directive is useful for defining constant values,
for example:

```
.equ BUFSIZE, 16
```

You might find the following source code comment useful for reminding
yourself about calling conventions:

```
/*
 * Notes:
 * Callee-saved registers: rbx, rbp, r12-r15
 * Subroutine arguments:  rdi, rsi, rdx, rcx, r8, r9
 */
```

In Unix and Linux, standard input is file descriptor 0.

The GNU assembler allows you to define "local" labels, which start
with the prefix `.L`.  You should use these for control flow targets
within a function.  For example (from the [echoInput.S](assign02/echoInput.S)
example program):

```
	cmpq $0, %rax                 /* see if read failed */
	jl .LreadError                /* handle read failure */

	...

.LreadError:
	/* error handling goes here */

```

**Hint about determining which characters are printable**: the range of
printable ASCII characters is 32 through 126, inclusive.  Any byte value
that is not in this range should be printed as "`.`" (period).  Note
that "`.`" has ASCII value 46.

## Example assembly language functions

This section shows implementations of a couple of assembly language functions
you might find useful.

Here is an assembly language function called `strLen` which returns the number
of characters in a NUL-terminated character string:

```
/*
 * Determine the length of specified character string.
 *
 * Parameters:
 *   s - pointer to a NUL-terminated character string
 *
 * Returns:
 *    number of characters in the string
 */
	.globl strLen
strLen:
	subq $8, %rsp                 /* adjust stack pointer */
	movq $0, %r10                 /* initial count is 0 */

.LstrLenLoop:
	cmpb $0, (%rdi)               /* found NUL terminator? */
	jz .LstrLenDone               /* if so, done */
	inc %r10                      /* increment count */
	inc %rdi                      /* advance to next character */
	jmp .LstrLenLoop              /* continue loop */

.LstrLenDone:
	movq %r10, %rax               /* return count */
	addq $8, %rsp                 /* restore stack pointer */
	ret
```

In C, the declaration of this function could look like this:

```c
long strLen(const char *s);
```

Unit testing this function might involve the following assertions:

```c
ASSERT(13L == strLen("Hello, world!"));
ASSERT(0L == strLen(""));
ASSERT(8L == strLen("00000010"));
```

## Assignment tips

For this assignment, you should start by writing the C language
implementations first. Be sure to write unit tests and check your work
along the way.

After writing your C functions, start thinking about how you can translate
your implementation into assembly. Be sure to consider register usage
and be aware of pushing/popping the stack.

# Submitting

Before you submit, prepare a `README.txt` file so that it contains your
names, and briefly summarizes each of your contributions to the submission
(i.e., who worked on what functionality.) This may be very brief if you
did not work with a partner.

Submit a zipfile containing your complete project.  The recommended
way to do this is to run the command `make solution.zip`.  This
will create a file called `solution.zip` with all of the required
files.  **Important**: all of the files in the zipfile must be
at the top level, not a subdirectory.  For example, if your
zipfile is called `solution.zip` and you run the command `unzip -l solution.zip`
to list its contents, you should see something like the following output:

```
Archive:  solution.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
     1713  2022-02-06 13:55   Makefile
     1079  2022-02-06 13:36   hexfuncs.h
     3989  2022-02-06 13:48   tctest.h
     1063  2022-02-06 13:40   c_hexfuncs.c
      895  2022-02-06 13:36   c_hexmain.c
     1458  2022-02-06 13:36   hextests.c
     3948  2022-02-06 13:36   tctest.c
     3459  2022-02-06 13:36   asm_hexfuncs.S
     4150  2022-02-06 13:36   asm_hexmain.S
---------                     -------
    21754                     9 files
```

Upload this zipfile to Gradescope for both parts 1 and 2 of Assignment 2.
Make sure to include your name and email address in *every* file you
turn in (well, in every file for which it makes sense to do so anyway!)
