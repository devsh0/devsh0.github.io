---
layout: post
title:  "Next-Page Prefetcher in Skylake-X"
date:   2026-10-02 23:52:25 +0530
categories: intel microarchitecture
---

## Introduction

Prefetchers are hardware components that preload data that the program might need soon. The goal is to eliminate
memory stalls when the CPU actually demands the piece of data that the prefetcher could have loaded earlier. Many
types of prefetchers exist in modern CPUs. Next-Line prefetchers prefetch the immediate next cache line when the
N'th cache line is accessed. Stride prefetchers, which are usually IP-based, track the stride of consecutive
memory accesses that originate from the same instruction. If the access pattern is regular enough, the stride
prefetcher would take notice and eagerly initiate the load from address that it predicted. Next-Page Prefetchers
preload page table entries to the Translation Lookaside Buffer (TLB) provided software traverses memory pages in
predictable manner. These are by no means the only three types; a high-performance CPU may implement many other
kinds of prefetchers, some of which are documented in [this][intel-whitepaper] whitepaper.

Prefetchers that are required to be very fast and logically simple tend not to cross page boundaries because that may
complicate things with, e.g. page faults, page walks on a TLB miss etc. For example, the next-line prefetcher is supposed
to fetch the next cache line if the accesses are sequential, but if the next cache line exists in a different page it may
decide not to prefetch at all.

Next-page prefetchers are slightly different in that regard; they prefetch the next page table entry into the TLB along
with a few cache lines from the next page. They have to work across page boundaries by design.

## Next-Page Prefetchers in Intel

Intel publishes very little about NPPs in their P-cores. Back in the Ivy Bridge days, it was known that IVB included a
next-page prefetcher. Here's an excerpt from an old copy of the Intel Architectures Optimization Reference Manual:

> Hardware prefetch enhancement: A next-page prefetcher (NPP) is added in Intel microarchitecture code name Ivy Bridge.
> The NPP is triggered by sequential accesses to cache lines approaching the page boundary, either upwards or downwards.

A recent issue of the optimization manual specifies that Redwood Cove includes a Last Level Cache (LLC) Page Prefetcher
which is like NPP but with a slightly more sophisticated prefetch algorithm:

> The LLC Page Prefetcher extends the Next-Page-Prefetcher algorithm, which detects a sequence of accesses that are about
> to reach the end of a page (either going up towards the next page in memory or down towards the previous page in memory)
> and triggers a linear prefetch to the start of the next predicted page. NPP’s intends to prefetch the translation of the
> next page and initiate prefetching of the first few cache lines from that predicted page. The LLC Page Prefetch enhances
> Next-Page Prefetcher in two significant ways:
>
>   - It issues prefetches two pages ahead in linear address space.
>   - The prefetch attempts to bring the entire 4Kb page to the last level cache using opportunistic IDI request slots
>   without the DCU or MLC buffers.
>
> LLCPP is effective when streaming through memory over a buffer greater than 8KB. Smaller buffer access might result in
> redundant prefetching to the last level cache.

Redwood Cove is a relatively recent microarchitecture that shipped in Meteor Lake P-cores starting late 2023. Ivy Bridge
on the other hand is very old; It launched back in 2012 and was discontinued by 2015, right before the initial launch of
Skylake. That raises the question, what happened in-between? Did every microarchitecture after IVB include a next-page
prefetcher? How did they work? What exactly triggered the prefetch? How many future PTEs were prefetched?

It's unlikely that the details remain unchanged from one microarchitecture to the next, but I was curious what was the
case with Skylake-X. Unfortunately, this is the only machine I have access to for my experiments, so you are very much
encouraged to do your own experiments and share the results.

## A Word on Performance Monitoring on Intel Platforms

Documenting the many details of performance monitoring on various platforms deserves a post of its own, and a fairly long
one at that. For simplicity, we'd only focus on Linux in this post. Linux allows configuring performance monitoring events
via the `perf_event_open` syscall. That's what I'll be using in my experiments as well but generally speaking, we don't
need special system APIs to access PMU counters. At least on Intel platforms, we only need the ability to write a set of
performance monitoring MSRs (usually only the IA32_PERFEVTSELx MSRs) and read the counters back from a set of corresponding
IA32_PMCx MSRs. [`WRMSR`][insn-wrmsr] and [`RDMSR`][insn-rdmsr] require Ring0 access so if you're on Windows, you might
need to [write a kernel-mode driver][win-driver] and use [`__readmsr`][win-rdmsr] and [`__writemsr`] [win-wrmsr] intrinsics
to read/write the relevant MSRs. That said, the hard part of making custom drivers for Windows isn't writing the driver
itself, rather signing it so that it could be loaded by the kernel. More on that in a future blog post.

