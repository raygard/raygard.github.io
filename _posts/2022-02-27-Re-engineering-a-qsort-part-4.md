---
layout: post
title:  Re-engineering a qsort function (part 4)
date:   2022-02-27 05:00:00 -0600
---

### Aliasing concerns

While looking at commit logs for various BSD `qsort` versions, I noticed [this](https://cgit.freebsd.org/src/log/lib/libc/stdlib/qsort.c):

```
*   libc qsort(3): stop aliasing.   Konstantin Belousov 2018-06-10  1   -44/+17
```

Which led to [some details](https://cgit.freebsd.org/src/commit/lib/libc/stdlib/qsort.c?id=6609261660989cac8bbb8acbf94d4e80d8c59c70):

<!-- more -->

```
libc qsort(3): stop aliasing.
Qsort swap code aliases the sorted array elements to ints and longs in
order to do swap by machine words.  Unfortunately this breaks with the
full code optimization, e.g. LTO.

See https://gcc.gnu.org/bugzilla/show_bug.cgi?id=83201 which seems to
reference code directly copied from libc/stdlib/qsort.c.
```

Which in turn led to [this, at the gcc development site](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=83201):

```
505.mcf_f produces incorrect output when built with both LTO/FDO. Using either
option separately is fine. GCC trunk r255207 was used. Following are options
used.

[...]

Likely invalid.  spec_qsort is full of alias violations.  We sort

typedef struct basket
{
    arc_t *a;
    cost_t cost;
    cost_t abs_cost;
    LONG number;
} BASKET;

and spec_qsort does stuff like

[...]

#define swap(a, b)                              \
        if (swaptype_long == 0) {               \
                long t = *(long *)(a);          \
                *(long *)(a) = *(long *)(b);    \
                *(long *)(b) = t;               \
        } else if (swaptype_int == 0) {         \
                int t = *(int *)(a);            \
                *(int *)(a) = *(int *)(b);      \
                *(int *)(b) = t;                \
        } else                                  \
                swapfunc((char *)a, (char *)b, es, swaptype_long, swaptype_int)

eh... (no, swapfunc isn't any better)

[...]

I just looked at the source seeing *(long *) ptr and bells went off.

```

So the GCC team found that a certain benchmark produced anomalous resuts when both LTO (Link Time Optimization) and FDO (Feedback-Directed Optimization) were used, and attributed it to aliasing issues. The rules in sec. 6.5 of the C99 and C11 standards appear to say that accessing the elements being sorted as `long`, `int`, or sequences of `long` or `int` violate aliasing rules, at least if the actual data being sorted are not of exactly those types.

As a result, the FreeBSD team removed all the special swapping code (derived from the Bentley-McIlroy EASF paper) from their `qsort` implementation, leaving only code that swaps elements a byte at a time.

I have made `qs22k.c` by modifying `qs22j.c` to use `memcpy()` to do all the swapping. I also made modified versions `qs22bk.c` and `qs22ck.c` from `qs22b.c` and `qs22c.c`, and ran some more tests. I also included the system sort, which is apparently from glibc. Here are some results:

```
   Tot.rank     Swaps     Compares      Time   Ratio Implementation
 1.  430   1295628260   6360932820  97.304 s   1.000 qs22j
 2.  447   1295628260   6360932820  97.666 s   1.004 qs22k
 3.  553   1474008500   6888529840 102.664 s   1.055 qs22bk
 4.  574   1408957420   6901692000 103.303 s   1.062 qs22ck
 5.  592   1616259440   7927606400 116.397 s   1.196 qs22b
 6.  760   1565906240   7960573720 118.266 s   1.215 qs22c
 7.  776   1459080660   7891557400 118.475 s   1.218 qs22i
 8.  827   1448442400   7917118940 118.951 s   1.222 qs22h
 9. 1157   1448442400   7917118940 121.839 s   1.252 illumos
10. 1177   1448442400   7917118940 122.455 s   1.258 qs22f
11. 1262            0   8221568620 150.855 s   1.550 bentmcil
12. 1321   1585231370   8221568620 125.109 s   1.286 bentley_mcilroy2
13. 1342            0  11193777940 173.175 s   1.780 system
14. 1382   2057439810   8221568620 144.101 s   1.481 bentley_mcilroy
```

And the corresponding "izabera" (I. Bosia) tests:

```
   Tot.rank     Swaps     Compares      Time   Ratio Implementation
 1.  372    845862200   5800329540  84.991 s   1.000 qs22b
 2.  373    664027320   4882687260  74.581 s   0.878 qs22j
 3.  379    770930420   5482457360  81.157 s   0.955 qs22ck
 4.  388    664027320   4882687260  74.881 s   0.881 qs22k
 5.  389    823141500   5562942420  81.829 s   0.963 qs22bk
 6.  440            0   4657468760  77.355 s   0.910 system
 7.  446    795927280   5749257820  85.320 s   1.004 qs22c
 8.  451    664035420   5713784460  86.051 s   1.012 qs22i
 9.  495    663370440   5824862140  87.337 s   1.028 qs22h
10.  675    663370440   5824862140  89.244 s   1.050 illumos
11.  720    663370440   5824862140  90.345 s   1.063 qs22f
12.  789    777632180   6128745270  93.485 s   1.100 bentley_mcilroy2
13.  790            0   6128745270 100.835 s   1.186 bentmcil
14.  853    898924590   6128745270  99.273 s   1.168 bentley_mcilroy
```

So we see that the `memcpy` versions are not as fast as the pointers-to-swap-functions versions, but still perform respectably. The `memcpy` here is intrinsic in GCC on Intel. I suspect it is much slower if it is an external function. But if you need to be concerned about possible aliasing problems, it may be a good option.

On the other hand, though it is undefined according to the standard, I have to wonder if the actual aliasing problem exposed by the GCC benchmark described above was actually triggered by the swapping function being called with pointers to the same location. Recall that `bentley_mcilroy` and my `qs22a` both do call for swapping an element with itself, and `bentley_mcilroy2` and my own later versions avoid this.

I would welcome some way of testing if GCC can be made to display an actual aliasing problem on `qs22b` or later versions.

(See also: [part 1](../../../../2022/01/17/Re-engineering-a-qsort-part-1/), [part 2](../../../../2022/02/24/Re-engineering-a-qsort-part-2/), [part 3](../../../../2022/02/26/Re-engineering-a-qsort-part-3/), [part 5](../../../../2022/03/09/Re-engineering-a-qsort-part-5/))
