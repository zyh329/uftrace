uftrace
=======

The uftrace tool is to trace and analyze execution of a program
written in C/C++.  It was heavily inspired by the ftrace framework
of the Linux kernel (especially function graph tracer) and supports
userspace programs.  It support various kind of commands and filters
to help analysis of the program execution and performance.


Features
========

It traces each function in the executable and shows time duration.  It
can also trace external library calls - but only entry and exit are
supported and cannot trace internal function calls in the library call
unless the library itself built with profiling enabled.

It can show detailed execution flow at function level, and report which
function has the highest overhead.  And it also shows various information
related the execution environment.

You can setup filters to exclude or include specific functions when tracing.
In addition, it can save and show function arguments and return value.

It supports multi-process and/or multi-threaded applications.  With root
privilege, it can also trace kernel functions as well( with `-k` option)
if the system enables the function graph tracer in the kernel
(`CONFIG_FUNCTION_GRAPH_TRACER=y`).


How to use uftrace
==================
The uftrace command has following subcommands:

 * `record` : runs a program and saves the trace data
 * `replay` : shows program execution in the trace data
 * `report` : shows performance statistics in the trace data
 * `live` : does record and replay in a row (default)
 * `info` : shows system and program info in the trace data
 * `dump` : shows low-level trace data
 * `recv` : saves the trace data from network
 * `graph` : shows function call graph in the trace data

    $ uftrace
    Usage: uftrace [OPTION...] [record|replay|live|report|info|dump|recv|graph] [<command> args...]
    Try `uftrace --help' or `uftrace --usage' for more information.

If omitted, it defaults to the `live` command which is almost same as running
record and replay subcommand in a row (but does not record the trace info
to files).

For recording, the executable should be compiled with `-pg`
(or `-finstrument-functions`) option which generates profiling code
(calling mcount or __cyg_profile_func_enter/exit) for each function.

    $ uftrace tests/t-abc
    # DURATION    TID     FUNCTION
      16.134 us [ 1892] | __monstartup();
     223.736 us [ 1892] | __cxa_atexit();
                [ 1892] | main() {
                [ 1892] |   a() {
                [ 1892] |     b() {
                [ 1892] |       c() {
       2.579 us [ 1892] |         getpid();
       3.739 us [ 1892] |       } /* c */
       4.376 us [ 1892] |     } /* b */
       4.962 us [ 1892] |   } /* a */
       5.769 us [ 1892] | } /* main */

For more analysis, you'd be better recording it first so that it can run
analysis commands like replay, report, graph, dump and/or info multiple times.

    $ uftrace record tests/t-abc

It'll create uftrace.data directory that contains trace data files.
Other analysis commands expect the directory exists in the current directory,
but one can use another using `-d` option.

The `replay` command shows execution information like above.  As you can see,
the t-abc is a very simple program merely calls a, b and c functions.
In the c function it called getpid() which is a library function implemented
in the C library (glibc) on normal systems - the same goes to __cxa_atexit().

Users can use various filter options to limit functions it records/prints.
The depth filter (`-D` option) is to omit functions under the given call depth.
The time filter (`-t` option) is to omit functions running less than the given
time. And the function filters (`-F` and `-N` options) are to show/hide functions
under the given function.

The `report` command lets you know which function spends the longest time
including its children (total time).

    $ uftrace report
      Total time   Self time  Nr. called  Function
      ==========  ==========  ==========  ====================================
        2.723 us    0.337 us           1  main
        2.386 us    0.330 us           1  a
        2.056 us    0.366 us           1  b
        1.690 us    0.927 us           1  c
        1.277 us    1.277 us           1  __monstartup
        0.936 us    0.936 us           1  __cxa_atexit
        0.763 us    0.763 us           1  getpid


The `graph` command shows function call graph of given function.  In the above
example, function graph of function 'a' looks like below:

    $ uftrace graph  a
    # 
    # function graph for 'a'
    # 
    
    backtrace
    ================================
     backtrace #0: hit 1, time  19.334 us
       [0] main (0x4004f0)
       [1] a (0x40069f)
    
    calling functions
    ================================
      19.334 us : (1) a
      16.667 us : (1) b
      15.001 us : (1) c
       5.333 us : (1) getpid


The `dump` command shows raw output of each trace record.  You can see the result
in the chrome browser, once the data is processed with `uftrace dump --chrome`.

The `info` command shows system and program information when recorded.

    $ uftrace info
    # system information
    # ==================
    # program version     : uftrace v0.6
    # recorded on         : Tue May 24 11:21:59 2016
    # cmdline             : uftrace record tests/t-abc 
    # cpu info            : Intel(R) Core(TM) i7-3930K CPU @ 3.20GHz
    # number of cpus      : 12 / 12 (online / possible)
    # memory info         : 20.1 / 23.5 GB (free / total)
    # system load         : 0.00 / 0.06 / 0.06 (1 / 5 / 15 min)
    # kernel version      : Linux 4.5.4-1-ARCH
    # hostname            : sejong
    # distro              : "Arch Linux"
    #
    # process information
    # ===================
    # number of tasks     : 1
    # task list           : 5098
    # exe image           : /home/namhyung/project/ftrace/tests/t-abc
    # build id            : a3c50d25f7dd98dab68e94ef0f215edb06e98434
    # exit status         : exited with code: 0
    # cpu time            : 0.000 / 0.003 sec (sys / user)
    # context switch      : 1 / 1 (voluntary / involuntary)
    # max rss             : 3072 KB
    # page fault          : 0 / 172 (major / minor)
    # disk iops           : 0 / 24 (read / write)


How to install uftrace
======================

The uftrace is written in C and tried to minimize external dependencies.
Currently it requires `libelf` in elfutils package to build, and there're some
more optional dependencies.

Once you installed required software(s) on your system, it can be built and
installed like following:

    $ make
    $ sudo make install

For more advanced setup, please refer
[INSTALL.md](https://github.com/namhyung/uftrace/blob/master/INSTALL.md) file.


Limitations
===========
- It can only trace a native C/C++ application compiled with -pg option.
- It *cannot* trace already running process.
- It *cannot* be used for system-wide tracing.
- It only supports x86_64 and ARM (v6,7) for now.


License
=======
The uftrace program is released under GPL v2.  See COPYING file for details.