> Refer to Vol 3B, Chapter 22 (PERFORMANCE MONITORING) of the Intel Software Developer's Manual to learn more about the layout
> and configurable fields of IA32_PERFEVTSELx. Also see [perfmon-events][perfmon-events] for a list of named events and their
> respective field configuration.
{: .note}

On Linux, counting performance monitoring events is a bit simpler. The [`man pages`][perf-man] include an example of how
to use `perf_event_open`. For our measurements we need the following three things in order:

1. Configure the events we want to count. Initially all counters will be disabled.
2. Enable the counters. Here's where the IA32_PMCx registers start ticking.
3. Stop the counters and read their values.

We will write a helper for each of the three steps: `configure_event_group()` to configure the events, `start_counters()`
to enable the counters, and `stop_counters()` to stop and read the counters. We'll define `configure_event_group()` first:

{% highlight cpp %}
// Configures a set of events (i.e. an "event group") for counting.
void configure_event_group(PerfContext& ctx) {
    if (ctx.nevents < 1 || ctx.nevents > 4) {
        printf("Number of events must be in 1..4 (inclusive)!\n");
        exit(EXIT_FAILURE);
    }

    int pid = 0;                // Count events for the calling process.
    int cpu = -1;               // Count events for the calling thread. Counters don't reset when thread migrates CPUs.
    int group_fd = -1;          // The group leader, always created with fd = -1.
    int flags = 0;              // Use no special flags.

    perf_event_attr attr;
    memset(&attr, 0, sizeof(attr));

    attr.type = PERF_TYPE_RAW;              // Allow configuring events with raw event codes.
    attr.size = sizeof(attr);
    attr.config = ctx.events[0];            // Configure the first event in the event group.
    attr.disabled = 1;                      // Start out with counting disabled.
    attr.exclude_kernel = 1;                // Stop counting when control transfers to kernel space.
    attr.exclude_hv = 1;                    // Stop counting when control transfers to hypervisor.
    attr.read_format = PERF_FORMAT_GROUP;   // Read all counters in the event group with one `read` call.

    group_fd = syscall(SYS_perf_event_open, &attr, pid, cpu, group_fd, flags);
    if (group_fd == -1) {
        perror("perf_event_open");
        exit(EXIT_FAILURE);
    }
    ctx.open_fds[0] = group_fd;

    // Add the rest of the events in the event group.
    for (int i = 1; i < ctx.nevents; i++) {
        attr.config = ctx.events[i];
        int fd = syscall(SYS_perf_event_open, &attr, pid, cpu, group_fd, flags);
        if (fd == -1) {
            perror("perf_event_open");
            exit(EXIT_FAILURE);
        }
        ctx.open_fds[i] = fd;
    }
}
{% endhighlight %}

On success, `configure_event_group()` populates the `PerfContext::open_fds[4]` array. The first file descriptor in that
array (i.e. the `group_fd`) is used to identify the event group that we configured with `configure_event_group()`. We can
now use the `group_fd` to enable the counters using `start_counters()`:

{% highlight cpp %}
// Sets off the counters for events in the event group.
void start_counters(PerfContext& ctx) {
    int ret = ioctl(ctx.open_fds[0], PERF_EVENT_IOC_ENABLE, PERF_IOC_FLAG_GROUP);
    if (ret == -1) {
        perror("ioctl");
        exit(EXIT_FAILURE);
    }
}
{% endhighlight %}

A call to `start_counters()` sets off all the counters for every event configured in the event group identified by `group_fd`.
At this point we will run our workload, then call `stop_counters()` to stop and read the counters:

{% highlight cpp %}
// Stops and reads the counters in the event group. Then does some cleanup.
void stop_counters(PerfContext& ctx) {
    // Stop the counters.
    int group_fd = ctx.open_fds[0];
    int ret = ioctl(group_fd, PERF_EVENT_IOC_DISABLE, PERF_IOC_FLAG_GROUP);
    if (ret == -1) {
        perror("ioctl");
        exit(EXIT_FAILURE);
    }
    // Read the counters.
    u64 counters_with_header[5];
    int len = (ctx.nevents + 1) * sizeof(u64);
    ret = read(group_fd, &counters_with_header, len);
    if (ret == -1) {
        perror("read");
        exit(EXIT_FAILURE);
    }
    for (int i = 0; i < ctx.nevents; i++) {
        ctx.counters[i] = counters_with_header[i + 1];
    }
    // Close all fds in the event group.
    for (int i = 0; i < ctx.nevents; i++) {
        int ret = close(ctx.open_fds[i]);
        if (ret == -1) {
            perror("close");
            exit(EXIT_FAILURE);
        }
    }
}
{% endhighlight %}

