---
title: "Using Nim to count sequence-motifs in a BAM"
date: 2018-10-04T09:23:24-06:00
lastmod: 2018-10-04T09:23:24-06:00
draft: false
keywords: ["nim"]
description: ""
tags: ["nim-mundane"]
categories: ["nim"]
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

This is the 2nd post describing mundane uses of [nim](https://nim-lang.org) in day-to-day genomics. 

The first post is [here](https://brentp.github.io/post/nim-hts-example/).

For today's mundane task, a colleague asked me to count the occurence of 2 k-mers of length 39 in
each of 603 BAMs of 60X coverage. We don't care about partial matches or allowing mismatches so this
is a pretty simple task. There are more efficient computational methods for this, but I wrote the
simplest version, verified that it ran in < 2 hours for a single bam and then ran it overnight on
a single machine with 64 CPUs and sent the results in the morning.

The kmers we actually used are redacted to protect the innocent.

The code does the following:

+ implements a reverse-complement function and uses it on the 2 query sequences.
+ opens a BAM file for the path that is given as an argument to the program.
+ iterates over the bam and counts the presence of each forward or reverse-complemented kmer.
+ prints out the bam and the count of times each kmer was observed.

Again, there are better algorithms for sequencer- matching (though the string-matching implementation in nim must be pretty efficient),
but this took about 15 minutes to code and was fast enough to set running. Here is the code with explanatory comments:

{{< highlight nim >}}
import os
import strutils
import hts
import kmer

var aseq = "some other dna sequence".toUpperAscii
var bseq = "some other DNA sequence".toUpperAscii

# write a quick reverse-complement function
proc complement(s:char): char {.inline.} =
    if s == 'C':
        return 'G'
    elif s == 'G':
        return 'C'
    elif s == 'A':
        return 'T'
    elif s == 'T':
        return 'A'
    else:
        return s

proc reverse_complement(xs: string): string =
  result = newString(xs.len)
  for i, x in xs:
    # high == len - 1
    result[xs.high-i] = complement(x)

var raseq = aseq.reverse_complement
var rbseq = bseq.reverse_complement

var path = commandLineParams()[0]
var bam: Bam
open(bam, path, threads=2)

var acount = 0
var bcount = 0

# this gets filled with the sequence for each read.
var sequence: string
for aln in bam:
    # check flags to avoid double-counting
    if aln.flag.dup: continue
    if aln.flag.supplementary: continue
    # fill the `sequence` string with the values from this alignment
    discard aln.sequence(sequence)

    acount += sequence.count(aseq)
    acount += sequence.count(raseq)

    bcount += sequence.count(bseq)
    bcount += sequence.count(rbseq)

var base = path.split('/')
# just print the final file, not the full directory
echo base[base.high] & "\t" & $acount & "\t" & $bcount
#
{{< /highlight >}}
