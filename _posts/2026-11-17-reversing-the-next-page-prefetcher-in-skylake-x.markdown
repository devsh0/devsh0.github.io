---
layout: post
title:  "Reversing the Next-Page Prefetcher in Skylake-X"
date:   2026-11-17 21:30:25 +0530
categories: microarchitecture, intel, computer-architecture
---

## Introduction

Intel publishes very little about next-page prefetchers in their P-cores. Back in the Ivy Bridge days, it was known that
IVB included a next-page prefetcher thanks to this ancient excerpt from an old copy of the Intel Architectures Optimization
Reference Manual:

> Hardware prefetch enhancement: A next-page prefetcher (NPP) is added in Intel microarchitecture code name Ivy Bridge.
> The NPP is triggered by sequential accesses to cache lines approaching the page boundary, either upwards or downwards.

A recent issue of the optimization manual states that Redwood Cove includes a Last Level Cache (LLC) Page Prefetcher
which is like NPP but with a slightly more sophisticated prefetch algorithm:

> The LLC Page Prefetcher extends the Next-Page-Prefetcher algorithm, which detects a sequence of accesses that are about
> to reach the end of a page (either going up towards the next page in memory or down towards the previous page in memory)
> and triggers a linear prefetch to the start of the next predicted page. NPP intends to prefetch the translation of the
> next page and initiate prefetching of the first few cache lines from that predicted page. The LLC Page Prefetch enhances
> Next-Page Prefetcher in two significant ways:
>
>   - It issues prefetches two pages ahead in linear address space.
>   - The prefetch attempts to bring the entire 4Kb page to the last level cache using opportunistic IDI request slots
>     without the DCU or MLC buffers.

Redwood Cove is a relatively recent microarchitecture which shipped in Meteor Lake P-cores starting late 2023. Ivy Bridge
on the other hand is very old; It launched back in 2012 and was discontinued by 2015 right before Skylake launched. That
raises the question, what happened in-between? Did every microarchitecture after IVB include a next-page prefetcher?

It's unlikely that the details remain unchanged from one microarchitecture to the next but as it turns out, I recently
got unlimited access to a Skylake-X machine that I didn't have to pay a single dime for. Given the setup, it's only natural
I got nerd-sniped into reversing the next-page prefetcher...and that's what this post is about. I hope you enjoy it!

## So What's a Prefetcher

Hardware component that preloads data that the program might need soon. The goal is to eliminate memory stalls when the
CPU actually demands the piece of data that the prefetcher could have loaded earlier. All kinds of prefetchers exist in
modern CPUs. Next-Line prefetchers prefetch the immediate next cache line when the N'th cache line is accessed. Stride
prefetchers (usually IP-based) track the stride of consecutive memory accesses that originate from the same instruction and
if the access pattern turns out to be regular enough, it eagerly initiates the load from address that it predicted. Next-Page
Prefetchers preload page table entries into the Translation Lookaside Buffer (TLB) provided software traverses memory pages
in predictable manner. These by no means are the only three kind; a high-performance CPU may implement many other types
of prefetchers, some of which are documented in [this][intel-whitepaper] Intel whitepaper.

Prefetchers that are required to act fast are logically simple and tend not to cross page boundaries because that may complicate
things with, e.g. page faults and page walks on TLB misses. For example, the next-line prefetcher is supposed to fetch
the next cache line if the accesses are sequential, but if the next cache line sits in a different page it may decide not
to prefetch at all. Next-page prefetchers are slightly different in that regard; they prefetch the next page table entry
(PTE) into the TLB. They have to work across page boundaries by design.

In this post I'll try to answer the following questions:

1. Does Skylake-X have a next-page prefetcher?
2. If yes, what triggers the next-page prefetch?
3. How many "future" PTEs are prefetched?
4. Where does the NPP load future PTEs?
5. What's the direction of the prefetch? Forward-only? Backward-only (that'd be weird!)? Both?