By default, Linux imposes certain restrictions on hardware PMU access due to security reasons. We can overcome them by
setting [`perf_event_paranoid`][perf-paranoid] to `0` or `-1`. For maximum control, I'm going to set it
to `-1`:

{%highlight shell %}
echo "-1" | sudo tee /proc/sys/kernel/perf_event_paranoid
{% endhighlight %}

Now we have all the required pieces to start monitoring hardware events. Throughout this post, we are interested in the
following events:

| Event                                                | Config | Description                                               |
|------------------------------------------------------|--------|-----------------------------------------------------------|
| [CPU_CLK_UNHALTED.THREAD_P][evt-core-cycles]         | 0x003C | Cycles passed in the main clock driving the core.         |
| [MEM_INST_RETIRED.STLB_MISS_LOADS][evt-miss-loads]   | 0x11D0 | Retired load instructions that start a miss in the STLB.  |
| [MEM_INST_RETIRED.STLB_MISS_STORES][evt-miss-stores] | 0x12D0 | Retired store instructions that start a miss in the STLB. |
| [DTLB_LOAD_MISSES.STLB_HIT][evt-stlb-hit]            | 0x2008 | Loads that miss the data TLB and hit the unified STLB.    |

It's note hard to notice that the event configs are of the form 0x\<Umask>\<EventSel>, where Umask and EventSel are coming
from the "PROGRAMMING INFO" column of the corresponding event in the perfmon website (the 'H' suffix is Intel's way of
saying the number is given in hexadecimal form). Not all event configs are derived like that, but for the simple ones we
are using here, that's all we need.

## TLB Hierarchy in Skylake-X

SKX (codename for Skylake-X) has two TLB levels: L1 and L2. Each SKX core gets a physically separate L1 data TLB (dTLB)
and L1 instruction TLB (iTLB). The L2 TLB (also called the Second Level TLB or STLB) is unified, i.e. both instruction
and data PTEs go to the same STLB.

Both L1 and L2 TLBs are partitioned based on the page size. L1 dTLB has a total of 100 entries: 64 reserved for 4k pages,
32 for 2M or 4M pages, and 4 for 1G pages. The dTLB is 4-way set associative so each set can hold 4 page-table entries.
The iTLB is also partitioned: has 128 entries for 4k pages, 16 entries for 2M or 4M pages, and none for 1G pages. The
STLB has 1536 entries shared between 4k and 2M pages and an additional 16 entries for 1G huge pages.

STLB is not inclusive of the L1 TLBs. That means at any given instance, a page-table entry exists in one of the L1 TLBs,
the STLB, or both.

For the purposes of this post, we don't need to dig any further into the geometry of SKX TLBs. It's time to run some
experiments!

## Experiments and Findings

The very first question that we should ask is whether the next-page prefetcher exists in SKX at all. NPPs seem like a
good idea in general; Given that Intel had the silicon budget to include one in Ivy Bridge, we expect the answer to be
"Yes" in SKX too. To test this, we will design a workload that walks a mapped range of 1000 4K pages sequentially and
measure how many TLB misses we get. I cannot imagine a more favourable access pattern for the NPP to kick in if it truly
exists. If the workload below doesn't trigger page prefetches, we can safely conclude that SKX doesn't have an NPP.

{%highlight cpp %}
[[gnu::always_inline]]
inline void touch_all_bytes(int zero) {
    u8* map_base = (u8*)MAP_BASE;
    u8 dummy = 0;
    for (u64 i = 0; i < MAP_SIZE; i++) {
        dummy += map_base[i + zero];
        zero = zero & dummy;
    }
    asm volatile("" :: "r"(dummy));
}
{% endhighlight %}

Not much going on here. We have a function that touches every byte in the mapped range starting at `MAP_BASE`. The seemingly
useless parameter `zero` helps ensure that loads from the mapped range execute serially. Take a close look at how we're using
`zero` in the loop. Loading the next byte from the mapped range at iteration `i` depends on the value of `zero` computed
at iteration `i-1`. Computing `zero` at iteration `i-1` in turn depends on computing `dummy` at iteration `i-1`. We have
a cyclic loop-carried dependency chain here: the load in `map_base[i + zero]` feeds the `add` in `dummy += ...`. The
result of the `add` then feeds the `and` in `zero = zero & ...`. Result of the `and` feeds the next load at iteration
`i+1`. In layman terms, the loads at iteration `i` and iteration `i+1` cannot execute together or out-of-order. This is
intentional: if we allowed out-of-order execution of the two loads, it could significantly perturb our results. We'll see
why in a moment.

