---
title: 'WebAssembly Tail Calls'
description: 'This document explains the WebAssembly Tail Calls proposal and
demonstrates it with some examples'
author: 'Thibaud Michaud, Thomas Lively'
date: 2023-04-04
tags:
  - WebAssembly
---
We are shipping WebAssembly Tail Calls in V8 11.2! In this post we give a brief overview of this proposal, demonstrate an interesting use case for C++ coroutines with Emscripten, and show how V8 handles tail calls internally.


## What is Tail Call Optimization

A call is said to be in tail position if it is the last instruction executed before returning from the current function. Compilers usually optimize such calls by discarding the caller frame and replacing the call with a jump.

This is especially useful for recursive functions. For instance, take this C function that sums the elements of a linked list:

```c
int sum(List* list, int acc) {
  if (list == nullptr) return acc;
  return sum(list->next, acc + list->val);
}
```

With a regular call, this consumes O(n) stack space: each element of the list adds a new frame on the call stack. With a long enough list, this could very quickly overflow the stack. By replacing the call with a jump, tail-call optimization effectively turns this recursive function into a loop which uses
O(1) stack space:

```c
int sum(List* list, int acc) {
  while (list != nullptr) {
    acc = acc + list->val;
    list = list->next;
  }
}
```

This optimization is particularly important for functional languages. They rely heavily on recursive functions, and pure ones like Haskell don’t even provide loop control structures. Any kind of custom iteration typically uses recursion one way or another. Without tail-call optimization, this would very quickly run into a stack overflow for any non-trivial program.

### The WebAssembly Tail Call Proposal

There are two ways to call a function in Wasm MVP: `call` and `call_indirect`.  This proposal adds their tail-call counterparts: `return_call` and `return_call_indirect`. This means that it is the responsibility of the toolchain to actually perform tail-call optimization and emit the appropriate call kind, which gives it more control over performance and stack space usage.

Let’s look at a recursive fibonacci function. The Wasm bytecode is included here in the text format for completeness, but you can find it in C++ in the next section:

<pre>
(func $fib_rec (param $n i32) (param $a i32) (param $b i32) (result i32)
  (if (i32.eqz (local.get $n))
    (then (return (local.get $a)))
    (else
      (<b>return_call</b> $fib_rec
        (i32.sub (local.get $n) (i32.const 1))
        (local_get $b)
        (i32.add (local.get $a) (local.get $b))
      )
    )
  )
)

(func $fib (param $n i32) (result i32)
  (call $fib_rec (local.get $n) (i32.const 0) (i32.const 1))
)
</pre>

At any given time there is only one `fib_rec` frame, which will unwind itself before performing the next recursive call. When we reach the base case, `fib_rec` returns the result `a` directly to `fib`.

This has one observable consequence (besides a reduced risk of stack overflow): the tail-callers will not appear in stack traces. Neither will they appear in the stack property of a caught exception, nor in the DevTools stack trace. By the time an exception is thrown, or execution pauses, the tail-caller frames are gone and there is no way for V8 to recover them.

## Using tail calls with Emscripten

Functional languages often depend on tail calls, but it's possible to use them as a C or C++ programmer as well. Emscripten (and Clang, which Emscripten uses) supports the musttail attribute that tells the compiler that a call must be compiled into a tail call. As an example, consider this recursive implementation of a Fibonacci function that calculates the nth Fibonacci number mod 2^32 (because the integers will overflow for large n):

```c
#include <stdio.h>

unsigned fib_rec(unsigned n, unsigned a, unsigned b) {
  if (n == 0) {
    return a;
  }
  return fib_rec(n - 1, b, a + b);
}

unsigned fib(unsigned n) {
  return fib_rec(n, 0, 1);
}

int main() {
  for (unsigned i = 0; i < 10; i++) {
    printf("fib(%d): %d\n", i, fib(i));
  }

  printf("fib(1000000): %d\n", fib(1000000));
}
```

