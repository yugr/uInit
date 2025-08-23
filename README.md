# What is this?

This document contains instructions for setting up systems with Ubuntu distros
for stable benchmarking.

Modern distributions (e.g. Ubuntu or Debian) have a lot of
running daemons or services which eat precious CPU time and pollute caches
(this is collectively called "OS jitter").
If that's not enough, hyperthreading and dynamic frequency scaling (DVFS) also add to the jitter
so in practice you can see up to 5% noise in benchmark runs (e.g. SPEC2000)
which prevents reliable performance comparisons (typical compiler
optimization may yield around 2-3% improvement).

# Ways to reduce noise

To obtain more or less stable measurements (std. deviation less than 0.5%), you'll need to
* disable non-deterministic HW features in BIOS
* reserve CPU cores for benchmarking (so that OS never touches them)
* boot in non-GUI mode
* disable ASLR and other funky OS features which hurt stability
* execute your benchmark in special high-perf mode (increased priority, etc.)

## BIOS settings

Disable frequency scaling (sometimes called "power save mode", "turbo mode", "perf boost", etc.), hyperthreading, HW prefetching, Turbo Boost, Intel Speed Shift, etc. in BIOS settings.
 e.g. hyperthreading and frequency scaling ; note that disabling them in kernel won't work for all kernel versions so BIOS is preferred

## Run in non-GUI mode

To boot to non-GUI mode on systemd systems see https://linuxconfig.org/how-to-disable-enable-gui-in-ubuntu-22-04-jammy-jellyfish-linux-desktop
(`sudo init 1` otherwise).

## Reserve cores for benchmarking

Add to `/etc/default/grub`:
```
GRUB_CMDLINE_LINUX_DEFAULT="nohz=on nohz_full=8-15 kthread_cpus=0-7 irqaffinity=0-7 isolcpus=domain,managed_irq,8-15"
```
(this assumes that cores 0-7 are left to system and 8-15 are reserved for benchmarks).
The run
```
$ sudo update-grub
```
and reboot.

Also add permissions to change scheduling policy for ordinary users:
```
$ sudo setcap cap_sys_nice=ep /usr/bin/chrt
```
(otherwise kernel [will schedule all threads which run under `taskset` on same core](https://serverfault.com/questions/573025/taskset-not-working-over-a-range-of-cores-in-isolcpus), [reproduced on Ubuntu 22 and 24](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/2116749)).

You can now use `chrt -f 1 taskset 0xff00 ...` to run benchmarks on reserved cores
(`chrt -r` also works, `chrt -b` does not).

WARNING: if benchmark runs more threads than isolated cores (which e.g. happens
with Rust benchmarks which rely on `std::thread::available_parallelism`)
NOISE INCREASES 10x (reproduced on Ubuntu 22 kernel 6.8.0-60-generic).

## Fix frequency

Set scaling governor to `performance` via
```
# Give CPU startup routines time to settle
$ sleep 120
$ echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

## Increase priority

Add to `/etc/security/limits.conf`:
```
USERNAME soft nice -20
USERNAME hard nice -20
```
and run benchmarks under `nice -n -20 ...`.

## Disable ASLR

Run benchmarks under `setarch -R ...`.

## Fix start address of stack

On POSIX systems memory segment that is used for stack is first filled
with environment variables, then padded to OS-specific alignment and
finally the aligned address is used as start of the program stack.

Because of this addresses of program's local variables
may change depending on environment variables even if you've disabled ASLR.
Some variables, like `$PWD` or `$_`, may vary across benchmark invocations
and influence results (5% fluctuations are not uncommon for microbenchmarks).

It is thus strongly recommended to run benchmarks that do not rely on
environment under `env -i`.

## Fix code layout

Due to [intricacies of modern CPU frontends](https://www.bazhenov.me/posts/2024-02-performance-roulette/)
some benchmarks may be sensitive to particular code layout
(i.e. offsets of functions and basic blocks)
produced by compiler and linker. This layout can vary due to unrelated changes
(e.g. [changing order in which object files are passed to linker](https://dl.acm.org/doi/10.1145/1508244.1508275))
and complicate performance comparisons.

Although not a full solution, it's recommended to compile C/C++ code with
`-falign-functions=64` (`-Z min-function-alignment=64` for Rust).

# Running SPEC benchmarks

`Taskset`, `nice`, etc. can be set in `monitor_specrun_wrapper` in SPEC config:
```
monitor_specrun_wrapper = chrt -f 1 taskset 0xff00 nice -n -20 setarch -R \$command
```

# Further work

Above instructions allow to achieve <0.5% noise which is usually enough in practice.

If you want to lower this further, here are some suggestions:
* turn off network via

    ```
    # Ubuntu (note that change is permanent)
    $ nmcli networking off

    # Didn't work for me
    $ sudo /etc/init.d/networking stop
    $ systemctl stop networking.service
    ```

* [boot to single-user mode](https://askubuntu.com/questions/132965/how-do-i-boot-into-single-user-mode-from-grub)
* run from ramdisks
* examine various platform settings in https://www.spec.org/cpu2006/flags/
* experiment with performance-related BIOS settings
* enable Huge Pages in kernel
