---
title: "Noise"
date: 2018-03-14T19:35:37-06:00
draft: false
toc: false
tags: ["noise", "python", "variant"]
categories: ["Genomics"]
---

This afternoon, I did a quick analysis to attempt to find variants on the X chromosome that are under
recessive constraint--that is that they appear in some non-zero frequency in females but never
occur in male. That is, without an extra copy, a variant might be embryonic lethal in males, but
could be seen in females thanks to a backup copy. I thought that these might be occurring at a
relatively high allele frequency (greater than 0.001). I encountered some surprises.

A reasonable first pass is to filter to variants on X that have:

1. 0 homozygous alternate samples in females (or males)
2. 0 heterozygous alternate samples in males.
3. some number of heterozygous sites in females.

This information is easily extracted from the [gnomad exomes VCF](http://gnomad.broadinstitute.org/downloads)
with 120K samples
as it reports `GC_Male` and `GC_Female`
which, for bi-allelic variants, have 3 values indicating the number of samples that
are respectively homozygous reference, heterozygous, and homozygous alternate.

I created an expected proportion of alternate alleles (not samples) using the females and then used that
as the expected success rate for testing if males were depleted for the allele. Since gnomad also reports
the total number of alleles in males and females (accounting already for the fact that males only have 1 allele),
it's simple to put this into a binomial test


```python
import scipy.stats as ss
from cyvcf2 import VCF
vcf = VCF("gnomad.vcf.gz")
for v in vcf("X:2781479-155701382"): # exclude PAR
    gcm = v.INFO["GC_Male"]
    gcf = v.INFO["GC_Female"]
    success_prob = gcf[1] / float(v.INFO["AN_Female"])
    # 0 successes == 0 males with an alternate out of AN_Male alleles with p defined by females.
    p = ss.binom_test(0, v.INFO["AN_Male"], p=success_prob)
...
```
There are other ways to do this, but I figured this was a reasonable check to start

After requiring a p-value of < 1e-10 to more than account for multiple testing, more than 500 variants popped
out. 

[here](http://gnomad.broadinstitute.org/variant/X-14937753-T-G) is one of the top variants. It has an allele frequency
of 0.01553 but has never been seen as a homozygous alternate in 147,273 sampled alleles.

What Aaron noticed after I sent a message to slack indicating that I'd solved the genome was that it always occurs on 
the red (forward) reads in the alignment browser at the bottom of that page. Although this variant has PASS in the FILTER
field it has an FS of 172.927. `FS` is "Phred-scaled p-value using Fisher's exact test to detect strand bias". High is bad.

After requiring an FS <= 10, the number dropped dramatically, but there were still many variants. The next filter was to
require that the variant appear in gnomad whole genomes as we noticed that many of the candidates did not appear in whole genomes.
To have a high enough p-value to pass the 1e-10 threshold, the allele frequency should be high enough to appear in 12K samples.
After this, we also found that the remaining variants were frequently filtered in gnomad genomes VCF so we required them to be
present in the genomes, not flagged, and we again checked that there were 0 male heterozygotes or homozygous alternates in the
whole genomes as well. After this **7 variants remained**.

I don't inted to pursue this, but it was a fun afternoon's tinkering. Here are those variants in case anyone has some
insight:

```
X	2836140	rs202219772	G	A
X	48270249	rs377690870	G	A
X	101473058	rs782534766	A	G
X	119293216	rs782542106	C	CG
X	119293240	rs199940228	G	A
X	134988581	rs145404090	A	G
X	153461539	.	C	T
```

[here is the script used to find these](https://gist.github.com/brentp/e8b400bf8bfb297c8d49f9673e32d2e4#file-male-depleted-x-py)