The last line is a common trick to avoid the compiler throw away our computation; It's falsely advertising that the inline
asm uses the value of `dummy` computed in the loop and forcing the compiler to materialize `dummy` into a register.

With `g++ -O2`, the loop in the workload compiles to:

{%highlight assembly %}
.loop:
    movsxd rdx, ebp                     ; tmp = zero
    add    rdx, rax                     ; tmp += i
    add    rax, 0x1                     ; i++
    add    bl, BYTE PTR [rcx+rdx]       ; dummy += map_base[tmp]
    and    ebp, ebx                     ; zero = zero & dummy
    cmp    rax, 4096000                 ; if (i < MAP_SIZE)
    jne    .loop                        ; goto .loop
{% endhighlight %}

Nice. A few instructions here and there but we got the loop compiled exactly how we wanted. Note that the dependency chain
discussed in the C++ code translates exactly to assembly too. Running the workload gives the following counts for the two
events:

```text
DTLB_LOAD_MISSES.STLB_HIT           = 70
MEM_INST_RETIRED.STLB_MISS_LOADS    = 1
```
Over a few runs, the `DTLB_LOAD_MISSES.STLB_HIT` count fluctuates in the 50..80 band, never exceeding 80. That's already
interesting: we walked through 1000 pages in the loop but `DTLB_LOAD_MISSES.STLB_HIT` is nowhere near 1000.
`DTLB_LOAD_MISSES.STLB_HIT=70` means only 70 out of the 1000 PTE requests missed in dTLB and hit STLB. The obvious next
question is what happened to rest of the 1000 - 70 = 930 fresh PTE requests that the loop might have initiated? There are
two possible answers:

1. They miss in the STLB too.
2. They didn't miss dTLB.

We can quickly rule out (1) because `MEM_INST_RETIRED.STLB_MISS_LOADS = 1`. So that leaves us with (2): about 930 new PTE
requests hit in the dTLB when the demand arrived. But that must mean that SKX has a prefetcher that's prefilling the dTLB
with future PTEs. Yep! It's the next-page prefetcher in action.

> A subtle point that is easy to miss is that the prefetcher is loading entries into th dTLB. This doesn't automatically
> prove or disprove that it's also prefilling STLB. In SKX, the STLB acts as a victim cache, i.e. entries that are
> evicted from L1 TLBs are stored in the STLB. Thinking along those lines, it doesn't make much sense for the prefetcher
> to prefill the STLB too, but I'd refrain from any speculations. Perhaps a carefully designed experiment can unveil that
> secret, but that's not our focus in this post. We'd also not worry about the iTLB for now.
{: .note}

Microarchitecture explorations typically require careful use of statistics to extract signal buried in noise. Luckily in
this case the signal is so strong that we can get away with sloppiness.

Alright, we have established that SKX has a next-page prefetcher that prefills the dTLB. On to the next question!

## What Triggers the Next-Page Prefetcher

Something in the loop in `touch_all_bytes()` must have triggered the prefetch of page `X`. The most obvious guess is
a load from page `X-1`. Question is, which load? Is it just a single load from a special offset? A set of loads? There
are so many questions that could be asked. We need to design an experiment which narrows down the set of possibilities.
We can imagine an experiment where we touch only the left half of each page (bytes in the range `page[0..2047]`, inclusive)
and count the TLB misses. If we still see TLB miss numbers like in our first experiment, that'd mean accesses in the left
half of the page triggers the prefetch. If we instead see TLB misses that add up to 1000 (or more), we can repeat the same
experiment but this time with the right half of the page (bytes in the range `page[2048..4095]`, inclusive). A generic
workload that accepts an `offset` and a `slice_length` can handle both cases:

{%highlight cpp %}
[[gnu::always_inline]]
inline void touch_page_slice(u64 offset, u64 slice_length, int zero) {
    u8 dummy = 0;
    for (u64 p = 0; p < N_MAPPED_PAGES; p++) {
        u8* page_base = (u8*)(MAP_BASE + p * PAGE_SIZE);
        for (u64 i = offset; i < offset+slice_length; i++) {
            dummy += page_base[i + zero];
            zero &= dummy;
        }
    }
    asm volatile("" :: "r"(dummy));
}
{% endhighlight %}

