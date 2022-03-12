---
layout: post
title:  Re-engineering a qsort function (part 2)
date:   2022-02-24 05:00:00 -0600
---

(Continued from [part 1](../../../../2022/01/17/Re-engineering-a-qsort-part-1/).)

I was starting to feel pretty good about my `qsort` implementation. I also downloaded a number of `qsort` implementations from various sources. Several were from `libc` and related implementations, including NetBSD, FreeBSD, OpenBSD, dietlibc, u-boot, μClibc and uclibc-ng, Newlib, picolibc, klibc, Bionic, and musl.

<!-- more -->

Several of these were derived from EASF by way of 4.4BSD-Lite. The u-boot, μClibc and uclibc-ng versions are actually slightly modified versions of a Shell sort I wrote over 30 years ago. The musl version is an implementation of E.W. Dijkstra's smoothsort, and klibc claims to be combsort, which I believe is a diminishing-increment sort (as in Shell sort) variant bubble sort.

My `rg22b` and `rg22c` performed well against all of them. I found that Newlib, picolibc, FreeBSD, Bionic and Reactos all go quadratic on certain inputs (primarily Bentley-McIlroy's "reverse front half" patterns), so I omitted them from tests here. The musl and klibc and the Shell sorts performed poorly enough that I omitted them from most subsequent tests.

I got results like this:

```
   Tot.rank     Swaps     Compares      Time   Ratio Implementation
 1.  204    115637080    544721860   8.111 s   1.000 rg22c
 2.  257    121272940    550721420   8.186 s   1.009 rg22b
 3.  599    118093240    585420940   9.042 s   1.115 bentley_mcilroy2
 4.  601            0    590927860  10.485 s   1.293 openbsd
 5.  640            0    585420940  10.646 s   1.313 bentmcil
 6.  654            0    590927860  10.586 s   1.305 netbsd
 7.  693    158006340    585420940  10.742 s   1.324 bentley_mcilroy
 8.  981            0    978597180  17.215 s   2.122 uclibcng
 9. 1087            0    862289174  16.191 s   1.996 dietlibc
10. 1146            0    978597180  17.902 s   2.207 uboot
11. 1151            0   1401141880  32.548 s   4.013 musl
12. 1347            0   3769242120  48.261 s   5.950 klibc
```

#### More about the test program

Then I found [Isabella Bosia's `qsortbench`](https://github.com/izabera/qsortbench). She collected several `qsort` implementations and wrote a nice benchmark to compare their performance against various data patterns. I liked the output format and the tests themselves, so I reimplemented the tests, plus a few more into my test program. I added a swap count to her output format. Her system ranks the results of each test over all the sorts, and then totals the rank for each sort in the summary at the end. I also added a summary of total compares. The ranking technique causes the final rankings to sometimes be different in order than the times or total compares for each sort implementation.

The EASF paper contained an outline of a "certification" program that was to be used more as a torture test than as a benchmark. It generates data in a variety of patterns known to give trouble to some quicksort implementations. The original used C int and double datatypes. I implemented my own version and added sorting of pointers and structs.

For sorting pointers, I represented the original integer values as strings and used pointers to the strings. I also artificially increased the effort of the comparison function by reversing one of the strings twice, to make comparisons more expensive. For structs, I use a 230 byte struct and store the string representation of the integer into the last 20 bytes. I padded the struct to make swaps more expensive.

Except for Bosia's own implementations, I did not use the sorts in her set. I obtained all the sorts shown here from primary sources. Also, unless otherwise indicated, these tests are using the EASF data patterns, 10,000 elements repeated 5 times, data types int, double, pointers, and structs.

I had not seen the Illumos system `qsort` before I found Bosia's `qsortbench`. I got a copy from the OpenIndiana site. My `rg22b` and `rg22c` did well against all the sorts found so far, but the version from the Illumos system did better on swaps:

```
   Tot.rank     Swaps     Compares      Time   Ratio Implementation
 1.  217    115637080    544721860   8.019 s   1.000 rg22c
 2.  256    121272940    550721420   8.081 s   1.008 rg22b
 3.  427    105373660    549881100   8.424 s   1.051 illumos
 4.  608            0    590927860  10.254 s   1.279 openbsd
 5.  673    118093240    585420940   8.914 s   1.112 bentley_mcilroy2
 6.  688            0    590927860  10.344 s   1.290 netbsd
 7.  710            0    585420940  10.530 s   1.313 bentmcil
 8.  747    158006340    585420940  10.578 s   1.319 bentley_mcilroy
 9. 1074            0    862289174  15.948 s   1.989 dietlibc
```

### Trying to get better

My versions ran a little faster, but I wondered how the Illumos version did so much better on swaps. I didn't want to look at the code to avoid copying theirs. I did grep for swap routines so I could insert swap counting, and saw that they use separate functions for different element sizes and set up pointers to the functions as needed.

I noticed that there were copious comments, so I grepped the comments out of the Illumos code and read what the author (I believe that is Saso Kiselkov) says about how the code works. Instead of swapping the pivot to a special location, he leaves it in place. The partitioning is done in a way that avoids swapping some elements that compare equal to the pivot, by sometimes changing the pivot pointer to refer to a different but equal-key element. This is rather hard to explain, but the code may make it clearer. I went through several attempts to get my function to match the Illumos code on compares and swaps, without referring to the Illumos code except to note how the pivot selection code and thresholds differed from EASF.

I ultimately got a function `rg22f` that matched the Illumos sort on compares and swaps, but it's not quite as fast as `rg22b` and `rg22c`, probably due to its more complex partitioning:

```
   Tot.rank     Swaps     Compares      Time   Ratio Implementation
 1.  227    115637080    544721860   8.029 s   1.000 rg22c
 2.  287    121272940    550721420   8.131 s   1.013 rg22b
 3.  372    105373660    549881100   8.338 s   1.038 rg22f
 4.  436    105373660    549881100   8.429 s   1.050 illumos
 5.  658            0    585420940  10.512 s   1.309 bentmcil
 6.  679    118093240    585420940   8.922 s   1.111 bentley_mcilroy2
 7.  701    158006340    585420940  10.584 s   1.318 bentley_mcilroy
```

I modified `rg22f` to use pointers to swap functions instead of the EASF-style conditional swap code, to get `rg22h`:

```
   Tot.rank     Swaps     Compares      Time   Ratio Implementation
 1.  268    115637080    544721860   8.185 s   1.000 rg22c
 2.  312    121272940    550721420   8.270 s   1.010 rg22b
 3.  382    105373660    549881100   8.401 s   1.026 rg22h
 4.  440    105373660    549881100   8.446 s   1.032 rg22f
 5.  540    105373660    549881100   8.547 s   1.044 illumos
 6.  782    118093240    585420940   9.123 s   1.115 bentley_mcilroy2
 7.  797            0    585420940  10.775 s   1.316 bentmcil
 8.  799    158006340    585420940  10.757 s   1.314 bentley_mcilroy
```

This runs a little faster than `rg22f` or the illumos version, with the same number of compares and swaps. But then I tweaked the pivot selection code to get `rg22i`. I also added tests to skip recursing or iterating on subfiles of less than two elements:

```
   Tot.rank     Swaps     Compares      Time   Ratio Implementation
 1.  338    115637080    544721860   8.081 s   1.000 rg22c
 2.  390    107019100    541185040   8.157 s   1.009 rg22i
 3.  391    121272940    550721420   8.161 s   1.010 rg22b
 4.  424    105373660    549881100   8.277 s   1.024 rg22h
 5.  538    105373660    549881100   8.402 s   1.040 rg22f
 6.  629    105373660    549881100   8.549 s   1.058 illumos
 7.  869            0    585420940  10.585 s   1.310 bentmcil
 8.  887    118093240    585420940   9.023 s   1.117 bentley_mcilroy2
 9.  934    158006340    585420940  10.671 s   1.321 bentley_mcilroy
```

This shows `rg22i` takes about .984 as many compares as illumos and about 1.016 times as many swaps. The tradeoff looks better with the "izabera" tests (test_sorts -z -r 5 100000), with about .95 as many compares and barely (1.00067 times) more swaps:

```
   Tot.rank     Swaps     Compares      Time   Ratio Implementation
 1.  176     73895300    470485920   6.918 s   1.000 rg22b
 2.  202     69813700    474913700   6.956 s   1.005 rg22c
 3.  227     56212300    467930740   7.064 s   1.021 rg22i
 4.  295     56174920    493061860   7.374 s   1.066 rg22h
 5.  366     56174920    493061860   7.470 s   1.080 rg22f
 6.  418     56174920    493061860   7.534 s   1.089 illumos
 7.  508     66704230    503202250   7.717 s   1.116 bentley_mcilroy2
 8.  513            0    503202250   8.189 s   1.184 bentmcil
 9.  535     78515800    503202250   8.263 s   1.194 bentley_mcilroy
 ```

But note that despite these improvements, `rg22b` and `rg22c` still ran faster.

More work on my version is covered in [part 4](../../../../2022/02/27/Re-engineering-a-qsort-part-4/). Before getting to that, I want to digress a bit on the BSD legacy of EASF in [part 3](../../../../2022/02/26/Re-engineering-a-qsort-part-3/).

(See also: [part 1](../../../../2022/01/17/Re-engineering-a-qsort-part-1/), [part 5](../../../../2022/03/09/Re-engineering-a-qsort-part-5/))
