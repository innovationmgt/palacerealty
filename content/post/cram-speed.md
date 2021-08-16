---
title: "get the least out of your CRAM files"
date: 2018-10-11T13:39:21-06:00
lastmod: 2018-10-11T13:39:21-06:00
draft: false
keywords: ["cram", "nim"]
description: ""
tags: ["cram", "nim"]
categories: ["cram", "nim"]
author: "Brent"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
autoCollapseToc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
---

This post is highlight the speed benefit of [CRAM](https://samtools.github.io/hts-specs/CRAMv3.pdf) files over BAM files as it seems
to not be widely used.

CRAM files are often about 50% of the size of an identical BAM for lossless compression largely due to not saving the
sequence of each read, instead keeping only the delta to the reference sequence for the alignment. Additional savings can be gained from
lossy compression of base-qualities and read-names.

The [htslib](htslib.org) implementation of CRAM has an additional advantage beyond file size: **speed**. It allows the user to specify
which of the pieces of the alignment will be needed and it only decodes those. (While this is possible in BAM format, for example as
implemented in [pybam](https://github.com/JohnLonginotto/pybam) it is not available for BAM via `htslib`.) This control can
greatly speed processing when only a few pieces of the alignment are needed. This is quite useful since CRAM parsing can otherwise be
the bottleneck for much genomics work.

In short, the most costly fields to decode are:

1. base-qualities
2. sequence
3. aux tags

It's possible to get better speed and parallelization by not decoding combinations of these. The plot below shows
**time on the Y-axis** (lower is better) that it took to iterate through chromosome 20 of my test cram
for **exclude::thread combinations on the X-axis**.

![times plot](/img/cram_exclude.png)

The bars a grouped by the excluded fields and colored by the number of decompression threads used.

Interested readers can study this in more detail, but the key points are:

1. It's possible to get a 2X speed improvement simply by not parsing the base qualities (QUAL).
2. If you don't need SEQ, QUAL (base-qualities), AUX (tags like NM, SA, etc) and RG (read-groups) you can get a 3 to 4X speedup.

This is part of what makes [mosdepth](https://github.com/brentp/mosdepth) so fast to calculate depth on CRAM as it does not need
the base-qualities, sequence or read-groups. That's [how I learned about](https://github.com/brentp/mosdepth/issues/2) this feature of CRAM.

We have recently used this to speed up [svtyper](https://github.com/hall-lab/svtyper) which uses pysam; you can see how to do that [here](https://github.com/hall-lab/svtyper/commit/b43902c9bb5295d880b2c9fe8f8b29d973b82131#diff-2dee31fbd1190eb77e963360893ea86aR127)
And we've added it to [lumpy](https://github.com/arq5x/lumpy-sv) to make pre-processing alignments about twice as fast for CRAM.

These options are available from the commandline samtools via `--input-format-options required_fields=####` which is documented [here](http://www.htslib.org/doc/samtools.html)


I have gisted the nim code to do these timings the the python script for plotting [here](https://gist.github.com/brentp/213f214dd5677dd447150e62e2e360f5)
in case anyone wants to try various combinations.
