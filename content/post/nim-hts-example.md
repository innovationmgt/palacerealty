---
title: "Nim Hts Example"
date: 2018-10-01T13:17:00-06:00
draft: false
keywords: ["nim"]
description: ""
tags: ["nim-mundane"]
categories: ["nim"]
author: "Brent"
---

Several folks have recently expressed interest in learning [nim](https://nim-lang.org) which
I have found to be very useful for genomics because it has a simple syntax like python, but
it compiles to be as fast as C. In the case of [mosdepth](https://github.com/brentp/mosdepth) which
is written in nim, it is faster than the competing C implementations because of choice of 
algorithm.

I have started using `nim` in my day-to-day scripting to replace python, in part, so this will be
the first in a series of posts that show some relatively mundane code that I write like a script but
will compile to run very fast. Hopefully, this can be informative for people who want to start doing
some genomics data-processing with `nim`.

Today, I am comparing outputs from 2 different softwares for extracting split and discordant reads
for input to lumpy. They produce very similar, but not identical results. I want to better understand
how they differ. They make relatively small output bams, so the process is to

1. read each bam into a table (like a python dict) keyed by read-name + read flag and with value of
   each alignment
2. find reads that are in 1 table and not another
3. print the flag since that is the main "decider" of what's included in the output.

Here is the code with comments:

{{< highlight nim >}}
import tables
import os
import hts

var
 abam:Bam
 bbam:Bam


# two bam paths are sent in as arguments
open(abam, commandLineParams()[0]) 
open(bbam, commandLineParams()[1])

# define a simple function to use as a key in the table
proc key(aln:Record): string =
  return aln.qname & "//" & $aln.flag


# fill a table with the key defined above and the value of the record
proc fill_table(bam:Bam): TableRef[string,Record] =
  result = newTable[string, Record]()
  for aln in bam:
    # have to copy since the bam parser re-uses the underlying pointer during iteration
    result[aln.key] = aln.copy()


# get a table for each bam.
var atable = fill_table(abam)
var btable = fill_table(bbam)

# get a seq (list) of Records that are in atbl, but not in btbl
proc diff(atbl, btbl: TableRef[string,Record]): seq[Record] =
  ## return the alignments in a, but not in b
  for k, aln in atbl:
    if not (k in btbl):
      result.add(aln)

# print out the differences. note that UFCS let's us
# use atable.diff(btable) which is equivalent to btable.diff(atable)
for aln in atable.diff(btable):
  echo aln.flag
  # aln.flag is a uint16, but it has a string method defined on it so this
  # will print, e.g.: PAIRED,REVERSE,MREVERSE,READ2,SUPPLEMENTARY
  # indicating which bits are set in the flag

{{< /highlight >}}

note the `flag` is actually a `uint16` but can still print as an informative string.
This uses a method on the flag similar to python's `__repr__` or `__str__`.

This gives a starting-point for looking into the problem at hand.
For more info on using [hts-nim](https://github.com/brentp/hts-nim), have a look at the [docs](https://brentp.github.io/hts-nim/)

