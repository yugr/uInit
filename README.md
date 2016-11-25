# What is this?

uInit is a script and method for obtaining stable benchmarks results
on modern Linux distros.

Modern distributions (e.g. Ubuntu or Debian) have a lot of
running daemons or services which eat precious CPU time and pollute caches
(this is collectively called "OS jitter").
If that's not enough, hyperthreading and dynamic frequency scaling (DVFS) also add to the jitter
so in practice you can see up to 5% noise in benchmark runs (e.g. SPEC2000)
which prevents reliable performance comparisons.

To obtain more or less stable measurements (std. deviation less than 0.5%), you'll need to
* disable non-deterministic hardware features in BIOS e.g. hyperthreading and frequency scaling (also "power save mode", "turbo mode", "perf boost" and other strange options); note that disabling them in kernel won't work for all kernel versions so BIOS is preferred
* boot in single-user mode using uSingle script (the script will also disable ASLR and other funky OS features which hurt stability)
* execute your benchmark in special high-perf mode (increased prio, etc.)

# Single-user mode

Booting in single-user mode will depend on your distribution. Old-school distros support this
out-of-the-box by changing runlevel
```
$ sudo init 1
```
but you don't get this luxury with systemd/Upstart-based systems (e.g. Ubuntu or Debian).
There you can resort to *manual single-user mode*:
* put the uInit script below to /uInit (you may need to update contents of MODULES environment variable - check comment in script)
* edit kernel boot line (if you use GRUB - hold Shift at boot, then press 'e' to edit boot options):
 * find line which looks like

    ```
    linux	/boot/vmlinuz-... root=... ro splash quiet $vt_handoff
    ```

 * replace the

    ```
    splash quiet $vt_handoff
    ```

   part with

    ```
    init=/uInit
    ```

You'll then get access to bare-bones Linux system. Beware:
* Ctrl-C, Ctrl-Z, etc. shortcuts will not work so if you started a long-running program (e.g. benchmark), you have to either wait or reboot via Ctrl-Alt-Del
* background shell jobs will not work either
* you screen will not get locked, so it may be unsafe to leave it; one good practice when you start a long-running program is to finish it with exit:

    ```
    $ run_something_really_long.sh; exit
    ```

All this can probably be fixed by hacking uInit but may not worth the hassle.

# Run SPEC

Run as usual but set monitor\_specrun\_wrapper in your SPEC config to
```
monitor_specrun_wrapper = nice -20 taskset -c 1 \$command
```
to prevent thread migration. Note that we bind to the first core
as this produces more stable results on some systems (don't ask).

You may also want to set a number of portability options to avoid problems with modern GCC:

# Further work

The is probably more to be done but above instructions already achieve <0.5% noise
which is enough in practice.
If you want to lower this further, here are some suggestions:
* collect counters on a nightly SPEC run via

    ```
    $ perf record -a
    ```

  to see what's causing problems
* set performance scheduler in kernel
* turn off network
* examine overhead from kernel threads (e.g. rcuos is particularly CPU-hungry)
* examine various platform settings in https://www.spec.org/cpu2006/flags/
* experiment with performance-related BIOS settings
* enable Huge Pages in kernel
* try setting aggressive scheduling policy for benchmarks (via chrt(1))
* check Linaro's work at https://git.linaro.org/toolchain/spec2xxx-utils.git (although AFAIK they've only got to 1% noise)
* google for OS jitter and examine existing solutions
