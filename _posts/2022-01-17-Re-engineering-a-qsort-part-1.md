---
layout: post
title:  Re-engineering a qsort function (part 1)
date:   2022-01-17 17:00:00 -0600
---

#### Improved `qsort` implementations

I've been working on better (at least faster) implementations of `qsort` for the C standard library. A long time ago I wrote an implementation of the C standard library function `qsort`. I thought it was pretty good, but recently I've made much better versions.

<!-- more -->

#### Summary / tl;dr

This is a long series of posts. My conclusion is that you should consider `qsort` implementations `rg22j.c`, `rg22k.c`, and `rg22bk.c`; they are all faster than any others I have found. If you need a very compact implementation, the Shell sort `rg22ss.c` may fit, though it runs about 2.5 times as long as the quicksort-based versions. And if you need guaranteed O(N log N) worst-case performance, consider `rg22heap3.c`, though it runs about 3.8 times as long as the quicksort versions.

Code for these posts are in [https://github.com/raygard/qsort_dev](https://github.com/raygard/qsort_dev).

All my own code is licensed 0BSD, meaning you can do whatever you like with it. There is no warranty, but I've tried to avoid defects (of course). If you can find any bugs, please let me know at raygard at gmail dot com.

#### My original qsort

I followed the guidance of Robert Sedgewick in [Implementing Quicksort Programs (Communications of the ACM, vol 21 no 10, Oct. 1978 pp 847–857)](https://www.csie.ntu.edu.tw/~b93076/p847-sedgewick.pdf). Sedgewick had recently completed his PhD thesis on quicksort, so I knew his approach was solid.

(NOTE: If you want to use that article to implement a quicksort, be sure to read the Corrigendum from CACM, June 1979, page 368. You can see it on the [last page here](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.419.3613&rep=rep1&type=pdf). If that or any other links here do not work, please let me know at raygard at gmail dot com and I will do my best to fix it.)

#### Quicksort, explained quickly

If you already know the algorithm well, skip this.

The basic idea, discovered / invented by C.A.R. Hoare around 1960 [(Algorithm 64 in CACM vol 4 no 7, July 1961 p 321)](https://courses.cs.vt.edu/~cs3114/Summer14/Notes/Supplemental/p321-hoare.pdf) (also [here](http://algol60.org/acm/064.pdf), which omits the partition procedure but includes useful modifications) is to rearrange array elements into two sections, so all the elements in the lower section compare less than or equal to all the elements in the upper section. Then do the same to each of those sections, and then again to each of the resulting subsections, recursively until the sections are down to a single element. Then all elements are in order. The rearranging part is called partitioning and does the heavy lifting of the algorithm.

The usual method is to pick one element as the "pivot", and get all the elements less or equal to it to one end and all the larger elements to the other end. (Items equal to the pivot can really go either way.) Doing this efficiently and correctly is the tricky part, so I followed Sedgewick closely on that.

The most common way to implement the partition phase is to have two pointers approach each other from either end of the array. First the low pointer is incremented until it reaches an element not less than the pivot. Then the high pointer is decremented until it reaches an element not greater than the pivot. These two elements are swapped and then the process is repeated until the pointers meet or cross. There are more ways to get this wrong than right. Also note that the process stops on elements equal to the pivot. That prevents the sort from running in quadratic (O(N<sup>2</sup>)) time with (for example) inputs of random zeros and ones.

A couple of important points: Instead of splitting the sections down to single elements, Sedgewick suggests using insertion sort for very small subsections (often called subfiles in the literature), an idea he credits to Hoare in a [paper on quicksort in Computer Journal, April 1962](https://academic.oup.com/comjnl/article/5/1/10/395338) (also [here](https://www.cs.ox.ac.uk/files/6226/H2006%20-%20Historic%20Quicksort.pdf) and [here](http://rabbit.eng.miami.edu/class/een511/quicksort.pdf)). Sedgewick leaves the small sections unsorted until the end and then runs insertion sort over the entire array. This was a win for him but I've seen it claimed that modern memory caches make it better to sort the small sections individually. That's what I did in my version.

Another important matter is to try to split each segment as equally as possible. A bad enough split can make the algorithm run in quadratic time. Ideally the pivot should be the median element, but that's not known _a priori_. Sedgewick uses the median of the first, last, and middle elements, credited to R.C. Singleton in 1969. Sedgewick cleverly arranges things so that the lowest and highest of the three are used as sentinels in the partitioning.

#### An embarrassing defect

I contributed an early version of my `qsort` to the Zortech C compiler team. It was shipped in the next release of the compiler, and then a Zortech engineer found a defect that caused a crash in DOS "large" memory model. The insertion sort had a loop:

{: .lh-tight }
```C
    for ( ; j >= base && (*comp)(j, j+size) > 0; j -= size)
        SWAP(j, j+size);
```

This attempts to decrement pointer `j` until it is lower than the `base` of the array. This is undefined behavior, but it "works" in many cases. I fixed the insertion sort but this was embarrassing.

After modifying it for ANSI C, the corrected version of the `qsort` was included in the C SNIPPETS collection maintained by the late Bob Stout (RIP).

#### My `qsort` was not *that* bad

A few years ago I was poking around, vainly trying to see if any of my old code was in use. I came across a thread in a forum at [SourceForge](https://sourceforge.net) but the original forum entries seem to have disappeared. There is still a [reference to the discussion](https://sourceforge.net/p/pdos/bugs/1/):

{: .lh-tight }
```
#1 qsort() stack overflow
Status: closed-fixed Owner: nobody Labels: None
Priority: 5
Updated: 2006-11-18 Created: 2006-05-24 Creator: Martin Baute

The qsort() implementation in PDPCLIB suffers from a
possible stack overflow. You are using a static stack
with 40 elements. With each iteration, the larger
subarray's indices get stored on the stack (2 indices -
> 20 usable slots on the stack). With T == 7, this
means a worst-case scenario could result in an
overflow on an array with as few as ( T + stacksize )
== 28 elements!

Ref. PDCLib bug #1494254
(http://sourceforge.net/tracker/index.php?
func=detail&aid=1494254&group_id=94882&atid=609405)
```

I should have made the stack larger, but his analysis is wrong.

The 20 slots are good for sorting at least 7,864,320 elements, and in my defense, that seemed huge back around 1990.

#### Try to do better

But this got me thinking about upgrading my `qsort`. I remembered seeing references to a paper about a better `qsort` function but could not then get my hands on the paper. I looked around more recently and found [Engineering a Sort Function](https://cs.fit.edu/~pkc/classes/writing/samples/bentley93engineering.pdf) (also [here](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.14.8162&rep=rep1&type=pdf)) by Jon Bentley and Doug McIlroy, Software--Practice and Experience, vol. 23 no. 11, Nov. 1993, pp 1249-1265 (I'll refer to it as EASF).

The motivation for Bentley and McIlroy was finding that an existing Unix `qsort` went quadratic (O(N<sup>2</sup>) time) on some inputs. I read the paper and decided to try to apply some of the its techniques to improve my older SNIPPETS `qsort`. I copied the code from the paper and got it running to use as a comparison. It was much faster than my old `qsort`.

The EASF paper also has pseudocode for a "certification" program to use as a sort of torture test for `qsort` functions. I wrote a version of that to compare different `qsort` implementations. The test program generates arrays of integers in a variety of patterns and sorts them, counting the number of comparisons. It also runs the same tests after converting the C `int`s to `double`s. I added tests converting the `int`s to pointers to strings containing ASCII representations of the integers, and also converting to `struct`s containing string representations of the integers.

I don't know what kinds of data are most common for `qsort`, but I suspect that sorting pointers to strings or `struct`s, or actual arrays of `struct`s are more common than arrays of scalar numbers.

The paper introduced some ideas about implementing quicksort that differed somewhat from Sedgewick's recommendations. Some of them were:

* Better median estimation: instead of just using a median-of-3, Bentley and McIlroy used middle element for small arrays, median of 3 for mid-size arrays, and a median of three medians of three, taken from the first, middle, and last third of the array (something dubbed a "ninther" by mathematician John Tukey).

* A more efficient swap method than just swapping elements bytewise.

* A way to efficiently handle large numbers of duplicate keys.

I modified my old `qsort` to incorporate these improvements. I found and downloaded a number of other `qsort` implementations, many of them using some of the same or similar code to that in EASF. My sort was done a bit differently, mainly in the handling of keys equal to the chosen pivot. EASF uses a clever technique of pushing these keys to the ends of the array during partitioning, then moving them adjacent to the pivot. I do this too, but a little differently.

The EASF sort tests for elements equal to the pivot and swaps them to the ends of the array in the inner loops. My inner loops are closer to the original Sedgewick logic:

```c
        while ((kj=COMP(j -= size, left)) > 0)
            ;
        while (i < j && (ki=COMP(i += size, left)) < 0)
            ;
```

The tests below were run on an HP Probook 450 G5 with an Intel Core i5 Kaby Lake using Ubuntu in WSL in Windows 10. The runs were on 10000 elements repeated 5 times. More details about the test program will follow in later parts. Note that the results may not be in order by time.

Sorts that report 0 swaps have not been modified to count them, though in some later tests there are sorts reporting 0 swaps because no swaps were needed for already-sorted data.

I had extracted the code from the EASF paper and called it `bentmcil.c`. I later found a different version at [Doug McIlroy's Web page](https://www.cs.dartmouth.edu/~doug/qsort.c), and called it `bentley_mcilroy` in my tests. My first version `rg22a` ran a little faster than theirs, but took more swaps:

```
   Tot.rank     Swaps     Compares      Time   Ratio Implementation
 1.  176    186396960    550721420  10.244 s   1.000 rg22a
 2.  264            0    585420940  11.070 s   1.081 bentmcil
 3.  280    158006340    585420940  11.088 s   1.082 bentley_mcilroy

```

To reduce the number of swaps, I modified it to simply skip swapping an element with itself. This gave me `rg22b`:

```
   Tot.rank     Swaps     Compares      Time   Ratio Implementation
 1.  182    121272940    550721420   8.039 s   1.000 rg22b
 2.  254    186396960    550721420   9.762 s   1.214 rg22a
 3.  375            0    585420940  11.120 s   1.383 bentmcil
 4.  389    158006340    585420940  10.450 s   1.300 bentley_mcilroy
```

I experimented with different thresholds for pivot selection. These are the breakpoints between insertion sort and quicksort, middle element pivot, median-of-3 pivot, and median-of-3-medians-of-3 (Tukey "ninther") pivot, to get `rg22c`. To be fair to EASF, their sort also swaps elements with themselves, about 25%-30% of the time, so I modified their code to avoid that and called it `bentley_mcilroy2`:

```
   Tot.rank     Swaps     Compares      Time   Ratio Implementation
 1.  222    121272940    550721420   8.162 s   1.000 rg22b
 2.  258    115637080    544721860   8.162 s   1.000 rg22c
 3.  396    186396960    550721420   9.922 s   1.216 rg22a
 4.  504            0    585420940  11.242 s   1.377 bentmcil
 5.  538    118093240    585420940   9.103 s   1.115 bentley_mcilroy2
 6.  602    158006340    585420940  10.784 s   1.321 bentley_mcilroy
```

The `rg22b` version takes fewer compares than the modified `bentley_mcilroy2`, but still a few more swaps. The `rg22c` version takes fewer swaps and fewer compares than either `bentley_mcilroy2` or `rg22b`, but for some reason, `rg22b` seems to run faster in most of my tests. This may be due to `rg22b` doing more insertion sorting and less quicksorting. Also, `rg22b` never chooses just the middle element, but always uses median-of-3 or ninther pivot choice.

I experimented with the insertion/pivot selection thresholds, but have not improved overall on `rg22c`.

See [part 2](../../../../2022/02/24/Re-engineering-a-qsort-part-2/) for further developments.

(See also: [part 3](../../../../2022/02/26/Re-engineering-a-qsort-part-3/), [part 4](../../../../2022/02/27/Re-engineering-a-qsort-part-4/), [part 5](../../../../2022/03/09/Re-engineering-a-qsort-part-5/))
