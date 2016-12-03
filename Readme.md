Rich man's profiler
===================

This is a fork of https://github.com/ygrek/ocaml-gdb, a gdb-based profiler for
native OCaml programs. Support for writing cpuprofile files has been added,
which can be viewed in Chromium's dev tools (press F12). It also supports
writing callgrind files, which can be viewed using
[https://kcachegrind.github.io/html/Home.html](kachegrind).

Quick start
-----------

RMP doesn't require instrumentation. Compile your code to native binaries. Optionally, add
`-g` in ocamlc or `-tag debug` in ocamlbuild to see source code locations.

Install dependencies: `opam install menhir extlib lwt ppx_deriving ppx_deriving_yojson ppx_tools oasis cppo containers`

Installation: `git clone https://github.com/copy/rmp && cd rmp && make`

Usage: `rmp.native -p <pid> --cpuprofile <path> --callgrind <path>`

Example:  ```rmp.native -p `pidof my_example_program.native` --cpuprofile example.cpuprofile --callgrind callgrind.out```

If you're getting a "not permitted" error, run the following:
`su -c 'sysctlkernel.yama.ptrace_scope=0'`
([https://rajeeshknambiar.wordpress.com/2015/07/16/attaching-debugger-and-ptrace_scope/](more
infos))

The output file must have a `.cpuprofile` extension, otherwise Chromium refuses to load it.

<hr>

![Bottom Up](https://i.imgur.com/smIR1tZ.png)

Bottom up view in Chromium.

<hr>

![Top Down](https://i.imgur.com/3EA26XM.png)

Top down view in Chromium.

<hr>

![Chart](https://i.imgur.com/8QEV98Y.png)

Chart view in Chromium.

<hr>

![Tree Map](https://i.imgur.com/2d4dvi7.png)

[Tree Map](https://kcachegrind.github.io/html/TreeMap.html) in kcachegrind.

<hr>

![Call Graph](https://i.imgur.com/pr7IDdH.png)

[Call Graph](https://kcachegrind.github.io/html/CallGraph.html) in kcachegrind.


Introduction
------------

RMP is a stack sampling profiler based on ocaml-gdb library for communication
with GDB machine interface (GDB/MI).

Stack sampling profiler works by taking many stack traces of the profiled process
and aggregating similar traces together, providing empirical evidence of program
behaviour with statistical estimation of percentage of time spent in different code paths.
This means, that profiler will show not only cpu-bound hotspots, but also
all places where the code is waiting for the kernel call to finish (e.g. sleep, blocking IO, etc).
One nice property of stack sampling approach is that the profiled program needn't be modified
or restarted - profiler only needs the permission to attach to the running process and collect
stack traces for the short period of time.

See http://poormansprofiler.org/ for general description of this technique.

Rationale
---------

RMP attaches GDB to the profiled program once and then instructs it to probe the stack of the
inferior process periodically. The traces are aggregated and displayed/updated immediately. It has access
to all the information that GDB provides (static stack unwinding, debug symbols, etc). Interactive mode
allows to stop/resume profiling at any time (e.g. when enough information is available or waiting for
the process to engage into some specific activity).

RMP was devised to fill the gap between existing solutions :

* gprof

  Instrumenting profiler, requires recompilation, instrumentation influences code behaviour significantly
  (inflating execution time for small functions).

* pmp

  Poor man's profiler shell script works by calling gdb several times each time attaching to the process,
  collecting trace and detaching. This extremely simple approach works everywhere and yields tons of
  useful information. Unfortunately, it also incurs significant pauses in the profiled process and takes
  quite a bit of time to collect the traces, also doesn't have interactive mode.

* linux perf

  Works only on linux, needs to be compiled with the matching version of kernel headers. Can profile both
  kernel and userland, but only the code that saves frame pointer on the stack. This is true for the majority
  of C code, but doesn't hold for the OCaml native code (in default mode).