But before we start, let's prepare the stage for our experiments.

## Setting-up Linux for Experiments

If you don't intend to run the experiments described in this post, you can skip this part. 

Documenting the many details of performance monitoring on various platforms deserves a post of its own, and a fairly long
one at that. For simplicity I'm only going to focus on Linux. Linux allows configuring performance monitoring events via
the `perf_event_open` syscall. That's what I'll be using in my experiments too but generally speaking, you don't need special
system APIs to access PMU counters. At least on Intel, you only need some way of writing a set of performance monitoring
MSRs (usually only the `IA32_PERFEVTSELx` MSRs) and read the counters back from a set of corresponding `IA32_PMCx` MSRs.
[`WRMSR`][insn-wrmsr] and [`RDMSR`][insn-rdmsr] require Ring0 access so if you're on Windows, you'd have to
[write a kernel-mode driver][win-driver] and use [`__readmsr`][win-rdmsr]/[`__writemsr`] [win-wrmsr] intrinsics to read/write
the relevant MSRs. That said, the hard part of making custom drivers for Windows isn't writing the driver itself, rather
convincing the kernel to load your driver. More on that in a future blog post.

Performance monitoring tools that allow reading PMU counters (e.g. perf, VTune) may also work. I'd stick to `perf_event_open`
because it allows better control over the exact segment of code I need to measure. The [`manpage`][perf-man] includes an
example of how to use `perf_event_open`. For our experimenting we need three things in order:

1. Configuring the events we want to count. Initially all counters will be disabled.
2. Enabling the counters and running the workload. Here's where the `IA32_PMCx` counters start ticking.
3. Once the workload completes execution, stopping the counters and reading their values.

We'll write a helper for each of these steps: `configure_event_group` to configure the events, `start_counters` to enable
the counters, and `stop_counters` to stop and read the counters. With that aside, here's `configure_event_group`:

