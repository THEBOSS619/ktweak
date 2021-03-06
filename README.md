# ktweak
A no-nonsense kernel tweak script for Android devices, backed with evidence.

# Another "kernel optimizer"?
No. Well, yes. However, a "kernel optimizer" is a poor way to put it. KTweak performs kernel adjustments based on facts and evidence. Unlike other optimizers with poorly written or heavily obfuscated code. For example:

* [LSpeed](https://github.com/Magisk-Modules-Grave/lspeed/blob/master/system/etc/lspeed/binary/main_function#L3896) is almost 4000 lines long; completely unnecessary.
* [NFS Injector](https://github.com/Magisk-Modules-Grave/nfsinjector/tree/master/system/etc/nfs/arm) uses compiled binaries that are closed source... yuck. Not to mention the typos in the README. This one is hard to look at.
* [LKT](https://github.com/Magisk-Modules-Grave/legendary_kernel_tweaks/blob/master/common/system.prop) sets random nonsensical build.props that likely don't even exist.
* [MAGNETAR](https://github.com/Magisk-Modules-Grave/MAGNETAR) uses (you guessed it) compiled binaries that install themselves to your */system/etc/ directory* (???). Great idea, install an external closed source, compiled binary to the system partition.

Need I go on?

# What's different about KTweak?
Unlike other "kernel optimizers", KTweak is:

* Concice, at around 200 lines long,
* Entirely open source with no compiled components,
* Backed by logic and evidence,
* Designed by an experienced kernel developer,
* Non-intrusive, being completely systemless.

# Benchmarks
The following benchmarks were performed on a OnePlus 7 Pro running the stock kernel provided by the OEM on Android 10.

### `hackbench -pTl 4000` (lower is better)
* Without KTweak: ~20-50 seconds on average
* With KTweak: ~4-6 seconds on average

### `perf bench mem memcpy` (lower is better) (average of 50 iters)
* Without KTweak: 14.01 ms
* With KTweak: 10.40 ms

### `synthmark` (voicemark) (higher is better)
* Without KTweak: 374.94
* With KTweak: 383.556

### `synthmark` (latencymark little) (lower is better)
* Without KTweak: 10
* With KTweak: 10

### `synthmark` (latencymark big) (lower is better)
* Without KTweak: 12
* With KTweak: 10

# The Tweaks
In order to remain genuine, I have commited to explaining each and every kernel tweak that KTweak applies. Grab your coffee, this could take a while.

### kernel.perf_cpu_time_max_percent: 25 --> 5
This is the **maximum** CPU time long perf event processing can take as a percentage. If this percentage is exceeded (meaning perf event processing used too much CPU time), the polling rate is throttled. This is reduced from 25% to 5%. We can afford inaccuracies with perf events in exchange for more time that a foreground task can use.

### kernel.randomize_va_space: 2 --> 0
ASLR has been shown to induce additional cache pressure on 32 bit executables, especially those compiled with PIE. It is a security feature, although we may see better memory performance with it disabled.

### kernel.sched_autogroup_enabled: 0 --> 1
The Linux Kernel scheduler (CFS) distributes timeslices to each active task. For example, if the scheduling period is 10ms, and there are 5 tasks running, CFS will give each task 2ms of runtime for that scheduling cycle. However, this means that a SCHED_OTHER task may compete with a SCHED_FIFO task. Autogrouping groups task groups together during scheduling. For example, if the scheduling period is 10ms, and there are 6 SCHED_OTHER tasks running and 4 SCHED_FIFO tasks running, the SCHED_OTHER tasks will get 50% of the runtime and the SCHED_FIFO tasks will get the other 50%. For each task group, the timeslices are once again divided. The SCHED_FIFO tasks will get 12.5% runtime and the SCHED_OTHER tasks will get ~8.3% runtime. This usually offers better interactivity on multithreaded platforms.
See scheduling priority documentation: https://man7.org/linux/man-pages/man7/sched.7.html
See autogrouping off: https://www.youtube.com/watch?v=uk70SeGA7pg
See autogrouping on: https://www.youtube.com/watch?v=prxInRdaNfc

### kernel.sched_enable_thread_grouping: 0 --> 1
To my knowledge using the limited documentation of this tunable, this is basically autogrouping for thread groups.

### kernel.sched_child_runs_first: 0 --> 1
When forking a child process from the parent, execute the child process before the parent process. This usually shaves down some latency on task initializations, since most of the time the child process is doing some form of heavy lifting.

### kernel.sched_downmigrate: 40    40
Do not allow tasks to migrate back down to a lower-power CPU until the estimated CPU utilization would go below 40% on said CPU. This means tasks will stay on higher-performance CPUs for longer than usual.

### kernel.sched_upmigrate: 60    60
Similar to the previous tunable, do not allow CPUs to migrate to the higher-performance CPUs unless the utilization goes above 60%.

### kernel.sched_group_downmigrate: 40
The same as kernel.sched_downmigrate, except for whole task groups.

### kernel.sched_group_upmigrate: 60
The same as kernel.sched_upmigrate, except for whole task groups.

### kernel.sched_tunable_scaling: 0
This is more of a precaution than anything. Since the next few tunables will be scheduler timing related, we don't want the scheduler to scale our values for multiple CPUs, as we will be providing CPU-agnostic values.

### kernel.sched_latency_ns: 10000000 (10ms)
Set the default scheduling period to 10ms. If this value is set too low, the scheduler will switch contexts too often, spending more time internally than executing the waiting tasks.

### kernel.sched_min_granularity_ns: 1000000 (1ms)
Set the minimum task scheduling period to 1ms. With kernel.sched_latency_ns set to 1ms, this means that 10 tasks may execute within the 10ms scheduling period before we exceed it.

### kernel.sched_migration_cost_ns: 500000 (0.5ms) --> 1000000 (1ms)
Increase the time that a task is considered to be cache hot. According to RedHat, increasing this tunable reduces the number of task migrations. This should reduce time spent balancing tasks and increase per-task performance.
See RedHat: https://www.redhat.com/files/summit/session-assets/2018/Performance-analysis-and-tuning-of-Red-Hat-Enterprise-Linux-Part-1.pdf

### kernel.sched_min_task_util_for_boost: 40
When a conservative sched_boost occurs, consider migrating the task to a higher-performance CPU if it's utilization is above this amount.

### kernel.sched_min_task_util_for_colocation: 20
When perfd triggers a sched_boost, consider migrating the task to a higher-performance CPU if it's utilization is above this amount.

### kernel.sched_nr_migrate: 32 --> 64
When migrating tasks between CPUs, allow the scheduler to migrate twice as many as usual. This should increase scheduling latency marginally, but increase the performance of SCHED_OTHER tasks.

### kernel.sched_rt_runtime_us: 950000 --> 1000000
Allow realtime tasks to consume the entirety of the scheduling period. While this may lead to CPU deadlocks if a rouge task is stuck in a loop, it can offer an additional 5% performance gain to realtime tasks.

### kernel.sched_schedstats: 1 --> 0
Disable scheduler statistics accounting. This is just for debugging, but it adds overhead.

### kernel.sched_wakeup_granularity_ns: 1000000 (1ms) --> 5000000 (5ms)
Require the current task to be surpassing the new task in vmruntime by 5ms instead of 1ms before preemption occurs. This should reduce jitter due to less frequent task interruptions.

### kernel.timer_migration: 1 --> 0
Disable the migration of timers among CPUs. Usually, when a timer is created on one CPU, it would be able to be migrated to another CPU. However, this increases realtime latencies and scheduling interrupts. It can be turned off.

### net.ipv4.tcp_ecn: 2 --> 1
Enable Explicit Congestion Notification for incoming and outgoing negotiations. This reduces packet losses.

### net.ipv4.tcp_fastopen: 3
Enable data transmission during the SACK exchange point in TCP negotiation. This reduces packet latencies. Enable it for senders and receivers.

### net.ipv4.tcp_slow_start_after_idle: 1 --> 0
Do not ramp up TCP speeds after being idle. Turning this off increases persistent connection speeds (i.e. during live video streaming without buffering, or during online gaming).

### net.ipv4.tcp_syncookies: 1 --> 0
This tunable, when enabled, prevents denial of service attacks by allowing connection ACKs to be tracked. However, this is more-or-less unnecessary for a mobile device. It is more applicable for servers. Disable it.

### net.ipv4.tcp_timestamps: 1 --> 0
RedHat claims that TCP timestamps may cause performance spikes due to time accounting code on high-performance connections. Disable it.
See RedHat: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_for_real_time/7/html/tuning_guide/reduce_tcp_performance_spikes

### vm.compact_unevictable_allowed: 1 --> 0
Do not allow compaction of unevictable pages. With this set to 1, more compactions can happen at the cost of small page fault stalls. Turn this off to compact less but avoid aforementioned stalls.

### vm.dirty_background_ratio: 5 --> 3
Start writing back dirty pages (pages that have been modified but not yet written to the disk) asynchronously at 3% memory dirtied. It's better to start background writeback early to avoid hitting the dirty_ratio poiaforementioned.

### vm.dirty_ratio: 20 --> 30
This tunable is the same as the former, but it is the ceiling for **synchronous** dirty writeback, meaning all I/O will stall until all dirty pages are written out to the disk. We usually won't need to worry about hitting this value, as the background writeback can catch up before we reach 20% memory dirtied. But as a precaution (i.e. heavy file transfers), increase this value to a 30% ceiling to prevent visible system stalls. We are sacrificing available memory in exchange for a reduced change of a brief system stall.

### vm.dirty_expire_centisecs: 300 (3s) --> 1000 (10s)
This is the longest that dirty pages can remain in the system before they are forcefully written out to the disk. By increasing this value, we can allow the dirty background writeback to take its time asynchronously, and avoid unnecessary writebacks that can clog the flusher thread.

### vm.dirty_writeback_centisecs: 500 (5s) --> 0 (0s)
Do not periodically writeback data every 5 seconds. Instead, leave it to the dirty background writeback to wake up when the dirty memory of the system hits 10%. This allows the dirty pages to stay in memory for longer, possibly increasing cache locality as the page cache is still available in memory.

### vm.extfrag_threshold: 500 --> 750
Compact memory more often, even if the memory allocation was estimated to be due to a low-memory status. This lets us put more data into RAM at the expense of running compation more often. This is a worthy tradeoff, as it reduces memory fragmentation, which is incredibly important for ZRAM.

### vm.oom_dump_tasks: 1 --> 0
Do not dump debug information when (or if) we run out of memory. If we have a lot of tasks running, and are OOMing often, then this overhead can add up.

### vm.page-cluster: 3 --> 0
Disable reading additional pages from the swap device (in most cases, ZRAM). This is the same philosophy as disabling readahead.

### vm.reap_mem_on_sigkill: 0 --> 1
When we kill a task, clean its memory footprint to free up whatever amount of RAM it was consuming.

### vm.stat_interval: 1 --> 10
Update /proc/stat information every 10 seconds instead of every second, reducing jitter on loaded systems.

### vm.swappiness: 100 --> 80
Swap to ZRAM less often if we don't have to. ZRAM can become expensive due to constant compression and decompression. If we can keep some of the memory uncompressed in regular RAM, we can avoid that overhead.

### vm.vfs_cache_pressure: 100 --> 200
This tunable controls the kernel's tendency to reclaim inodes and dentries over page cache. Inodes and dentries are information about file metadata and directory structures, while page cache is the actual cached contents of a file. By increasing this value to 200, we tell the kernel to prefer claiming inodes and dentries over the page cache, increasing the chance of a cache hit when referencing recently used data, while not polluting the RAM with less-important information.

### vm.watermark_scale_factor: 10 --> 100
Wake up kswapd to compact memory more often. This should help prevent LMK or LMKD from needlessly killing tasks if the cause of the low-memory condition happens to be fragmentation.

### Disabling Gentle Fair Sleepers
GFS gives recently awoken tasks 50% more virtual runtime than existing tasks in order to catch up with the rest of the system. While this makes sense, it also takes time away from already running tasks. Disabling GFS can improve jitter and it may improve throughput of high-performance tasks.

### Next Buddy
By scheduling the last woken task first, we can increase cache locality since that task is likely to touch the same data as before.

### No Strict Skip Buddy
Usually, the scheduler will always choose to skip tasks that call `yield()`. However, these yeilding tasks may be of higher importance than the last or next buddy that are available. Do not always skip the skip buddy if we don't have to.

### No Nontask Capacity
The scheduler decrements the perceived CPU capacity that longer the CPU has been idle for. This means that an idle CPU may be skipped during task placement, and a task can be grouped with a busier CPU. Disable this to improve task start latency.

### TTWU Queue
Allow the scheduler to place tasks on their origin CPU, increasing cache locality if the CPU is non-local (i.e. a cache hit would definitely have been missed).

### Governor Tweaks
* hispeed_load: 90 --> 80: Jump to a higher frequency if we are approaching the end of the frequency list, where a task may begin to starve or begin to stutter.
* hispeed_freq: <max>: Set the "higher freq" (referencing hispeed_load) to the maximum frequency available to take advantage of [Race-To-Idle](https://lwn.net/Articles/281629/).

### CAF CPU Boost Tweaks
* input_boost_freq: 1.4 GHz (closest freq) as a generic, universal boost frequency.
* input_boost_ms: 250 ms, not consuming too much power but boosting for important, interactive events such as clicking on things.

### I/O
* iostats: 1 --> 0: Disable I/O statistics accounting, which adds overhead.
* readahead: 0: Disable readahead, which is intended for disks with long seek times (HDD), whereas mobile devices use flash storage with zero seek time.
* nr_requests: 128 --> 512: Allow more I/O requests to be issued before flushing the queue, slightly increasing latencies but allowing more requests to be executed before being put to sleep.
* noop / none: Use a scheduler with little CPU overhead to reduce I/O latencies, which is essential for fast flash storage (eMMC & UFS).

### ZRAM
ZRAM reduces disk wear by reducing disk writes, and also increases cache locality by allowing more data to fit in RAM at once. KTweak configures ZRAM to take up at most half of the available RAM on the system, which is a good ratio of RAM to ZRAM for a mobile device.

# Other Notes
You should know that KTweak applies after 60s of uptime as to prevent Android's init from overwriting any values.

# Contact
You can find me on telegram at @tytydraco.
Feel free to email me at tylernij@gmail.com.
