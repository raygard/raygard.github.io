---
layout: post
title: More on "histogram diff", and a working program
date: 2025-01-29 05:00:00 -0600
---

The original version of histogram diff was implemented `jgit` by the late Shawn Pearce in 2010, with the [comment](https://eclipse.googlesource.com/jgit/jgit/+/b533a7293429258f34a6778a45a6c66dac55dc43) that "HistogramDiff is an alternative implementation of patience diff, performing a search over all matching locations and picking the longest common subsequence that has the lowest occurrence count. If there are unique common elements, its behavior is identical to that of patience diff."

I have written a stand-alone diff implementation that I believe exactly duplicates the behavior of histogram diff in `jgit`, and usually creates an identical (and otherwise very close) output to what `git diff --histogram` produces.
You can find my code [here](https://github.com/raygard/hdiff).

I don't think this actually behaves quite like patience diff, even if there are many unique lines, but it seems to usually do a good job.

In a paper from September 2019, [How different are different diff algorithms in Git?](https://link.springer.com/article/10.1007/s10664-019-09772-z) (PDF downloadable at the link), the authors compare two of git's diff algorithms: _Myers_ and _histogram_.
They considered _minimal_ and _histogram_ as improved versions of _Myers_ and _patience_ respectively, and said "Due to the similarity of the basic idea of Minimal and Histogram algorithms with their precursors, in this paper we only contrasted the two diff algorithms: _Myers_ and _Histogram_."
After studying to see how well Myers performed vs. histogram (a somewhat subjective matter), they concluded that histogram does better at indicating the intended changes between files.
It would be nice to have a serious study of histogram vs. patience diff, as I think they will often produce different results on real-world files.

The description in the paper linked above has a somewhat hand-wavy description of the histogram algorithm.
> The Histogram strategy works similarly to the Patience by developing a histogram of
the appearances for every line in the first version of a file. Every element in the second
version is subsequently shown to match with the first sequence in an orderly way to find
the existences of the elements and to count the occurrences. If the elements exist and their
presences are less than in the first sequence, they are expected to be a potential LCS. 

I studied the Java [code](https://github.com/eclipse-jgit/jgit/tree/master/org.eclipse.jgit/src/org/eclipse/jgit/diff) in `jgit` and analyzed what it actually does.

The code in `jgit` was a bit hard for me to follow, and the comments are misleading in some places.
To begin with, there is the nonstandard use of "longest common subsequence (LCS)" in the javadoc comment in `HistogramDiff.java`.
The comment says in part:
> The basic idea of the algorithm is to create a histogram of occurrences for each element of sequence A. Each element of sequence B is then considered in turn. If the element also exists in sequence A, and has a lower occurrence count, the positions are considered as a candidate for the longest common subsequence (LCS). After scanning of B is complete the LCS that has the lowest number of occurrences is chosen as a split point.

"If the element also exists in sequence A, and has a lower occurrence count" ... lower than what?
"...candidate for the longest common subsequence (LCS)" ... This algorithm does not find an LCS as the term is normally used in computing theory.
It finds what is sometimes called a longest common _substring_ or maybe longest common _contiguous_ sequence.
I am not quite convinced that it always finds a longest common substring, either.

The code comment in `HistogramDiffIndex.java` is a little closer to what really goes on:
> Each element in the range being considered is put into a hash table, tracking the number of times that distinct element appears in the sequence. Once all elements have been inserted from sequence A, each element of sequence B is probed in the hash table and the longest common subsequence with the lowest occurrence count in A is used as the result.

Though again, the region found is not a "longest common subsequence" as the term is normally understood.

Some people have found that the histogram algorithm doesn't always produce the most readable diffs, but [Linus Torvalds](https://lkml.org/lkml/2023/5/7/206) prefers it over the default Myers.

One thing that can confound the algorithm is if there are no lines appearing fewer than 65 times.
When that happens, the `jgit` and `git` implementations fall back to Myers.
My implementation currently cuts off at 512.
The comment in `jgit` says 
> All elements with the same hash are stored into a single chain. The chain
 size is capped to ensure search is linear time at O(len_A + len_B) rather
 than quadratic at O(len_A * len_B).

Actually, the underlying behavior is worse than quadratic.
A test with this limit uncapped, with sequences of a,b,c,a,b,c,... and c,b,a,c,b,a,... show that it is more likely to be cubic.
But this is artificial worst-case data.
Real-world data is not likely to be close to this bad, and the algorithm normally runs quite quickly.
