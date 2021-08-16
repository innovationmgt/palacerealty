---
title: "mosdepth ideas"
date: 2018-03-28T20:12:07-06:00
lastmod: 2018-03-28T20:12:07-06:00
draft: false
keywords: ["coverage"]
description: "introducing ideas in mosdepth for future utilities"
tags: ["coverage"]
categories: ["coverage"]
author: "Brent"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: false
toc: false
autoCollapseToc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
---

![logo](https://user-images.githubusercontent.com/1739/29678184-da1f384c-88ba-11e7-9d98-df4fe3a59924.png "logo by tom sasani")

I'm working on a new project and part of it is made possible by an observation that we
stumbled on with [mosdepth](https://github.com/brentp/mosdepth). It's something that's obvious in retrospect
but wasn't fully apparent to me until after `mosdepth` was mostly written. In short, that observation is
*computers can do stuff with arrays quickly*. 

The longer story behind that obvious and simple observation is as follows. `mosdepth` is a tool to calculate
depth from BAM/CRAM files. Rather than streaming out the coverage by keeping a heap of reads and keeping
a cursor in each read to indicate the position and popping reads off the heap as the cursor moves past the
end of the read and adding reads onto the heap as the genomic position advances, and ... [snip], 
`mosdepth` instead
allocates an array the size of the current chromosome, then considers each read one-at-a-time. For each,
it increments the read start position in the array by 1 and decrements then end position by 1 discards
that read, and proceeds with the next. Aaron used
this same algorithm in bedtools genome coverage which uses 2 arrays instead of 1. `mosdepth`
does a bit more work for reads with deletions and reads that overlap their mate, but that's it. Once
all reads from a chromosome are consumed and added, then the coverage is the cumulative sum of all elements
in the array. 

Given this array for human chromosome 1 which is over **249 million bases** (or 249 million int32's), in my laptop it takes about:

+ **0.125 seconds** to calculate this cumsum to convert the array values from start/end incr/decrs to per-base coverage
+ **0.204 seconds** to calculate the mean coverage for each of **111626** exons for all known transcripts on chromosome 1.
+ **0.756 seconds** to calculate quantized coverage (contiguous bins of coverage at specified depths) of `-q 0:1:2:3:4:5:6:7:8:12:16:20:25:30:40:`
+ **0.508** seconds to calculate the cumulative coverage distribution (proportion of bases covered at 1X, 2X... $NX)

... and so on. So yeah, computers are good at arrays and stuff.

We can add a lot of utility to `mosdepth`, all of them in a single run and it doesn't affect the runtime appreciably since (decompressing
and) parsing the bam dominate the run-time. Another benefit is that once everything is in the array, the times of all subsequent operations
are **independent** of the depth. I didn't realize all of these niceties until after it was mostly implemented.

So, this is convenient. `mosdepth` doesn't have too much in the way of new ideas, it just takes full advantage. One
thing I realized later is that depth is a special case of a more generic thing we might want to measure. For depth,
we increment/decrement the start/end of each read (or the read's component segments), but one could increment/decrement
the start/end of any event. For example the per-base locations of soft-clips/hard-clips/insertions/deletions, etc. I did
some of this in [bigly](https://github.com/brentp/bigly) which is based on the heap structure that we avoided with `mosdepth`,
but next post, I'll talk about what I'll be working on and how it's useful to do this in the `mosdepth` way.

*p.s. thanks to [Tom Sasani](https://twitter.com/tomsasani) (AKA [sixsani](https://www.nature.com/articles/nbt.4060))
for the logo and to Aaron for brainstorming sessions on what became the core of what became mosdepth*
