---
layout: post
title:  "The Next-Page Prefetcher in Skylake-X"
date:   2026-10-02 23:52:25 +0530
categories: intel microarchitecture
---

Prefetchers are hardware components that preload data that the program might need soon. The goal is to eliminate
memory stalls when the CPU actually demands the piece of data that the prefetcher loaded. Many types of prefetchers
exist in modern CPUs. Next-Line prefetchers (also called Stream Prefetchers) prefetch the immediate next cache line when
the $N^{th}$ cache line is accessed. Stride prefetchers track the stride of consecutive loads and stores and try to
predict the address of future memory accesses. If the access pattern is regular enough, the stride prefetcher would
take notice and eagerly initiate the load from address that it predicted earlier. There are also Next-Page Prefetchers
that preload page table entries to the Translation Lookaside Buffer if software traverses memory pages in predictable
manner. These by no means are the only three types; a high-performance CPU may implement many other kinds of prefetchers,
some of which are documented in [this][intel-whitepaper] Intel whitepaper.


`YEAR-MONTH-DAY-title.MARKUP`

Where `YEAR` is a four-digit number, `MONTH` and `DAY` are both two-digit numbers, and `MARKUP` is the file extension
representing the format used in the file. After that, include the necessary front matter. Take a look at the source for
this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature
requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[intel-whitepaper]: https://intel.github.io/optimization-zone/hardware/HWPrefetchTuning/#123-next-page-prefetcher-l1-npp
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
