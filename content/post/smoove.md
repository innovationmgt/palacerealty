---
title: "Smoove"
date: 2018-03-20T16:49:22-06:00
lastmod: 2018-03-20T16:49:22-06:00
draft: false
keywords: ["SV", "smoove"]
description: "introduce smoove"
tags: ["smoove", "SV"]
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

[smoove](https://github.com/brentp/smoove) wraps existing software and adds some internal read-filtering to
**simplify calling and genotyping structural variants**. It parallelizes each step as it can, for example, it
streams [lumpy](https://github.com/arq5x/lumpy-sv) output directly to multiple [svtyper](https://github.com/hall-lab/svtyper)
processes for genotyping.
It contains several sub-commands but users with cohorts of less than about 40 samples can get a
joint-called, genotyped VCF in a single command:

```Bash
smoove call -x --genotype --name $name --outdir . \
           -f $fasta --processes 12 --exclude $bed *.bam
```

It also greatly simplifies population-level calling.
We have recently used `smoove` to jointly call structural variants in **2392 samples on AWS in about 1 hour of human time**.

This should work for any diploid species with a reference genome.

#### Filters and Accuracy

`smoove` uses `lumpy_filter` from the [lumpy](https://github.com/arq5x/lumpy-sv)
repo to extract split (having a SA tag) and discordant (having an insert size outside of the expected range) reads as required
by lumpy. If `$sample.split.bam` and `$sample.disc.bam` are present (or soft-linked) to the output directory, then `smoove` will use those
so output from [samblaster](https://github.com/GregoryFaust/samblaster) can be used to bypass the need for `lumpy_filter`.

It will then further filter those reads as follows:

1. reads where both ends are soft-clips are excluded, e.g. discard a read with cigar `20S106M34S` but keep `87S123M`.
2. discard interchromosomal reads with > 3 mismatches (determined by NM sam tag).
3. discard interchromosomal that also has an XA tag indicating alternative matches.
4. discard interchromosomal split-reads are discarded unless the split (SA tag) goes to the same general location as the mate.
5. remove reads in exclude regions or on excluded chromosomes.
6. remove reads from regions where the depths of split or discordant reads is greater than 1000. This removes regions that
   contribute to spurious calls.
7. remove reads that are orphaned (dont have a mate) after all of the above filtering.

This removes more than **80%** of the reads from the `.split.bam` and `.disc.bam` files that are sent to lumpy.
In our tests, compared to vanilla lumpy, we found
that, in a [pac-bio derived truth set](http://eichlerlab.gs.washington.edu/publications/chm1-structural-variation/),
these filters removed about 4 ostensibly *true* deletions out of 1191 while removing 120 out of 910 deletions that
were not found in the pac-bio data. There is also a considerable speed improvement and memory reduction using `smoove`
as compared to vanilla `lumpy`. We will do more work to filter even more reads (and perhaps add some missing ones)
to make this more efficient and accurate.

#### Usage

The simplest way to get `smoove` and all dependencies is via [dockerhub](https://hub.docker.com/r/brentp/smoove/) (and soon via bioconda).

For a small cohort, the single command above is sufficient; for population-level calling, `smoove` should be used to first call by
sample, or by family, or in groups of `n` (we do this by family). Then it can extract and merge all sites from all groups (internally using
[svtools](https://github.com/hall-lab/svtools)). Then it can genotype at those sites (again, per sample or per small group),
and finally, it can create a square VCF using [bcftools](https://github.com/samtools/bcftools) merge.

Each of these commands is documented in the [smoove README](https://github.com/brentp/smoove).

#### Noise

As indicated, I will look into additional read-filters that can be applied. If you know of or see a noise signal that could be
used to reliably remove reads that lead to spurious calls, please comment, or open an issue on the
[smoove repo](https://github.com/brentp/smoove).

In calling the 2392 samples mentioned above, it is clear that there are several avenues to explore for filtering. One noise signature that appears to be
associated with bad calls is when a variant has a low allele balance (hovering around 0.1) in many samples and never approaches the expected value
of 0.5. Though we expect mapping bias, it should not be this severe; signals like this are indicative of low-level noise causing spurious calls.
We can also look at the relative rates of split and discordant reads in these types of variants. In addition, I am working on another way to mitigate this
type of noise that is upstream of variant-calling that I will blog about soon.

#### Future

We may also wrap additional tools such as a depth-based CNV caller (something I plan on looking into writing this year) or other SV callers that complement
what [lumpy](https://github.com/arq5x/lumpy-sv) reports. Suggestions on this are also welcome.
