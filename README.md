# Doing Things with Assembly Language and NASM

This is just a list of short assembly language programs that I used to
reboot my assembly language skills, this time in X86 and X86_64.  ("This
time" because the last time I wrote assembly language I was writing for
the Motorola 68000 line.)

The tutorial I based this off of is at http://asmtutor.com/ The source
code for the original tutorial, as well as the website, is by GitHub
contributor Daniel Givney at
[Assembly Tutorials](https://github.com/DGivney/assemblytutorials).

I'll see if I can't scrape together some other, more esoteric examples
in the future.

## Getting Started

There's a Makefile.  It has a nice help¹.

You will need to be running Linux on an Intel platform.  These lessons
do not apply to ARM chips like those on the Raspberry Pi (although it
would be super cool if they did!).

You will need a copy of [nasm](https://www.nasm.us/), the Netwide
Assembler, the most popular assembler currently in widespread use.
There are other assemblers, such as GAS (Used by the GNU GCC project),
MASM (from Microsoft), and so forth, but NASM is popular,
well-understood, and well-supported.  You will also need a linker; the
Makefile assumes you have the linker suppled with GNU Binutils.  On
Ubuntu-based platforms this comes with the `build-essentials` package.
If you have a different distribution, consult your archive.  If you can
compile a **C** program, you're fine.

## Lesson 2

There is no Lesson 1.  Okay, there *is*, but I didn't do it.  While I
was looking around for tutorials I found a couple that taught different
things, and one of the things they all agreed on was a proper exit
command.  Since all Lesson 2 does is add that command, that's what I
did.

I also used a few NASM features not in the ASM Tutorial.  The `%define`
Nasm preprocessor allows you to provide named constants, and I've used
them here.

The syntax `equ $-msg` basically means "The address from HERE, the first
byte of this named data segment, minus the address named," which puts
into `len` the length of the string.  It only works because `len` is the
immediate next data segment.

### Differences between the 32 and 64 bit versions.

The biggest difference that I see is that the Syscalls have all be
redefined.  "Write" and "exit" were 4 & 1 in 32-bit Linux, but 1 & 60 int
64-bit, respectively.  The ASM Tutorial was 32-bit only, and used the
first four registers.  When I ported it to the 64-bit version, the
syscall for `write()` uses different registers.

The 32 bit version uses `int 80h` to interrupt the kernel.  The 64 bit
uses `syscall`.  The
[Linux System Call Table](http://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/)
is handy here.

### Lessons

So far, the assembly language programs have two `sections`: one for
constant data, the other for the actual program.  Before either section
there are macros and directives.  Right now the only macros I'm using
define constants.

We aren't allocating any memory that's not in a `.data` segment.  And
that's okay.  Everything is happening inside registers.  The CPU has 16
of them.  Some of them have side-effects and optimizations, and others
are *required* for some operations.  The AX register, for example, used
to be the destination for mathematical operations.  The X86_64 CPU
architecture is built around stack-based operations, and the command
`push reg` will push a value (either a register or memory contents) onto
the stack pointed to by the SP and BP registers, *and then increment
those registers*.  So, you know, there are quirks to memorize.

The Makefile contains compiling and linking instructions.  They're
different for 32 and 64 bit programs, and learning those differences
would be useful if you intend to write a lot of assembly language.

## Lesson 3

Lesson 3 is a lot like lesson 2, only instead of knowing the length of
the string, we're going to calculate it, using the NULL value as our
end-of-string marker.  This also introduces comparison and jump
commands!

The question embedded in my comment in the source file is legitimate.
At the time, I didn't know if `sub` sets things like the "is zero" flag
when two values are the same value, the way `cmp` does.  The
[Intel X86 Manual](https://software.intel.com/sites/default/files/managed/39/c5/325462-sdm-vol-1-2abcd-3abcd.pdf)
(**Warning**: PDF, and very big!) doesn't say they do, and the contents
of those flags should probably not be regarded as robust or reliable
after a `sub` operation.

With the 64-bit version, rather than blindly copy the ax/bx/cx/dx
sequence of registers, I deliberately chose to use `RSI` (the Source
Index Register) for my data source.  While the first eight registers are
considered "general purpose," RSI is (somewhat) optimized to read data
out of memory and its use is a signal to the CPU's predictive cache.  I
don't know if that's any use to me yet, but it's something I'm aware of
and I might someday have a use for it.

### Memory addressing syntax

Lesson three also introduces the `cmp byte [rax], 0` syntax, which does
a few things.  First, there are a *crazy* number of opcodes for the X86
architecture, and `cmp` is only one-half.  An opcode is the numeric
representation of an instruction to the chip; it's bit sequence
literally instructs which nanoscopic wires in the chip to light up to
perform an operation.  Not including the wild stuff, an Intel chip has
something like 1,900 opcodes.  But you'll only need to know about 20 of
them.

The `[rax]` syntax tells nasm to generate the `cmp` opcode for which the
first operand is an address in memory; `cmp` will fetch the thing at
that address first before doing the comparison.  (I'm not sure if this
occupies another register or what.  The manual doesn't say!)  The `byte`
command says that the comparison is on a byte-by-byte basis, so that's a
*different* opcode, but I suspect nasm makes it easy to remember which
is which with mnemonics.  You don't need to know different ASM commands
for "compare two registers," "compare a memory location with a
register," and "compare a memory location with a constant," because
nasm's syntax makes it easy to understand those operations.

What I do know is that the one thing you *can't* do is compare two
memory locations directly.  `cmp` works with two registers, or a
register and a memory location, or a register and a constant, but no
other combination.

## Lesson 4: Subroutines

Lesson four introduces two new pairs of instructions: `push` & `pop`,
and `call` and `ret`.  The first two push values onto the stack and
then pop them off. The latter two call a subroutine and then return
from it; `call` pushes the address of the next instruction onto the
stack, and `ret` pops it off and sets the IPR (Instruction Pointer
Register) to the calling routine.

In these examples, I think I've engaged in what is known as *callee
cleanup*, which means that the subroutine has the responsibility for
restoring the registers after using them.  Then again, I may be
hopelessly confused.  Hopefully, future lessons will clear up the
`cdecl()` and other assembly conventions.

As is clear in
[the commit](https://github.com/elfsternberg/asmtutorials/commit/89b58186fbc54508891c0077cc3e32b3fed8d7cb)
and in the comments itself, I've hopelessly abused convention by storing
the results in the EDX and RDX registers, rather than EAX as is the
convention.  On the one hand this is definitely *unstylish ASM*, on the
other hand it's something one can do in hand-written ASM, saving exactly
one cycle (register copies are *cheap*, people) on my computer that
(checks `lshw`) executes approximately 2,870,000 instructions per
**second**.

## Lesson 5: Includes

Lesson 5 takes the functions we wrote in Lesson 4 and moves them into
their own file, so that they can be called multiple time.  This means
that the "register abuse" I engaged in in Lesson 4 has to be backed out;
I have to be "good" and use the registers as recommended by the
textbooks, because now they'll have multiple users and the conventions
must be honored in that case.

## Lesson 7 & 8: Print-with-linefeed and Argv

Lesson 6 is virtually indistinguishable from Lessons 5 and 7; it's a
tiny jump to using null instead of LF as our terminator, and I was
already doing that.  Lesson 7 creates a wrapper around `puts()` that
automatically appends a line-feed to the end of your null-terminated
string.

This leads into lesson 8, in which the environment provides a new chunk
of memory containing the strings with which the program was initialized,
and pointers to those strings are placed on the stack.  The first value
on the stack is the number of pointers.

With the "add a line feed" wrapper, the original text has you putting
your line-feed string data into the stack, but I cheaped out and made my
line-feed a two-byte (LF + NULL) constant and referred to it by address
instead.

One thing I did learn here?  When I ported it to X86_64, it broke
badly.  It turns out that `syscall`, unlike `int 80h`, clobbers the
counter register `rcx`.  And since that's what we were using in the
32-bit version as our argv counter, I preserved that semantic in the
64-bit version, which also means I had to modify `putslf()` to push
`rcx` onto the stack and pop it off afterward.

### Sidebar: A bug!

Early on in Lesson 4, I spotted and fixed a bug where I had one too many
`pops` off the stack (see
[commit 89b58186](https://github.com/elfsternberg/asmtutorials/commit/89b58186fbc54508891c0077cc3e32b3fed8d7cb#diff-89abeb42c81885d8d2e202657820501bL58)),
but what perplexed me is how the system didn't crash with a stack
underflow.  Now I know why: the stack had two values on it already: the
counter, and the pointer to the program name, which is always `argv[0]`.
Kinda cool to realize that now.

More to come... maybe

## Authors

Yours truly!  Elf M. Sternberg <elf.sternberg@gmail.com>.

## License

Daniel Givney does not specify a license for his code, but it is his
copyright.  I did type in, modify, and write these examples on my own (I
find that I only *learn* things in my brain if they go through my
fingers, so I rarely cut-and-paste anything), and unless Daniel has a
complaint, I'm tagging my code with the MIT License.  See the
`LICENSE.txt` file for the full details.

## Acknowledgements

* Daniel Givney, of course.
* [The NASM Documentation](https://www.nasm.us/doc/) is very well-written!
* [Nayuki](https://github.com/nayuki) has added much to my understanding
* [David Evans](http://www.cs.virginia.edu/~evans/cs216/guides/x86.html)
helped with my understanding of syntax and register use.
* [Ray Toal](http://cs.lmu.edu/~ray/notes/nasmtutorial/)'s notes on NASM
are also useful.

---
Footnotes!

¹ I firmly believe that no command, typed blindy, should modify the
contents of your hard drive.  `Make` takes target arguments, and you
should specify the targets you want built.  So `make` by itself only
issues help.