When `touch_half_page()` is invoked with `offset=0, slice_length=2048`, it walks left half of each of the 1000 mapped
pages. To walk the right half, we simply call it with `offset=2048, slice_length=2048`. Here are the results:

```text
First half:
DTLB_LOAD_MISSES.STLB_HIT=33
MEM_INST_RETIRED.STLB_MISS_LOADS=1001

Second half:
DTLB_LOAD_MISSES.STLB_HIT=32
MEM_INST_RETIRED.STLB_MISS_LOADS=2
```

Walking the left half generates `MEM_INST_RETIRED.STLB_MISS_LOADS=1001`. The most sensible explanation is that the NPP
was not triggered while walking the left half, thus every new PTE request missed in the dTLB. These misses then made it
to the STLB and they missed there too.

Comparatively, walking the right half of each page is not bad at all. In fact this time the prefetcher clearly did trigger.
We can keep splitting the slice that triggers the prefetch binary-search style and further narrow the access range. The
table below shows the results for SKX:

| Left half    | Right half   | Slice length | Pretech Triggered By |
|--------------|--------------|--------------|----------------------|
| [0..2047]    | [2048..4095] | 2048         | Right Half           |
| [2048..3071] | [3072..4095] | 1024         | Right Half           |
| [3072..3583] | [3584..4095] | 512          | Left Half            |
| [3072..3327] | [3328..3583] | 256          | Right Half           |
| [3328..3455] | [3456..3583] | 128          | Left Half            |
| [3328..3391] | [3392..3455] | 64           | Neither Half         |

Hmm, neither left nor right slice of the last row triggers the prefetch. But we do know that walking `page[3328..3455]`
does work. So here's the new plan: we start our walk at offset 3328 but not stop at 3391, rather keep pushing the right
edge one byte at a time until we cause the prefetcher to trigger. For example, we'd walk `page[3328..3392]`. If that
didn't trigger NPP, we'd walk `page[3328..3393]` and so forth. We expect that the prefetcher would trigger before we reach
`page[3328..3455]`. We can reuse `touch_page_slice()` here too; just call it with `offset=3328` and `slice_length=65,66,...`.
It doesn't take long to identify the slice that triggers the prefetch: the sneaky `page[3328..3392]`. Invoking
`touch_page_slice(3328, 65, zero)` immediately triggers the prefetch. Now, 3392 is the first byte in the 53rd cache line
(counting zero-indexed) within a page. Does that mean accessing the 53rd cache line triggers the prefetch? But that can't
be true because we accessed every single byte in the 53rd line and verified no prefetches (see "Right half" of the last
row in the table). Offset 3328 also starts the 52nd cache line (counting zero-indexed) within a page. So we also know that
accessing every byte in the 52nd cache line also doesn't trigger prefetch (see the "Left half" of the last row in the table).
That leaves us with only the following conclusion: loads must cross from 52nd cache line into the 53rd cache line to trigger
the prefetch.


[intel-whitepaper]: https://intel.github.io/optimization-zone/hardware/HWPrefetchTuning/#123-next-page-prefetcher-l1-npp
[insn-wrmsr]: https://intel.github.io/SDM/instruction/WRMSR.html
[insn-rdmsr]: https://intel.github.io/SDM/instruction/RDMSR.html
[win-driver]: https://learn.microsoft.com/en-us/windows-hardware/drivers/gettingstarted/writing-a-very-small-kmdf--driver
[win-rdmsr]: https://learn.microsoft.com/en-us/cpp/intrinsics/readmsr
[win-wrmsr]: https://learn.microsoft.com/en-us/cpp/intrinsics/writemsr
[perf-man]: https://man7.org/linux/man-pages/man2/perf_event_open.2.html
[perfmon-events]: https://perfmon-events.intel.com
[perf-paranoid]: https://www.kernel.org/doc/html/latest/admin-guide/perf-security.html#unprivileged-users
[evt-core-cycles]: https://perfmon-events.intel.com/platforms/skylakex/core-events/core/?h=skylak#event-CPU_CLK_UNHALTED.THREAD_P
[evt-miss-loads]: https://perfmon-events.intel.com/platforms/skylakex/core-events/core/?h=skylak#event-MEM_INST_RETIRED.STLB_MISS_LOADS
[evt-miss-stores]: https://perfmon-events.intel.com/platforms/skylakex/core-events/core/?h=skylak#event-MEM_INST_RETIRED.STLB_MISS_STORES
[evt-stlb-hit]: https://perfmon-events.intel.com/platforms/skylakex/core-events/core/?h=skylak#event-DTLB_LOAD_MISSES.STLB_HIT
