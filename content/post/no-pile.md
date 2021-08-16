---
title: "You don't need to pileup"
date: 2018-07-31
draft: false
keywords: ["nim", "smoove"]
description: "an alternative to pileup"
tags: ["genomics", "nim"]
categories: []

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

I stumbled on this (now) obvious way of doing things that I hadn't seen
used much/at all; In a project soon to be released, we needed to quickly assay thousands
of sites from BAM/CRAM files and do a sort of cheap genotyping--or allele
counting. Given this task, a common tool to reach for is the pile-up.

Pile-up is pretty fast but it has to do a lot of work. Even to assay a
single site, a pileup will first get each read and a pileup structure (`bam_pileup1_t` in htslib) for each read into memory. Each pileup structure keeps a sort of cursor in the read. This is a flexible
approach and pretty fast when assaying multiple consecutive sites. However, especially when
assaying sites more than a few bases apart, it does a lot of extra work so it's probably faster to not use pileup.

A faster way is to take each read:

1. iterate over the cigar until the requested genomic position is reached.
2. calculate the offest into the alignment (query) sequence for that position
3. check if the query base matches the reference.
4. discard the read.

The last item is to explicitly note that the reads are not kept in memory.
This can be done in any language that allows access to the cigar. And can be much faster
in interfaces like [pysam](http://pysam.readthedocs.io/en/latest/api.html) whose pileup API,
 when given a query of even 1 base will create an iterator that generates a pileup for
all positions that have an alignment that also overlaps the query location. So if a 10KB nanopore
read overlaps that position of interest, it will generate pileups for all 10K sites in that read
as well as the single site of interest.

Here is an example using [hts-nim](https://github.com/brentp/hts-nim/) which makes it especially palatable (IMNSHO).
It queries a particular `chrom` and `position` and checks against the expected `ref_allele`

```nim

  for aln in bam.query(chrom, position, position + 1):
    var 
      off = aln.start
      qoff = 0
      roff_only = 0
      nalt = 0
      nref = 0

    for event in aln.cigar:
      var cons = event.consumes
      if cons.query:
        qoff += event.len
      if cons.reference:
        off += event.len
        if not cons.query:
          roff_only += event.len

      # continue until we get to the genomic position
      if off <= position: continue

      # since each cigar op can consume many bases
      # calc how far past the requested position
      var over = off - position - roff_only
      # get the base 
      var base = aln.base_at(qoff - over)
      if base == ref_allele:
        nref += 1
      else:
        nalt += 1

    # now nalt and nref are the allele counts ready for use.
```

the final result has `nalt` and `nref` which can be used for genotyping. The logic could probably be simplified
a bit, but I wrote that in very short order and can now re-use. This can easily be extended
to get the base-qualities at each position (hts-nim has `aln.base_quality_at(qpos)` to complement the `aln.base_at` seen here) and to check for indels as well as SNPs. 

I haven't seen this used elsewhere, perhaps due to the convenience of the pileup API or for lack of looking?
I may try to convert this to have a sort of API (or just a function) that allows flexible querying of
this sort that hides the cigar accounting.
