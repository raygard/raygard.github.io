---
layout: post
title:  How "histogram diff" actually works
date:   2025-01-28 05:00:00 -0600
---

I've been interested in file comparison (diff) algorithms for quite a while.
I'd assumed for some time that the classical approach, which uses a longest-common-subsequence (LCS) algorithm, is the best way to do this.
The original Unix diff used [an algorithm](https://www.cs.dartmouth.edu/~doug/diff.pdf) by James W. Hunt and M.D. (Doug) McIlroy.
After Eugene Myers published his [O(ND) algorithm](http://www.xmailserver.org/diff2.pdf) in 1986, it was adopted, with adaptation and variations, as the most common algorithm for file comparison, in gnu diff and many other places.
For a long time, the Myers algorithm seemed to be the best solution for diff and diff-like programs.

In 2007, BitTorrent creator Bram Cohen [published on his blog](https://bramcohen.livejournal.com/37690.html) a brief description of a non-LCS-based diff algorithm he called "patience diff".
This approach paired up lines unique to each file being compared, followed by a pass to retain only the longest sequence of such pairs that appear in order in both files.
(The pairing of unique lines was originally published by [Paul Heckel in 1978](https://dl.acm.org/doi/pdf/10.1145/359460.359467), though he did not attempt to remove "crossing" pairs, but retained them as moves between files.)
What happened in patience diff after the initial pairing of unique lines in both files was [somewhat fluid](https://bramcohen.livejournal.com/73318.html).

The patience diff algorithm can be slow, and it's not clear to me what it can do if there are few or no unique matching lines.
I believe it is generally implemented with a "fallback" method (usually Myers LCS?) to use if no matching unique lines are found, or to use between the matching unique lines.

In 2010, the `jgit` project added the [HistogramDiff class](https://javadoc.io/doc/org.eclipse.jgit/org.eclipse.jgit/latest/org.eclipse.jgit/org/eclipse/jgit/diff/HistogramDiff.html).

This was claimed to be "An extended form of Bram Cohen's patience diff algorithm."
It seems to make pretty good diffs pretty fast and made me question the primacy of the LCS approach.
I searched the WWWeb in vain for a clear and detailed explanation of how it actually works.
The description in the document linked just above is pretty unclear.
I wanted to know enough about it to implement it myself.
Finally I examined the Java [source](https://github.com/eclipse-jgit/jgit/tree/master/org.eclipse.jgit/src/org/eclipse/jgit/diff) in `jgit` and analyzed what it actually does.

Here is a text and pseudocode desciption.

Further discussion and a working histogram diff program are in a [later post](https://www.raygard.net/2025/01/29/a-histogram-diff-implementation/).

<!-- more -->

## Histogram diff algorithm

Compare files A and B via histogram diff algorithm:

A *range* is a sequence of zero or more consecutive lines in a file.
A *region* is a range of file A together with a range of file B.
A *matching region* is one with the same number of lines in the A and B ranges, with each line in the A range matching the corresponding line in the B range.

Begin with a region comprising all of file A and file B.

The idea is to find the "best" matching region within the current region, then recursively do the same with the regions before and after the matching region.
The best matching region is the longest one with the lowest low count of A lines.

Eventually, the region under consideration will have no matching lines, and is therefore a difference.

- If the region has no lines in its A range, the difference is an insertion of the B range.
- If it has no lines in its B range, the difference is a deletion of the A range.
- If it has lines in both its A and B ranges, the difference is a change.

In the following, consider only lines in the current region.
That is, "lines in A" mean lines in the A range of the current region, unless otherwise noted, and similarly for B.

For each line in A, count how many times that line occurs in A.

Set _lowcount_ initially to 65 (in jgit; somewhat arbitrary to limit time in pathological cases; I use 512).

Start finding matching regions with the first line in B, and process the B lines as follows.

### Find_best_matching_region:

Take each line in A (from low to high line number) that matches the current line from B.
If there is no matching line, or the occurrence count of this A line exceeds _lowcount_, go to the next line in B.

If no line in B matches any line in A, or all the A lines have counts over _lowcount_, the region is a difference, and we're done with the current region.

Having a matching line in A and B, widen the match by finding the longest matching region containing the original match.
Note the line in A (of the matching region) that has the fewest occurrence count, and call this count the region lowcount.
If this matching region is the first one, note its length and region lowcount, and call it the best matching region (so far).
If it's not the first matching region found, then if it has more lines OR lower region lowcount than the current best matching region, this becomes the new best matching region.
Keep the lowest region lowcount in _lowcount_.
Skip any remaining matching lines in A that fall within the matching region just found.
If any more lines in A (after the matching region) match the B line, repeat the process.
Skip the remaining B lines in the matching region and go to Find_best_matching_region to continue with the B line that follows the current matching region.

After all B lines have been considered, the best matching region is used to split the original region into before-and-after regions, and the process continues until all the differences have been found.


### Pseudocode:

```
push region(all of file A and file B) on region stack.

While region stack is non-empty:
    pop region stack to current_region
    for each B line, find all matching A lines
                        and count how many times this line occurs in A
    best_match = find_best_matching_region_in(current_region)
    if best_match is empty region:
        append current_region to diff_list
    else:
        after_match = lines in current_region after best_match
        if after_match non-empty: push after_match on region stack
        before_match = lines in current_region before best_match
        if before_match non-empty: push before_match on region stack

def find_best_matching_region_in(current_region):
    set best_match as an empty region
    set lowcount = 65 // arbitrary limit
    set i to first B line
    set nexti to i+1
    for (i = first B line; i <= last B line; i = nexti):
        nexti = i + 1
        if no line in A matches line i in B: continue
        find first line in A matching line i in B
        if count of A occurrences > lowcount: continue
        set j to this matching A line
        loop:   // consider each matching A line
            set current_match to region(j in A, i in B)
            widen current_match to include consecutive matching lines
                    j-1 and i-1, j+1 and i+1, etc.
                    to largest matching region inside current_region
            set region_lowcount to lowest occurrence count of
                    any line of A in current_match
            // Compare current_match to best_match
            if best_match is empty
                OR current_match is longer than best_match
                OR region_lowcount is less than lowcount:
                    set best_match to current_match and
                    set lowcount to region_lowcount
            set nexti to B line following current_match
            if another line in A (within current region) matches i in B:
                set j to that line in A
            else:
                break loop
        end loop    // loop on A lines matching i in B
    return best_match




File A                                    File B               
┌───────────────────────────────────────────────┐              
│                                               │              
│                                               │              
│....                                           │              
│    .........                                  │              
│            ................                   │current region
│                           ........            │  │           
│                                  ........     │  ▼           
│        before matching region           ......│─────         
│                                               │              
│===                                            │              
│  ======                                       │              
│       ===================                     │              
│                          ===========          │              
│                                     ==========│              
│                                               │              
│===       matching region                      │              
│  ======                                       │              
│       ===================                     │              
│                          ===========          │              
│                                     ==========│              
│           after matching region               │              
│                                  .............│──────        
│     ... ..........................            │  ▲           
│.. .                                           │  │           
│                                               │  │           
│                                               │              
└───                                            │              
   ──────                                       │              
        ─────────────                           │              
                     ────────────               │              
                                ──────          │              
                                      ──────────┘              
```

### Notes

The order in which the recursion is done (after-region pushed before before-region) results in the diff list coming out in order from beginning to end, ready to process into a diff listing.

I have tried to capture the original histogram diff algorithm in [jgit](https://github.com/eclipse-jgit/jgit/tree/master/org.eclipse.jgit/src/org/eclipse/jgit/diff) as closely as I could.
My pseudocode mimics but does not exactly duplicate the way the original algorithm is implemented.
I have omitted details of how the matches and counts are found.
In the original algorithm in `jgit`, if all the A lines have a too-high count to be considered for matching, the algorithm "falls back" to the Myers diff algorithm.
This should only happen if no lines in A occur fewer than 65 times.
In my implementation, I simply let the entire region be considered a diff in this case.