{% highlight cpp %}
// Configures a set of events (i.e. an "event group") for counting.
void configure_event_group(PerfContext& ctx) {
    if (ctx.nevents < 1 || ctx.nevents > 4) {
        printf("Number of events must be between 1..4!\n");
        exit(EXIT_FAILURE);
    }

    int pid = 0;                // Count events for the calling process.
    int cpu = -1;               // Count events for the calling thread. Counters don't reset when thread migrates CPU.
    int group_fd = -1;          // The group leader, always created with fd = -1.
    int flags = 0;              // Use no special flags.

    perf_event_attr attr;
    memset(&attr, 0, sizeof(attr));

    attr.type = PERF_TYPE_RAW;              // Allow configuring events with raw event codes.
    attr.size = sizeof(attr);
    attr.config = ctx.events[0];            // Configure the first event in the event group.
    attr.disabled = 1;                      // Start out with counters disabled.
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

Full source code of every experiment, data structure, and helpers used in this article can be found on [github][github-gist].
The `group_fd` identifies the event group that you configure via `perf_event_open`. You can use that `group_fd` to perform
certain operations on the event group, like enabling or disabling the counters. That's what we're going to do next: enable
the counters with `start_counters` which is defined like this:

{% highlight cpp %}
// Sets off the counters for events in the event group. ctx.open_fds[0] is the group_fd.
void start_counters(PerfContext& ctx) {
    int ret = ioctl(ctx.open_fds[0], PERF_EVENT_IOC_ENABLE, PERF_IOC_FLAG_GROUP);
    if (ret == -1) {
        perror("ioctl");
        exit(EXIT_FAILURE);
    }
}
{% endhighlight %}

Calling `start_counters` sets off the counters for every event configured in the event group identified by `group_fd`.
At this point we'll run our workload, then call `stop_counters` to stop and read the counters:

{% highlight cpp %}
// Stops and reads the counters in the event group. ctx.open_fds[0] is the group_fd.
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

Before we can read PMU counters using the functions defined above, we need to tweak some settings on Linux. Depending on
your distro, certain restrictions may apply by default on hardware PMU access due to security reasons. You can overcome
them by setting [`perf_event_paranoid`][perf-paranoid] to `0` or `-1`:

{%highlight shell %}
echo "-1" | sudo tee /proc/sys/kernel/perf_event_paranoid
{% endhighlight %}

You may also need to [disable THP][disable-thp] and pin the calling thread to a core if there's too much going on in your
machine.

Now we have all the pieces required to start monitoring hardware events. Throughout this post we'd work with the following events:

| Event                                                | Config | Description                                               |
|------------------------------------------------------|--------|-----------------------------------------------------------|
| [MEM_INST_RETIRED.STLB_MISS_LOADS][evt-miss-loads]   | 0x11D0 | Retired load instructions that start a miss in the STLB.  |
| [MEM_INST_RETIRED.STLB_MISS_STORES][evt-miss-stores] | 0x12D0 | Retired store instructions that start a miss in the STLB. |
| [DTLB_LOAD_MISSES.STLB_HIT][evt-stlb-hit]            | 0x2008 | Loads that miss the data TLB and hit the unified STLB.    |

It's note hard to observe that the event configs are of the form 0x\<Umask>\<EventSel>, where Umask and EventSel are coming
straight from the "PROGRAMMING INFO" column of the corresponding event in the [perfmon website][perfmon-events] (the 'H'
suffix is Intel's way of saying the number is given in hexadecimal). Not all event configs are derived this way, but it's
enough for the simple ones we're gonna use here.

## TLB Hierarchy in Skylake-X

SKX (codename for Skylake-X) has two TLB levels: L1 and L2. Each SKX core gets a physically separate L1 data TLB (dTLB)
and L1 instruction TLB (iTLB). The L2 TLB (also called Second Level TLB or STLB) is unified, i.e. both instruction and
data PTEs go to the same STLB.

Both L1 and L2 TLBs are partitioned based on the page size. L1 dTLB has a total of 100 entries: 64 reserved for 4k pages,
32 for 2M or 4M pages, and 4 for 1G pages. The dTLB is 4-way set associative so each set can hold 4 page-table entries.
The iTLB is also partitioned: has 128 entries for 4k pages, 16 entries for 2M or 4M pages, and none for 1G pages. The
STLB has 1536 entries shared between 4k and 2M pages and an additional 16 entries for 1G huge pages. STLB is not inclusive
of the L1 TLBs. That means at any given instance, a page-table entry may exist in one of the L1 TLBs, the STLB, or both.

For the purposes of this post, we don't need to dig any further into the geometry of SKX TLBs. It's time to run some
experiments!

## Does it Exist

The very first question that we should ask is whether the next-page prefetcher exists in SKX at all. NPPs seem like a
good idea in general; given that Intel had the silicon budget to include one in Ivy Bridge, we expect the answer to be
"Yes" in SKX too. To test this, we'll design a workload that walks a mapped range of 1000 4K pages sequentially and
measure how many TLB misses we get. I cannot imagine a more favourable access pattern for the NPP to kick in if it truly
exists. If the workload below doesn't trigger the prefetcher, we can safely conclude that SKX doesn't have one.

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
useless `zero` helps ensure that loads from the mapped range execute serially. Take a closer look at how we're using
`zero` in the loop. Loading the next byte from the mapped range at iteration `i` depends on the value of `zero` computed
at iteration `i-1`. Computing `zero` at iteration `i-1` in turn depends on computing `dummy` at iteration `i-1`. We have
a cyclic loop-carried dependency chain here: the load in `map_base[i + zero]` feeds `add` in `dummy += ...`. The result
of that `add` then feeds the `and` in `zero = zero & ...`. Result of the `and` feeds the next load at iteration `i+1`.
In layman terms, the loads at iteration `i` and iteration `i+1` cannot execute together or out-of-order. This is intentional:
alloweing out-of-order execution of the two loads could significantly perturb our results. We'll see why in a moment.

The last line is a common trick to avoid the compiler throw away our computation; it's falsely advertising that the inline
asm uses the value of `dummy` computed in the loop and forcing the compiler to materialize `dummy` into a register.

With `g++ -O2`, the loop compiles to:

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
we just discussed translates exactly to assembly too. Running the workload gives the following count for the two events:

```text
DTLB_LOAD_MISSES.STLB_HIT           = 70
MEM_INST_RETIRED.STLB_MISS_LOADS    = 1
```
Over a few runs `DTLB_LOAD_MISSES.STLB_HIT` fluctuates between the 50..75 band, but never exceeds 75. That's already
interesting: we walked through 1000 pages in the loop but `DTLB_LOAD_MISSES.STLB_HIT` is nowhere near 1000.
`DTLB_LOAD_MISSES.STLB_HIT = 70` means only 70 out of the 1000 PTE requests missed in dTLB and hit STLB. The obvious next
question is what happened to rest of the 1000 - 70 = 930 fresh PTE requests that the loop might have initiated? There are
two possible answers:

1. They miss in the STLB too.
2. They didn't miss dTLB.

We can quickly rule out (1) because `MEM_INST_RETIRED.STLB_MISS_LOADS = 1`. So that leaves us with (2): about 930 new PTE
requests hit in the dTLB when the demand arrived. But that must mean that SKX has a prefetcher that's prefilling the dTLB
with future PTEs. Yep! It's the next-page prefetcher in action. Well, strictly speaking this could also be a non-NPP prefetcher
that is allowed prefetch beyond page boundaries but as they say, if it quacks like a duck, it's the next-page prefetcher!

> Subtle point that is easy to miss: the prefetcher is loading entries into th dTLB. This doesn't automatically
> prove or disprove that it's also prefilling STLB. In SKX the STLB acts as a victim cache, i.e. entries that are
> evicted from L1 TLBs are stored in the STLB. Thinking along those lines, it doesn't make much sense for the prefetcher
> to prefill the STLB too, but I'd refrain from such speculations. Perhaps a carefully designed experiment can unveil that
> secret, but that's not my focus in this post. We'd also not worry about the iTLB for now.
{: .note}

Microarchitecture explorations typically require careful use of statistics to extract signal buried in noise. Luckily in
this case the signal is so strong that we can get away with sloppiness.

Alright, we have established that SKX has a next-page prefetcher that prefills the dTLB. On to the next question!

## What Triggers the Next-Page Prefetcher

Something in the loop in `touch_all_bytes` must have triggered the next-page prefetch. The most obvious guess is a load
from the previous page. Question is, which load? Is it just a single load from a special offset? A set of loads? There
are so many questions that could be asked here...we need an experiment that narrows down the space of possibilities. We can
imagine a workload that touches only the left half of each page (bytes in the range `page[0..2047]`, inclusive) and count the
TLB misses. If we still see TLB miss numbers like in our first experiment, that'd mean accesses from the left half of the
page triggers the prefetch. If we instead see TLB misses that add up to 1000 (or more), we can repeat the same experiment
but this time touching the right half of the page (bytes in the range `page[2048..4095]`, inclusive). A generic workload
that accepts an `offset` and a `slice_length` can handle both cases:

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

When `touch_page_slice` is called with `offset = 0, slice_length = 2048`, it walks left half of each of the 1000 mapped
pages. Walking the right half is just as simple, just call `touch_page_slice` with `offset = 2048` and `slice_length = 2048`.
Results from walking the left half:

```text
Left half:
DTLB_LOAD_MISSES.STLB_HIT               = 33
MEM_INST_RETIRED.STLB_MISS_LOADS        = 1001
```

Walking the left half generated `MEM_INST_RETIRED.STLB_MISS_LOADS = 1001`. That's suspiciously close to the total number
of pages we walked! The most sensible explanation is that the NPP did not trigger while walking the left half, thus every
new PTE request missed in the dTLB. Those misses then made it to STLB and then missed there too.

Alright, what about the right half? Here's the result from waking the right half:

```text
Right half:
DTLB_LOAD_MISSES.STLB_HIT           = 32
MEM_INST_RETIRED.STLB_MISS_LOADS    = 2
```

Comparatively, walking the right half is not bad at all. In fact this time the prefetcher clearly did trigger. We can keep
splitting the slice that triggers prefetch binary-search style and further narrow the access range. The table below shows
the results:

| Left half    | Right half   | Slice length | Pretech Triggered By |
|--------------|--------------|--------------|----------------------|
| [0..2047]    | [2048..4095] | 2048         | Right Half           |
| [2048..3071] | [3072..4095] | 1024         | Right Half           |
| [3072..3583] | [3584..4095] | 512          | Left Half            |
| [3072..3327] | [3328..3583] | 256          | Right Half           |
| [3328..3455] | [3456..3583] | 128          | Left Half            |
| [3328..3391] | [3392..3455] | 64           | Neither Half         |

Hmm, neither left nor right slice of the last row triggers the prefetcher. But we do know that walking `page[3328..3455]`
does work. So here's the new plan: we start our walk at offset 3328 but not stop at 3391, rather keep pushing the right
edge one byte at a time until we cause the prefetcher to trigger. For example, we'd walk `page[3328..3392]` and if that
didn't trigger NPP, we'd walk `page[3328..3393]` and so forth. We expect that the prefetcher would trigger before we reach
`page[3328..3455]`. We can reuse `touch_page_slice` here too; just call it with `offset = 3328` and `slice_length = 65,66,...`.

It doesn't take long to identify the slice that triggers prefetch: the sneaky `page[3328..3392]`. Calling `touch_page_slice`
with `offset = 3328` and `slice_length = 65` immediately triggers the prefetch. If you're into this kind of shannonigans,
it shouldn't take too long to notice that 3392 is the first byte into the 53rd cache line (counting zero-indexed) within
a page. Ok, does that mean accessing the 53rd cache line triggers the prefetch? But that can't be true because we accessed
every single byte in the 53rd line and verified no prefetches (see "Right half" of the last row in the table). Moreover,
offset 3328 starts the 52nd cache line (counting zero-indexed) within a page, so we also know that accessing every byte
in the 52nd cache line also doesn't trigger the prefetch (see the "Left half" of the last row in the table). Could it be
that for the prefetcher to trigger, subsequent loads must cross the 52nd cache line into the 53rd cache line? If the
prefetch condition truly is as simple as "two subsequent loads, one from cache line 52, the next from cache line 53",
then the following must hold:

1. Touching every byte in a page except any from cache line 52 will not trigger prefetch.
2. Touching every byte in a page except any from cache line 53 will not trigger prefetch.
3. Two back-to-back loads, the first loading any byte in cache line 52 and the second loading any byte in cache line 53
   will be sufficient to trigger prefetch.

To test these hypotheses, we need a workload that touches every byte in a page except a few. This should work:  

{%highlight cpp %}
[[gnu::always_inline]]
inline void touch_page_with_hole(u64 hole_offset, u64 hole_length, int zero) {
    u8 dummy = 0;
    u64 repeat_idx = hole_offset == 0 ? 0 : hole_offset - 1;
    for (u64 p = 0; p < N_MAPPED_PAGES; p++) {
        u8* page_base = (u8*)(MAP_BASE + p * PAGE_SIZE);
        for (u64 i = 0; i < PAGE_SIZE; i++) {
            u64 outside_hole = (i - hole_offset) >= hole_length;
            u64 idx = (i & -outside_hole) | (repeat_idx & (outside_hole - 1));
            dummy += page_base[idx + zero];
            zero &= dummy;
        }
    }
    asm volatile("" :: "r"(dummy));
}
{% endhighlight %}

The strange looking trick to compute `idx` in the inner loop helps avoid a branch in the probe path. Yeah, we are also
banking on the fact that the branch predictor would do just fine when we're done walking a page in the inner loop and
jump back to the outer loop. Strictly speaking, a branch mispredict shouldn't mess with TLBs in a way that affects our
results but in general, eliminating noise sources to the best of your ability is a good idea with these kind of experiments.
Aside from that there's not much going on here. We are skipping the range `page[hole_offset, hole_offset+hole_length]`
and touching every other byte in the page. When `i` falls within the skipped range, we simply redirect the load to
`hole_offset - 1`.

Ok so what are the results if we skip cache line 52 and 53, i.e. range `page[3328, 3391]` and `page[3392, 3456]`:

```text
Skipping 52nd cache line (touch_page_with_hole(3328, 64, zero)):
DTLB_LOAD_MISSES.STLB_HIT           = 63
MEM_INST_RETIRED.STLB_MISS_LOADS    = 1000

Skipping 53rd cache line (touch_page_with_hole(3392, 64, zero)):
DTLB_LOAD_MISSES.STLB_HIT           = 66
MEM_INST_RETIRED.STLB_MISS_LOADS    = 1000
```

`MEM_INST_RETIRED.STLB_MISS_LOADS = 1000` for both cases. This verifies (1) and (2) from our list of hypotheses. We can
test (3) by calling `touch_page_slice` with `offset = 3391` and `slice_length = 2`. Here are the results:

```text
Load from cache line 52 and 53:
DTLB_LOAD_MISSES.STLB_HIT           = 4
MEM_INST_RETIRED.STLB_MISS_LOADS    = 3
```

As expected, merely touching a single byte in each of 52nd and 53rd cache lines engages the prefetcher. Now admittedly
the experiment doesn't prove the "any byte in either cache line" part; we've only shown that loading last byte in cache
line 52 followed by the load of first byte in cache line 53 triggers the prefetch, but you can easily verify that by
tweaking the parameters of `touch_page_with_hole`. For example, `hole_offset = 3329` and `hole_length = 126` touches first
byte of line 52 and last byte of line 53, skipping everything in-between. This still triggers prefetch. To limit the size
of this post, I'll leave out certain details like that from here on after. I'd be happy to fill those gaps in the comment
section if you have questions.

Note that for the prefetcher to trigger, load from 52nd cache line must execute before the load from 53rd cache line
(unless the prefetcher is bidirectional, but we'll get to that in a minute). This is why it was crucial to introduce
the dependency chain that throttles out-of-order execution of loads from the page.

If you're wondering why would Intel design the prefetcher with such a thin trigger condition, you're not alone. I had the
same exact question. I mean, triggering prefetch when loads get closer and closer to the next page makes sense. But why
put all your bets on crossing just the 52nd cache line?! It'd make more sense if any couple of accesses beyond a threshold
triggered the prefetch. Peter Cordes (yeah, [that Peter Cordes][peter-cordes-profile]) had a [brilliant answer][peter-cordes-answer]
on why this makes sense. Go read it if you're interested.

## What About Stores?

Good question. Our experiments so far have only used loads. A sane person would think stores shouldn't behave any different.
But to my surprise, stores don't seem to trigger prefetches. If you're browsing the [full source][github-gist] on GitHub you'd
notice that we are faulting the mapped range with loads. That choice is deliberate: we're intentionally mapping every
virtual page in the mapped range to the physical zero page. This reduces cache footprint, and potentially restricts one
more source of noise.

We cannot keep doing this for this next experiment, though; we don't want page faults in the probe path. So we switch to
faulting with stores before we start the experiment. That out of the way, here's the kernel we'll use to verify no prefetch
on stores:

{%highlight cpp %}
[[gnu::always_inline]]
inline void touch_with_stores(int zero) {
    u8* map_base = (u8*)MAP_BASE;
    for (u64 i = 0; i < MAP_SIZE; i++) {
        map_base[i + zero] = i;
        zero *= zero;
    }
}
{% endhighlight %}

Running this produces `MEM_INST_RETIRED.STLB_MISS_STORES = 1000`. Note that this time we are measuring STLB misses on
stores. You can verify that each of these misses start at least one page-walk by counting `DTLB_STORE_MISSES.WALK_COMPLETED`
along with `MEM_INST_RETIRED.STLB_MISS_STORES`. That rules out the possibility of dTLB hits and a last-level TLB (other than
STLB) catching all the misses from STLB.

## How Many PTEs are Prefetched

We have established that the next-page prefetcher prefetches at least one future PTE. Let's figure out how many future
PTEs are prefetched _exactly_. We'll use the following workload for this:

{%highlight cpp %}
void how_many_pages_prefetched(int future_limit, int zero) {
    u8 dummy = 0;
    int poke_limit = 8;
    for (int p = 0; p < poke_limit; p++) {
        u8* page_base = (u8*)(MAP_BASE + p * PAGE_SIZE);
        dummy += page_base[3391 + zero];
        zero &= dummy;
        dummy += page_base[3392 + zero];
        zero &= dummy;
    }
    for (int i = 0; i < future_limit; i++) {
        u8* page_base = (u8*)(MAP_BASE + (i + poke_limit) * PAGE_SIZE);
        dummy += page_base[zero];
        zero &= dummy;
    }
    asm volatile("" :: "r"(dummy));
}
{% endhighlight %}

The idea is to "heat up" the prefetcher by crossing 52nd cache line in the first `poke_limit` pages — that's what the first
loop does — then load first byte of `future_limit` pages beyond `poke_limit` in the second loop. We'll call this function
repeatedly, each time increasing `future_limit` by 1, and count the TLB misses. We can get a baseline with `future_limit = 0`
first:

```text
Run 1:
DTLB_LOAD_MISSES.STLB_HIT           = 1
MEM_INST_RETIRED.STLB_MISS_LOADS    = 1

Run 2:
DTLB_LOAD_MISSES.STLB_HIT           = 2
MEM_INST_RETIRED.STLB_MISS_LOADS    = 1

Run 3:
DTLB_LOAD_MISSES.STLB_HIT           = 0
MEM_INST_RETIRED.STLB_MISS_LOADS    = 3

...
```

Over many runs, STLB miss baseline fluctuates between 1 and 3 but never seems to exceed 3. Alright, we can work with that.
Now let's try `future_limit = 1`:

```text
Run 1:
DTLB_LOAD_MISSES.STLB_HIT           = 0
MEM_INST_RETIRED.STLB_MISS_LOADS    = 3

Run 2:
DTLB_LOAD_MISSES.STLB_HIT           = 1
MEM_INST_RETIRED.STLB_MISS_LOADS    = 2

Run 3:
DTLB_LOAD_MISSES.STLB_HIT           = 1
MEM_INST_RETIRED.STLB_MISS_LOADS    = 2

...
```

Yep, expected! The last load in the "heater" must've triggered the prefetch for page at `poke_limit`, so `future_limit = 1`
shouldn't increase the number of observed misses. What about `future_limit = 2`?

```text
Run 1:
DTLB_LOAD_MISSES.STLB_HIT           = 0
MEM_INST_RETIRED.STLB_MISS_LOADS    = 4

Run 2:
DTLB_LOAD_MISSES.STLB_HIT           = 0
MEM_INST_RETIRED.STLB_MISS_LOADS    = 5

Run 3:
DTLB_LOAD_MISSES.STLB_HIT           = 1
MEM_INST_RETIRED.STLB_MISS_LOADS    = 4

...
```

Looks like the NPP prefetches only one page ahead. Cool, no complaints there!

## Prefetch Direction

Does the prefetcher work in backward direction too? This is the last piece of the puzzle and fortunately very easy to test:
just walk the mapped range backwards. If you get close to 1000 misses the answer is no, otherwise yes. Here's the workload:

{%highlight cpp %}
[[gnu::always_inline]]
inline void touch_bytes_backward(int zero) {
    u8* map_base = (u8*)MAP_BASE;
    u8 dummy = 0;
    for (u64 i = 1; i <= MAP_SIZE; i++) {
        dummy += map_base[MAP_SIZE - i + zero];
        zero &= dummy;
    }
    asm volatile("" :: "r"(dummy));
}
{% endhighlight %}

...and the results:

```text
DTLB_LOAD_MISSES.STLB_HIT               = 69
MEM_INST_RETIRED.STLB_MISS_LOADS        = 1000
```

This is a bit shocking. If you recall from the "Introduction" section, Intel manuals seem to suggest the next-page prefetcher
triggers while walking "upwards" (what I call forward) or "downward" (what I call backward). This behavior is documented
for both Ivy Bridge and Redwood Cove...but that's not what we see for SKX. I don't have any theories explaining this disconnect
between SKX and the other uarchs, if you do feel free to share in the comment section.

## So Where Can I Use This Knowledge

If you're a software developer looking for a cool new way of optimizing program performance, probably nowhere. Optimizing
programs for a single microarchitecture from a single CPU vendor is almost always a bad idea for anything with half-life
beyond next Friday. My goal here is to share how I attempt reversing undocumented machine behavior and invite fellow nerds
to kick-start discussions that hopefully help me or somebody else learn something new.

That said, if you're one of those weirdos who can manage to care only about one type of machine (Skylake-X to be precise),
you may find the details interesting from a practical PoV. Either way, I'm tired. This is too much writing for a first
post. 👋

[intel-whitepaper]: https://intel.github.io/optimization-zone/hardware/HWPrefetchTuning/#123-next-page-prefetcher-l1-npp
[insn-wrmsr]: https://intel.github.io/SDM/instruction/WRMSR.html
[insn-rdmsr]: https://intel.github.io/SDM/instruction/RDMSR.html
[win-driver]: https://learn.microsoft.com/en-us/windows-hardware/drivers/gettingstarted/writing-a-very-small-kmdf--driver
[win-rdmsr]: https://learn.microsoft.com/en-us/cpp/intrinsics/readmsr
[win-wrmsr]: https://learn.microsoft.com/en-us/cpp/intrinsics/writemsr
[perf-man]: https://man7.org/linux/man-pages/man2/perf_event_open.2.html
[disable-thp]: https://access.redhat.com/solutions/46111
[perfmon-events]: https://perfmon-events.intel.com
[perf-paranoid]: https://www.kernel.org/doc/html/latest/admin-guide/perf-security.html#unprivileged-users
[evt-core-cycles]: https://perfmon-events.intel.com/platforms/skylakex/core-events/core/?h=skylak#event-CPU_CLK_UNHALTED.THREAD_P
[evt-miss-loads]: https://perfmon-events.intel.com/platforms/skylakex/core-events/core/?h=skylak#event-MEM_INST_RETIRED.STLB_MISS_LOADS
[evt-miss-stores]: https://perfmon-events.intel.com/platforms/skylakex/core-events/core/?h=skylak#event-MEM_INST_RETIRED.STLB_MISS_STORES
[evt-stlb-hit]: https://perfmon-events.intel.com/platforms/skylakex/core-events/core/?h=skylak#event-DTLB_LOAD_MISSES.STLB_HIT
[peter-cordes-profile]: https://stackoverflow.com/users/peter-cordes
[peter-cordes-answer]: https://stackoverflow.com/a/80011111/14180973
[github-gist]: https://gist.github.com/devsh0/66fea1019aee8eda54925ee75b73852a
