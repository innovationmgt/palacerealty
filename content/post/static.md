---
title: "Static"
date: 2019-02-28T11:32:22-07:00
lastmod: 2019-02-28T11:32:22-07:00
draft: false
keywords: []
description: ""
tags: []
categories: []
author: ""

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

After recent twitter discussions about usable software, I thought that one thing I could do to
relieve a common problem of users of my software would be to build static binaries.
I know this would be helpful because I never get any questions/bugs about installs with 
[vcfanno](https://github.com/brentp/vcfanno) or [indexcov](https://github.com/brentp/goleft)
for which I provide static binaries (thanks to use of #golang) even though they are (I think) widely used.

Since most of my tools are command-line apps, that rely on htslib this can be difficult. I ended up
using a docker image based on alpine linux and building an htslib without libcurl. Alpine linux uses
musl so we can avoid the warnings/errors seen when trying to use a binary
built with a new version of (g)libc on a system with an older (g)libc.

The docker image is [musl-hts-nim](https://hub.docker.com/r/brentp/musl-hts-nim) and, using that, I made
a static binary so that, given a nim file, for example [this one](https://github.com/brentp/hts-nim-tools/issues/5#issuecomment-462097499) where I recently helped Ming (note in that issue, Ming had problems with libhts.so which would have been avoided with a static binary), we can
create a static binary with:

{{< highlight shell >}}

$ hts_nim_static_builder -s ./atac.nim --deps "hts@>=0.2.7" --deps "binaryheap"
#
{{< /highlight >}}


This will create a static binary in `atac`:
{{< highlight shell >}}

$ ldd ./atac
    statically linked
$ file ./atac
    ./atac: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, not stripped
#
{{< /highlight >}}

`hts_nim_static_builder` is avaiable [here](https://github.com/brentp/hts-nim/releases/tag/v0.2.8)

See the [README section](https://github.com/brentp/hts-nim#static-builds) for more information and advanced usage.

I hope that this will make using [hts-nim](https://github.com/brentp/hts-nim) even more enticing for building command-line tools
that utilize htslib.