After compiling with `emcc test.c -o test.js`, running this program in Node gives a stack overflow error. We can fix this by adding `__attribute__((__musttail__))` to the return in `fib_rec` and adding `-mtail-call` to the compilation arguments. Now the produced Wasm modules contains the new tail call instructions, so we have to pass `--experimental-wasm-return_call` to Node, but the stack no longer overflows.

Here's an example using mutual recursion as well:

```c
#include <stdio.h>
#include <stdbool.h>

bool is_odd(unsigned n);
bool is_even(unsigned n);

bool is_odd(unsigned n) {
  if (n == 0) {
    return false;
  }
  __attribute__((__musttail__))
  return is_even(n - 1);
}

bool is_even(unsigned n) {
  if (n == 0) {
    return true;
  }
  __attribute__((__musttail__))
  return is_odd(n - 1);
}

int main() {
  printf("is_even(1000000): %d\n", is_even(1000000));
}
```

Note that both of these examples are simple enough that if we compile with -O2, the compiler can precompute the answer and avoid exhausting the stack even without tail calls, but this wouldn't be the case with more complex code. In real-world code, the musttail attribute can be helpful for writing high-performance interpreter loops as described in this blog post:
<https://blog.reverberate.org/2021/04/21/musttail-efficient-interpreters.html>.

Besides the `musttail` attribute, C++ depends on tail calls for one other feature: C++20 coroutines. The relationship between tail calls and C++20 coroutines is covered in extreme depth in this blog post: https://lewissbaker.github.io/2020/05/11/understanding_symmetric_transfer, but to summarize, it is possible to use coroutines in a pattern that would subtly cause stack overflow even though the source code doesn't make it look like there is a problem. To fix this problem, the C++ committee added a requirement that compilers implement "symmetric transfer" to avoid the stack overflow, which in practice means using tail calls under the covers.

When WebAssembly tail calls are enabled, Clang implements symmetric transfer as described in that blog post, but when tail calls are not enabled, Clang silently compiles the code without symmetric transfer, which could lead to stack overflows and is technically not a correct implementation of C++20!

To see the difference in action, use Emscripten to compile the last example from the blog post linked above and observe that it only avoids overflowing the stack if tail calls are enabled. Note that due to a recently-fixed bug, this will only work correctly in Emscripten 3.1.35 or later.

## Tail Calls in TurboFan

As we saw earlier, it is not the engine’s responsibility to detect calls in tail position. This should be done upstream by the toolchain. So the only thing left to do is to emit an appropriate sequence of instructions based on the call kind and the target function signature.  For our fibonacci example from earlier, the stack would look like this:

![](/_img/tail-call.png)

On the left we are inside `fib_rec` (green), called by `fib` (blue) and about to recursively tail-call `fib_rec`. First we unwind the current frame by resetting the frame and stack pointer. The frame pointer just restores its previous value by reading it from the “Caller FP” slot. The stack pointer moves to the top of the parent frame, plus enough space for any potential stack parameters and stack return values for the callee (0 in this case, everything is passed by registers). Parameters are moved into their expected registers according to fib_rec’s linkage (not shown in the diagram). And finally we start running `fib_rec`, which will start by creating a new frame.

`fib_rec` will unwind and rewind itself like this until n == 0, at which point it will return `a` by register to `fib`.

This is a simple case where all parameters and return values fit into registers, and the callee has the same signature as the caller. In the general case, we might need to do complex stack manipulations:
- Read outgoing parameters from the old frame
- Move parameters into the new frame
- Adjust the frame size by moving the return address up or down, depending on the number of stack parameters in the callee

All these reads and writes can conflict with each other, because we are reusing the same stack space. This is a crucial difference with a non-tail call, which would simply push all the stack parameters and the return address on top of the stack.

These stack and register manipulations are handled by the “gap resolver”: a component of TurboFan which takes a list of moves that should semantically be executed in parallel, and generates the appropriate sequence of moves to resolve potential interferences between the move’s sources and destinations. If the conflicts are acyclic, this is just a matter of reordering the moves such that all sources are read before they are overwritten. For cyclic conflicts (e.g. swapping two stack parameters), this can involve moving one of the sources to a temporary register or to a temporary stack slot to break the cycle.
