# What is this?

This document contains instructions for setting up systems with Ubuntu distros
for stable benchmarking.

Modern distributions (e.g. Ubuntu or Debian) have a lot of
running daemons or services which eat precious CPU time
(each context switch costs 2-3K cycles), pollute caches (D$, I$, TLB, BTB, etc.)
and steal DRAM/L3 bandwidth (this is collectively called "OS jitter").
If that's not enough, hyperthreading and dynamic frequency scaling (DVFS) also add to the jitter
so in practice you can see up to 5% noise in benchmark runs (e.g. SPEC2000)
which prevents reliable performance comparisons (a typical compiler
optimization for general-purpose CPU may yield around 2-3% improvement).

Note that recommendations in this guide are for obtaining stable,
but not necessarily the fastest or even "realistic", measurements.
Also they are most applicable to SPEC-like
(userspace, CPU-bound) benchmarks.

I also don't cover multithreading-specific jitter
(e.g. caused by [false sharing](https://www.reddit.com/r/rust/comments/17z7eha/false_sharing_can_happen_to_you_too/)
or priority inversion).

# Ways to reduce noise

Obviously you should run benchmarks on real hardware
(not cloud server, virtual machine, Docker or WSL).

To obtain more or less stable measurements (std. deviation less than 0.5%), you'll also need to
* disable non-deterministic HW features in BIOS
* reserve CPU cores for benchmarking (so that OS never touches them)
* boot in non-GUI mode
* disable ASLR and other funky OS features which hurt stability
* execute your benchmark in special high-perf mode (increased priority, etc.)

## Basics

This section contains some really basic (but sadly often overlooked) recommendations.

If benchmark prints a lot of output to stdout/stderr,
it should be redirected to file (or `/dev/null`).

Before measuring the time prefer to do several warmup runs
to populate OS file cache and L1i.

Fix any random numbers in benchmarks that may influence program flow.

Try to measure all compared versions of benchmark on the same day
(differences in environment temperatures may toggle slightly different
hardware frequency scaling).

Finally, avoid running other programs in parallel with benchmark
(including other benchmarks on separate cores, `top`/`htop`, etc.).
In particular, disable cronjobs.

## BIOS settings

Disable frequency scaling (sometimes called "power save mode", "turbo mode", "perf boost", etc.),
hyperthreading, HW prefetching, Turbo Boost, Intel Speed Shift and Speed Step, C-states, etc. in BIOS settings.
Note that disabling them in kernel won't work for all kernel versions so BIOS is preferred.

## Run in non-GUI mode

To boot to non-GUI mode on systemd systems see https://linuxconfig.org/how-to-disable-enable-gui-in-ubuntu-22-04-jammy-jellyfish-linux-desktop
(`sudo init 1` otherwise).

## Disable unnecessary services

Even non-GUI multi-user mode modern distros will execute a lot of services
which may distort your measurements.

For systemd distros a list of active services can be obtained via
```
$ systemctl list-units --all
```
(look for anything `active` or `waiting`).

You can identify services which actually cause problems on _your_ system by running
```
$ script -qefc 'top -cbd3' > top.log
# Wait for several hours
$ grep -A2 CPU top.log | awk '{$1=$2=$3=$4=$5=$6=$7=$8=$10=$11=""; print $0}' | grep -v 'CPU\|^$\|\<script ' | sed 's/^ *//'
```
and then disable them via `sudo systemctl mask ...`.

On typical Ubuntu desktop I suggest disabling at least
```
$ UNITS=$(systemctl list-units --all --plain | awk '/^(apt-daily|dpkg|unattended-upgrades|update-notifier|cron|anacron|avahi|fwupd|man-db|snapd|irqbalance|cups|systemd-oomd|udisks2)/{print $1}')
$ sudo systemctl mask $UNITS
```
(note that simply stopping them is not enough).

## Reserve cores for benchmarking

Add to `/etc/default/grub`:
```
# rcu_nocbs requires kernel built with CONFIG_RCU_NOCB_CPU and
# nohz_full requires CONFIG_NO_HZ_FULL
# (default Ubuntu kernels seem to have both)
GRUB_CMDLINE_LINUX_DEFAULT="nohz_full=8-15 kthread_cpus=0-7 rcu_nocbs=8-15 irqaffinity=0-7 isolcpus=nohz,managed_irq,8-15"
```
(this assumes that cores 0-7 are left to system and 8-15 are reserved for benchmarks).

Then run
```
$ sudo update-grub
```
and reboot.

You can now use `taskset 0xff00 ...` to run benchmarks on reserved cores.

When selecting cores to reserve, use `lstopo` to check that
reserved cores do not share L2/L3 with unreserved ones.

WARNING: if benchmark runs more threads than isolated cores
(which often happens to Rust programs because they rely on `std::thread::available_parallelism`
which ignores affinity settings)
NOISE INCREASES 10x (reproduced on Ubuntu 22 kernel 6.8.0-60-generic).

Additional isolation of cores can be achieved by adding `domain` to
`isolcpus` setting above but I do not recommend this because
kernel will then [schedule all benchmark threads on single core](https://serverfault.com/questions/573025/taskset-not-working-over-a-range-of-cores-in-isolcpus)
([reproduced on Ubuntu 22 and 24](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/2116749)).

This can be worked around by using `SCHED_FIFO` or `SCHED_RR` policies.

## Use benchmark-friendly scheduling policy

`SCHED_FIFO` scheduling policy may result in fewer interrupts when running benchmarks.

To enable it, add permissions to change scheduling policy for ordinary users:
```
$ sudo setcap cap_sys_nice=ep /usr/bin/chrt
```

You can then use `chrt -f 1 ...` to enable `SCHED_FIFO`.

## Fix frequency

Disable C-states by adding to `/etc/default/grub`:
```
GRUB_CMDLINE_LINUX_DEFAULT="... intel_idle.max_cstate=0 processor.max_cstate=0"
```
running
```
$ sudo update-grub
```
and rebooting.

Also disable P-states (Turbo Boost) via
```
$ echo 1 | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo
```

Finally set scaling governor to `performance` via
```
# Give CPU startup routines time to settle
$ sleep 120
$ echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```
and reduce operating frequency to avoid it being reduced under heavy load:
```
# Set XXX to 70-80% of peak
$ echo XXX | sudo tee /sys/devices/system/cpu/*/cpufreq/scaling_min_freq /sys/devices/system/cpu/*/cpufreq/scaling_max_freq
```

Note that in some situations frequency scaling may still be enforced
at hardware level to avoid overheating the CPU
(as e.g. in the [infamous AVX-512 case](https://news.ycombinator.com/item?id=23824053)).

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
environment under `env -i`
(`hyperfine` runs benchmarks with [randomized environment](https://github.com/sharkdp/hyperfine/issues/235)
for this reason).

## Fix code layout

Due to [intricacies of modern CPU frontends](https://www.bazhenov.me/posts/2024-02-performance-roulette/)
(also [here](https://easyperf.net/blog/2018/01/18/Code_alignment_issues))
some benchmarks may be sensitive to particular code layout
(i.e. offsets of functions and basic blocks)
produced by compiler and linker. This layout can vary due to unrelated changes
(e.g. [changing order in which object files are passed to linker](https://dl.acm.org/doi/10.1145/1508244.1508275))
and complicate performance comparisons.

Although not a full solution, it's recommended to compile C/C++ code with
`-falign-functions=64` (`-Z min-function-alignment=64` for Rust).
Loops may also need to be aligned to fetchline size (via `-falign-loops=N`).

# Running SPEC benchmarks

`Taskset`, `nice`, etc. can be set in `monitor_specrun_wrapper` in SPEC config:
```
monitor_specrun_wrapper = chrt -f 1 taskset 0xff00 nice -n -20 setarch -R \$command
```

# Further work

Above instructions allow to achieve <0.5% noise which is usually enough in practice.

If you want to lower this further, here are some suggestions
(I haven't tried them myself though):
* use transparent Huge Pages (to reduce TLB pressure)
* turn off network via

    ```
    # Ubuntu (note that change is permanent)
    $ nmcli networking off

    # Didn't work for me
    $ sudo /etc/init.d/networking stop
    $ systemctl stop networking.service
    ```
* [boot to single-user mode](https://askubuntu.com/questions/132965/how-do-i-boot-into-single-user-mode-from-grub)
  - this would also disable networking
* run from ramdisks
* examine various platform settings in https://www.spec.org/cpu2006/flags/
* experiment with performance-related BIOS settings
* use `numactl` to control NUMA affinity
* disable thread migration
* disable returning memory to system in Glibc
  (to avoid e.g. [TLB shootdowns](https://github.com/bitcharmer/tlb_shootdowns))
  - `export M_MMAP_MAX=0 M_ARENA_MAX=1 M_TRIM_THRESHOLD=-1`
* disable irqbalancer (`IRQBALANCE_BANNED_CPULIST`)
* disable watchdogs
  - `nmi_watchdog=0 nowatchdog nosoftlockup`
* disable kernel hardening (`pti=off`, etc.)

# Additional readings

* Technical Itch: Reducing system jitter: [part 1](https://epickrram.blogspot.com/2015/09/reducing-system-jitter.html)
  and [part 2](https://epickrram.blogspot.com/2015/11/reducing-system-jitter-part-2.html)
* [Interference-free Operating System](https://arxiv.org/abs/2412.18104)
* [LinuxCNC latency and jitter improvements with PREEMPT_RT kernel parameter tuning](https://dantalion.nl/2024/09/29/linuxcnc-latency-jitter-kernel-parameter-tuning.html)
