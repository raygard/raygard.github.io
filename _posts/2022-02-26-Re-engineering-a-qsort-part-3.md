---
layout: post
title:  Re-engineering a qsort function (part 3)
date:   2022-02-26 05:00:00 -0600
---

### The qsort optimization that wasn't

In 1993, Jon Bentley and Doug McIlroy published a paper [Engineering a Sort Function](https://cs.fit.edu/~pkc/classes/writing/samples/bentley93engineering.pdf) (also [here](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.14.8162&rep=rep1&type=pdf)) (Software--Practice and Experience, vol. 23 no. 11, Nov. 1993, pp 1249-1265). Since then, most C `stdlib` `qsort` functions have been based on the code and/or the ideas in that paper.

The `qsort` in the BSD variants and descendants have followed the code in that paper. But someone in 1994 attempted to "improve" on Bentley and McIlroy, with potentially disastrous results.

<!-- more -->

The NetBSD code prior to 1994, in [NetBSD qsort 1.1](http://cvsweb.netbsd.org/bsdweb.cgi/src/lib/libc/stdlib/qsort.c?rev=1.1&content-type=text/x-cvsweb-markup) (copied from [386BSD](https://github.com/386bsd/386bsd/raw/2.0/usr/src/lib/libc/stdlib/qsort.c)), contained a `qsort` that attempts to follow the quicksort discussion in Donald Knuth's *The Art of Computer Programming, volume 3, Sorting and Searching* (1973, first edition). I believe the author would have been better off following the advice in the later paper by Robert Sedgewick, [Implementing Quicksort Programs (Communications of the ACM, vol 21 no 10, Oct. 1978 pp 847–857)](https://www.csie.ntu.edu.tw/~b93076/p847-sedgewick.pdf). Sedgewick was Knuth's PhD student and had done his thesis on quicksort.

The comments in the 386BSD code imply to me that some of Knuth was misunderstood. For example: "Knuth, Vol. 3, page 116, Algorithm Q, step b, argues that a single pass of straight insertion sort after partitioning is complete is better than sorting each small partition as it is created."

Oddly, Knuth never suggests a single pass of insertion sort instead of sorting each small partition; that was suggested later by Sedgewick. Knuth step b says "Subfiles of M or fewer elements are sorted by straight insertion..." and details doing that in step Q8, page 117.

A later comment says "Insertion sort has the same worst case as most simple sorts (O N^2).  It gets used here because it is (O N) in the case of sorted data." It is true that it is O(N<sup>2</sup>) worst case and O(N) if the data is sorted, but that is not why it is used with quicksort. It is used because it is faster for small subsections than quicksort due to its lower "bookkeeping" overhead, per Knuth on page 116.

The 386BSD routine departed from Knuth in an important respect: its partitioning code "jumped over" equal elements instead of stopping and swapping them. Bentley and McIlroy noted: "Shopping around for a better `qsort`, we found that a `qsort` written at Berkeley in 1983 would consume quadratic time on arrays that contain a few elements repeated many times--in particular arrays of random zeros and ones." And later: "The Berkeley qsort takes quadratic time on random zeros and ones precisely because it scans over equal elements..."

The author also did something else not found in Knuth, however. There is this comment:

```C
	 * Quicksort behaves badly in the presence of data which is already
	 * sorted (see Knuth, Vol. 3, page 119) going from O N lg N to O N^2.
	 * To avoid this worst case behavior, if a re-partitioning occurs
	 * without swapping any elements, it is not further partitioned and
	 * is insert sorted.  This wins big with almost sorted data sets and
	 * only loses if the data set is very strangely partitioned.  A fix
	 * for those data sets would be to return prematurely if the insertion
	 * sort routine is forced to make an excessive number of swaps, and
	 * continue the partitioning.
```

(Knuth does not say this on page 119; he does say on page 122 "If the original file is already in order ... this situation ... makes quicksort anything but quick; its running time becomes proportional to N<sup>2</sup> instead of N log N." This refers to a version that chooses the first element as the pivot. But Knuth describes ways of fixing the situation, including choosing the pivot at random or choosing the median of first, middle, and last elements. And the 386BSD routine does use the median-of-3 method.)

So the code includes a flag `didswap` which is set if the partitioning pass does any swapping, and then uses an insertion sort on the partition if `didswap` is false. This "optimization" loses very big if the data is "very strangely partitioned". This is probably the cause of this issue called out in Bentley and McIlroy: "...several other problems with the Berkeley code. (For instance, a mod-4 ‘sawtooth’ with its front half reversed ran a factor of 8 slower than expected.)" Also, the comment blithely suggests "A fix for those data sets would be to return prematurely if the insertion sort routine is forced to make an excessive number of swaps" without indicating how that might be done without too much code or computation.

The Bentley-McIlroy paper reported that "In any event, the new code represents enough of an improvement to have been adopted in our own lab and by Berkeley." But someone at Berkeley failed to read and understand the paper. The [first FreeBSD `qsort` (1994-05-27)](https://cgit.freebsd.org/src/tree/lib/libc/stdlib/qsort.c?id=58f0484fa251c266ede97b591b499fe3dd4f578e) based on the Bentley-McIlroy paper, (also in [NetBSD revision 1.1.1.2 (1994-06-16)](http://cvsweb.netbsd.org/bsdweb.cgi/src/lib/libc/stdlib/qsort.c?rev=1.1.1.2&content-type=text/x-cvsweb-markup) and [OpenBSD revision 1.1 (1995-10-18)](https://cvsweb.openbsd.org/src/lib/libc/stdlib/qsort.c?rev=1.1&content-type=text/x-cvsweb-markup)) was copied from [4.4BSD-Lite](http://http.pl.scene.org/pub/unix/systems/BSD/4.4BSD-Lite/). It has a variable `swap_cnt` that serves the same purpose as `didswap` in the earlier `qsort`. It's misnamed, because it's a 0/1 flag, not a counter. Then there is:

```C
	if (swap_cnt == 0) {  /* Switch to insertion sort */
		for (pm = a + es; pm < (char *) a + n * es; pm += es)
			for (pl = pm; pl > (char *) a && cmp(pl - es, pl) > 0;
			     pl -= es)
				swap(pl, pl - es);
		return;
	}
```

This causes exactly the problem with the sawtooth pattern mentioned above.

This problem was noticed a long time ago. See, e.g., [this thread by postgresql developers](https://postgrespro.com/list/thread-id/1675468). Apparently the "very strangely partitioned" data happened in the real world:

```
qsort, once again
From:Tom Lane
Date:16 March 2006, 18:37:58

I was just looking at the behavior of src/port/qsort.c on the test case
that Jerry Sievers was complaining about in pgsql-admin this morning.
I found out what the real weak spot is: it's got nothing directly to do
with good or bad pivots, it's this code right here:
   if (swap_cnt == 0)   {                            /* Switch to insertion sort */
   [... code shown above -- rdg ...]
   }
In other words, if qsort hits a subfile for which the chosen pivot is a
perfect pivot (no swaps are necessary), it switches to insertion sort.
Which is O(N^2).  In Jerry's test case this happens on a subfile of
736357 elements, and you can say goodnight to that process ....
```

This `swap_cnt` code was in place in NetBSD until `qsort` revision 1.20, 2009-06-01, about 15 years later. (Commit message: "qsort: remove the "switch to insertion sort" optimization because it causes catastrophic performance for certain inputs.")

It remained in OpenBSD until `qsort` revision 1.12, 2014-06-12, about 20 years later. (Commit message: "Disable the "switch to insertion sort" optimization to avoid quadratic behavior for certain inputs.  From NetBSD."

As of this writing, it remains in FreeBSD and in several other `libc` distributions, e.g. Newlib, picolibc, bionic, reactos.

The postgresql team apparently decided to use their own `qsort` exclusively. It includes a different optimization: at the beginning of each partitioning operation, it checks to see if the data is already in order, and skips sorting that subfile if it is already sorted. (Some members of the group are skeptical of this check.)

It also occurred to me that perhaps a Shell sort in place of the `if (swap_cnt == 0)` insertion sort would improve performance.

I took several versions that include the `swap_cnt` code and removed that code, then tried adding a pre-sorted test, a Shell sort in place of the `if (swap_cnt == 0)` insertion sort, and both the pre-sorted test and the Shell sort. These versions are named `nnn_nopt`, `nnn_pre`, `nnn_shell`, and `nnn_pre_shell` for the no-swap-count-optimization, pre-sort-check, Shell sort, and pre-sort-plus-Shellsort versions, where nnn is the name of the original version (freebsd, reactos, bionic). And I added `bentley_mcilroy_pre`, `bentley_mcilroy_shell`, and `bentley_mcilroy_pre_shell` versions as well.

I added a pre-check for sorted data to `rg22i` to get `rg22j`, and used that in the following tests.

Here are some results (10000 elements, 5 reps):

```
   Tot.rank     Swaps     Compares      Time   Ratio Implementation
 1.  377     94128580    459721400   6.806 s   1.000 rg22j
 2.  705    107019100    541185040   8.011 s   1.177 rg22i
 3.  817    111294680    508438960   7.933 s   1.165 reactos_pre_shell
 4.  918    111244760    508450060   7.935 s   1.166 bentley_mcilroy_pre_shell
 5.  955            0    508438960   8.064 s   1.185 freebsd_pre_shell
 6. 1003            0    535753740   8.258 s   1.213 freebsd_pre
 7. 1010    109557610    515009880   7.992 s   1.174 bentley_mcilroy_pre
 8. 1053            0    508438960   8.206 s   1.206 bionic_pre_shell
 9. 1124    108492120    535753740   8.295 s   1.219 reactos_pre
10. 1305    159348860    539955580  10.402 s   1.528 bentley_mcilroy_shell
11. 1326            0    535753740   8.377 s   1.231 bionic_pre
12. 1380    159381080    539941060  10.103 s   1.484 reactos_shell
13. 1487    118093240    585420940   8.908 s   1.309 bentley_mcilroy2
14. 1536            0    539941060  10.134 s   1.489 freebsd_shell
15. 1545            0    585420940  11.149 s   1.638 bentmcil
16. 1630            0    539941060  10.196 s   1.498 bionic_shell
17. 1680    155190940    590927860  10.575 s   1.554 reactos_nopt
18. 1727    158006340    585420940  10.597 s   1.557 bentley_mcilroy
19. 1754            0    590927860  10.648 s   1.564 freebsd_nopt
20. 1867            0    590927860  11.162 s   1.640 bionic_nopt
```

And here are results from the "izabera" (Isabella Bosia) tests:

```
   Tot.rank     Swaps     Compares      Time   Ratio Implementation
 1.  299     56211700    402119340   5.971 s   1.000 rg22j
 2.  441     56212300    467930740   6.952 s   1.164 rg22i
 3.  591            0    492160640   7.356 s   1.232 freebsd_pre
 4.  594     72794660    480684760   7.396 s   1.239 reactos_pre_shell
 5.  654            0    480684760   7.455 s   1.249 freebsd_pre_shell
 6.  685     72798140    480673320   7.437 s   1.246 bentley_mcilroy_pre_shell
 7.  698     66451700    492160640   7.412 s   1.241 reactos_pre
 8.  706     67278940    465405160   6.992 s   1.171 bentley_mcilroy_pre
 9.  729            0    480684760   7.541 s   1.263 bionic_pre_shell
10.  771     84726960    461110580   7.793 s   1.305 bentley_mcilroy_shell
11.  783            0    492160640   7.502 s   1.256 bionic_pre
12.  786     84720640    461115400   7.700 s   1.290 reactos_shell
13.  840            0    461115400   7.791 s   1.305 freebsd_shell
14.  870     76082840    506126400   8.012 s   1.342 reactos_nopt
15.  900            0    503202250   8.304 s   1.391 bentmcil
16.  906            0    506126400   8.145 s   1.364 bionic_nopt
17.  925            0    506126400   8.078 s   1.353 freebsd_nopt
18.  934            0    461115400   7.878 s   1.319 bionic_shell
19.  942     66704230    503202250   7.788 s   1.304 bentley_mcilroy2
20. 1065     78515800    503202250   8.271 s   1.385 bentley_mcilroy
```

Does the `swap_cnt` code improve the performance if we avoid the cases where it goes quadratic? Here are results including the original versions of bionic, freebsd, newlib, nlopt, picolibc, and reactos:

```
   Tot.rank     Swaps     Compares      Time   Ratio Implementation
 1.  345    812425500   3995362140  62.996 s   1.000 rg22j
 2.  579    934827860   4255667160  70.232 s   1.115 reactos_pre_shell
 3.  675    934744100   4255449220  70.785 s   1.124 bentley_mcilroy_pre_shell
 4.  707            0   4255667160  72.269 s   1.147 bionic_pre_shell
 5.  750    942426500   5280619020  80.768 s   1.282 rg22i
 6.  836            0   4462256520  85.932 s   1.364 picolibc
 7.  874            0   4255667160  73.264 s   1.163 freebsd_pre_shell
 8.  875    921501620   4550322480  73.574 s   1.168 reactos_pre
 9.  895            0   4462256520  86.235 s   1.369 newlib
10.  923            0   4550322480  75.254 s   1.195 freebsd_pre
11.  952            0   4550322480  75.365 s   1.196 bionic_pre
12.  957    925527200   4324550240  71.875 s   1.141 bentley_mcilroy_pre
13. 1018   1671482800   4462256520  86.726 s   1.377 reactos
14. 1097            0   4462256520  86.923 s   1.380 netbsd_1_4
15. 1268   1024111730   5490627380  85.876 s   1.363 bentley_mcilroy2
16. 1281            0   4462256520  90.780 s   1.441 bionic
17. 1328   1354974780   5048315340  95.209 s   1.511 reactos_shell
18. 1380   1354903560   5047593740  94.987 s   1.508 bentley_mcilroy_shell
19. 1413            0   4462256520  91.197 s   1.448 freebsd
20. 1431            0   4462256520  95.693 s   1.519 nlopt
21. 1446            0   5490627380 104.098 s   1.652 bentmcil
22. 1477   1334927640   5490627380  98.820 s   1.569 bentley_mcilroy
23. 1507            0   5532021360  99.464 s   1.579 freebsd_nopt
24. 1511   1318873580   5532021360  99.074 s   1.573 reactos_nopt
25. 1530            0   5048315340  97.048 s   1.541 freebsd_shell
26. 1573            0   5532021360 100.056 s   1.588 bionic_nopt
27. 1612            0   5048315340  97.321 s   1.545 bionic_shell
```

In most cases, adding only the Shell sort resulted in somewhat slower overall performance, but adding the pre-sort check and the Shell sort give better performance than the `swap_cnt` versions.

But including the cases that cause quadratic performance give clearly bad results for the `swap_cnt` sorts (ordered here by time rather than finishing rank):

```
   Tot.rank     Swaps     Compares      Time   Ratio Implementation
 1.  578     41547940    200606648   3.256 s   1.000 rg22j
 2.  929     49860980    219465656   3.699 s   1.136 reactos_pre_shell
 3.  973     49843156    219434364   3.717 s   1.141 bentley_mcilroy_pre_shell
 5. 1192            0    219465656   3.801 s   1.167 bionic_pre_shell
 8. 1346     47973632    222390056   3.874 s   1.190 bentley_mcilroy_pre
 6. 1308     47641164    232805696   3.885 s   1.193 reactos_pre
 4. 1098     47050340    246166168   3.914 s   1.202 rg22i
 9. 1370            0    232805696   4.024 s   1.236 freebsd_pre
 7. 1331            0    219465656   4.035 s   1.239 freebsd_pre_shell
10. 1440            0    232805696   4.092 s   1.257 bionic_pre
14. 1738     51969562    263591438   4.345 s   1.334 bentley_mcilroy2
15. 1766     70217484    240903484   4.722 s   1.450 reactos_shell
17. 1847     70204772    240834664   4.805 s   1.475 bentley_mcilroy_shell
22. 2153            0    240903484   4.877 s   1.498 bionic_shell
21. 2128            0    240903484   4.940 s   1.517 freebsd_shell
20. 2079     67699272    266621708   5.046 s   1.550 reactos_nopt
19. 2021     68827308    263591438   5.065 s   1.555 bentley_mcilroy
25. 2257            0    266621708   5.099 s   1.566 bionic_nopt
23. 2181            0    266621708   5.099 s   1.566 freebsd_nopt
18. 2002            0    263591438   5.383 s   1.653 bentmcil
16. 1835            0   1624564520  31.208 s   9.583 netbsd_1_4
12. 1650            0   1624564520  31.601 s   9.704 picolibc
11. 1639            0   1624564520  32.435 s   9.960 newlib
13. 1724   1495494748   1624564520  32.919 s  10.108 reactos
24. 2194            0   1624564520  37.001 s  11.362 bionic
27. 2311            0   1624564520  37.474 s  11.507 freebsd
26. 2270            0   1624564520  38.115 s  11.703 nlopt
```

Not shown here are the Bosia tests, but they show similar results.

I conclude that if you want to use a `qsort` derived from Bentley and McIlroy, you should at least replace the `swap_cnt == 0` insertion sort with a Shell sort, and probably add the check for already-sorted data.

Another change made by BSD from the Bentley-McIlroy paper was to replace the second of the two recursive calls with `goto`, with the comment "Iterate rather than recurse to save stack space". This will save little or no space unless the partitioning is significantly uneven. But it also does not prevent a possible O(N<sup>2</sup>) stack depth usage. (Also, the `goto` goes to a point just before `SWAPINIT(a, es);`, so the `SWAPINIT()` macro gets called redundantly many times.) Only in 2017 did the major BSD-derived versions change to always recursing on the smaller partition, which eliminates the quadratic stack space problem. (Bentley and McIlroy explicitly decided not to deal with the possibility.)

One other change made by FreeBSD seems odd to me: Just recently (January 2022) they made a change "prevent undefined behavior," with the following comments:

```
Mark Milliard has detected a case of undefined behavior with the LLVM
UBSAN. The mandoc program called qsort with a==NULL and n==0, which is
allowed by the POSIX standard. The qsort() in FreeBSD did not attempt
to perform any accesses using the passed pointer for n==0, but it did
add an offset to the pointer value, which is undefined behavior in
case of a NULL pointer. This operation has no adverse effects on any
achitecture supported by FreeBSD, but could be caught in more strict
environments.

After some discussion in the freebsd-current mail list, it was
concluded that the case of a==NULL and n!=0 should still be caught by
UBSAN (or cause a program abort due to an illegal access) in order to
not hide errors in programs incorrectly invoking qsort().

Only the the case of a==NULL and n==0 should be fixed to not perform
the undefined operation on a NULL pointer.
```

I cannot see how this "is allowed by the [POSIX standard](https://pubs.opengroup.org/onlinepubs/9699919799/functions/qsort.html)," which says "The qsort() function shall sort an array of nel objects, the initial element of which is pointed to by base." Nowhere does it say the base pointer can be NULL; it must point to "an array of nel objects." This is clearly a bug in mandoc. There are many ways to call `qsort` with invalid arguments that can cause various sorts of trouble.

On the matter of avoiding undefined behavior, many of the BSD variants still use the Bentley-McIlroy "trick" of `((char *)a - (char *)0)` to get a usable "address" from the base pointer. This is undefined behavior; both pointers in a pointer subtraction must point within the same array (or one past the end). I think these should be updated to use `((uintptr_t)(void *)a)` if `uintptr_t` is available (it's an optional `<stdint.h>` type in C99 and C11).

(See also: [part 1](../../../../2022/01/17/Re-engineering-a-qsort-part-1/), [part 2](../../../../2022/02/24/Re-engineering-a-qsort-part-2/), [part 4](../../../../2022/02/27/Re-engineering-a-qsort-part-4/), [part 5](../../../../2022/03/09/Re-engineering-a-qsort-part-5/))
