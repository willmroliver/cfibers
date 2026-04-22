# cfibers

Cooperative 'fiber' context-switching and management in C and x86-64 assembly.

Fibers are lightweight coroutines with explicit yielding. Spindles manage a group of fibers with a simple round-robin scheduling approach. Spools are the synchronisation primitive, on top of which a promise-based architecture is implemented

The context-switch itself is implemented in x86 NASM, and relies on the C ABI to do save a few instructions. No `setjmp`/`longjmp` instructions used, purely stack manipulation.

## Features

### Fiber hijacking

`fiber_hijack()` takes the current call stack — usually `main()` — and wraps it into a fiber. Once hijacked, the main thread becomes something the scheduler can yield and resume cooperatively, so the package can be used without overriding the standard main function.

This is achieved by inverting the yield/run semantics on the hijacked fiber:

```
yield(hijacker) == run(parent)
run(hijacker)   == yield(parent)
```

One limitation is that `fiber_run()` on a hijacker can only be called from within the parent's job — this isn't inherently an issue if a full-blooded scheduling algorithm were supported in the parent fiber.

### Lock-free spools

Spools are the synchronisation primitive. Each one is a 64-byte cache-line-aligned struct so they don't false-share. `spool_wait()` tries to acquire with `lock cmpxchg`: if the spool is taken, it yields the calling fiber and retries on the next scheduling pass, achieving blocking without spinning.

Spools also support atomic inc/dec, facilitating wait-group counting with promises.

## Building

Needs `nasm` and a C compiler. Works on macOS and Linux.

```
make            # builds lib.dylib (macOS) or lib.so (Linux)
make DEBUG=1    # with -g
make clean
```

## Tests

```
make
build/test/fiber
build/test/spool
build/test/promise
build/test/spindle
build/test/ds/ring
build/test/ds/dl_list
```

Covers fiber lifecycle, hijacking, recurring jobs, spool contention, promise resolution, and the full scheduler loop.
