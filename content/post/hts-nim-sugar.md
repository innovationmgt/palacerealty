---
title: "Hts Nim Sugar"
date: 2019-01-25T10:15:01-07:00
lastmod: 2019-01-25T10:15:01-07:00
draft: false
keywords: ["nim"]
description: ""
tags: ["nim", "hts"]
categories: ["nim"]
author: ""

comment: true
toc: false
autoCollapseToc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
---

[hts-nim](https://github.com/brentp/hts-nim) is a library that allows one to use [htslib](https://www.htslib.org/) via the [nim](https://nim-lang.org) programming language.
Nim is a garbage-collected language that compiles to C and often has similar performance. 
I have become very productive in `nim` and especially in `hts-nim` and there are by now, at least a few other users of `hts-nim`.
This post is to show how one particular feature of nim enables users to write their own functions that will be used no differently than
`hts-nim`'s provided functionality.

Currently, in order to access the format (sample) fields in a VCF, one must first allocate a `seq` (something like a typed python list) that gets filled.
The example below shows a code stub to extract the `DP` (depth) and `GQ` (genotype quality) fields for each sample into those pre-allocated `seq`s.

{{< highlight nim >}}
var dps = newSeq[int32]()
var gqs = newSeq[int32]()
for variant in vcf:
    doAssert variant.format.get("DP", dps) == Status.OK
    doAssert variant.format.get("GQ", gqs) == Status.OK
    # do stuff with dps and gqs ...
{{< /highlight >}}

This lets `hts-nim` fill the `dps` and `gqs` sequences without allocating memory to provides maximal
performance. It's not much of a burden, but it might be nice to have a function that just returns the values and raises
an error if the field is not found so that pre-allocation is not necesary. 

Since nim has [universal function call syntax (UFCS)](https://en.wikipedia.org/wiki/Uniform_Function_Call_Syntax),
it's easy to write some *sugar* that can be used to get values without pre-allocating if that's what's required.
Here's how I'd write that (though see note at end of post).

{{< highlight nim >}}
proc ints*(f:FORMAT, field: string): seq[int32] =
  var vals = newSeq[int32]()
  if variant.format.get(field, vals) != Status.OK:
    raise newException(KeyError, "error getting field:" & field & ". " & $Status.OK
  return vals
{{< /highlight >}}


This will throw an exception if the requested field does not exist. Note that the `*` after `ints` means
the function is exported for use outside the currrent module, which is what we want here.

This function could be in a file named `sugar.nim` and then could be used as:

{{< highlight nim >}}
import hts/vcf
import ./sugar

for variant in vcf:
    var dps = variant.format.ints("DP") # note these could raise exceptions.
    var gqs = variant.format.ints("GQ")
    # do stuff with dps and gqs ...
{{< /highlight >}}

This shows the `UFCS` as `ints(variant.format, "DP")` is equivalent to `variant.format.ints("DP")`.

This means that users are not limited to the syntax and functionality that `hts-nim` provides, they can
write their own functions and use them as ergonomically as those that are part of hts-nim. Since hts-nim
errs on the side of efficiency rather than ergonomics, functions like these could make some nice syntactic 
sugar for using `hts-nim` as a sort of scripting language for processing genomic data.


## side note.

The above sugar function `ints` could be written more succinctly using nim`s `result` variable along with the
fact that `seq`s are never `nil` as:

{{< highlight nim >}}
proc ints*(f:FORMAT, field: string): seq[int32] =
  if variant.format.get(field, result) != Status.OK:
    raise newException(KeyError, "error getting field:" & field & ". " & $Status.OK
{{< /highlight >}}

where result will be initalized by nim and set to the proper size (and filled) by hts-nim.
