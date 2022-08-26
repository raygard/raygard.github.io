---
layout: post
title:  Diff and Longest Common Subsquence (LCS) with Hunt/Szymanski and Kuo/Cross algorithms
date:   2022-08-26 05:00:00 -0600
---

### Diff and Longest Common Subsequence (LCS)

(This post includes demo code; see [below after the "Read on" jump](#democode):

If you've used a diff program, you probably used a solution to the longest common subsequence (LCS) problem. The most well-known diff implementations, the original Unix diff and GNU diff, are both based on LCS solutions, but use different algorithms. Here, I'll try to explain two related LCS algorithms informally (or less formally than in the original technical papers).

<!-- more -->

Let *A* and *B* be sequences taken from some set of symbols. A subsequence is formed by removing zero or more (not necessarily contiguous) symbols from a sequence. If *C* is a subsequence of *A* and of *B* then it's a common subsequence. If it's as long as any possible common subsequence, then it's a longest common subsequence. Note that an LCS of *A* and *B* may not be unique.

Consider a couple of text files as sequences of lines. If you find the (or a) longest subsequence of lines that's common to both files, then the lines that are not in the LCS are the difference. These are the lines that would have to be deleted from or inserted into the first file to get the second file. That's exactly what *diff* finds.


### Longest Common Subsquence (LCS) with Hunt/Szymanski and Kuo/Cross algorithms

In the mid-1970s M. D. (Doug) McIlroy and J. W. Hunt wrote the original Unix *diff* program.
They described the algorithm and the program itself in some detail in a Bell Labs report (Bell Laboratories Computing Science Technical Report # 41, July 1976). Because this was an internal report, it was not (AFAIK) generally available or even known outside Bell Labs.
A similar algorithm was later published in Communications of the ACM by Hunt and T. G. Szymanski.
A less well known modification that often performs better was published in 1989 by S. Kuo and G. R. Cross.

See:

Hunt_McIlroy_1976: J. W. Hunt and M. D. McIlroy, "An algorithm for differential file comparison," *Bell Telephone Laboratories Computing Sciences Technical Report* #41 (1976). [https://www.cs.dartmouth.edu/~doug/diff.pdf](https://www.cs.dartmouth.edu/~doug/diff.pdf) (text edited from OCR, figures redrawn). Erratum: coauthor of referenced Szymanski paper is J. W. Hunt, not H. B. Hunt III.

Hunt_Szymanski_1977: J. W. Hunt and T. G. Szymanski, "A fast algorithm for computing longest common subsequences," *Commun. ACM,* vol. 20 no. 5, pp. 350-353, May 1977. [https://dl.acm.org/doi/pdf/10.1145/359581.359603](https://dl.acm.org/doi/pdf/10.1145/359581.359603) or [http://www.cs.ust.hk/mjg_lib/bibs/DPSu/DPSu.Files/HuSz77.pdf](http://www.cs.ust.hk/mjg_lib/bibs/DPSu/DPSu.Files/HuSz77.pdf)

Kuo_Cross_1989: S. Kuo and G. R. Cross, "An improved algorithm to find the length of the longest common subsequence of two strings," *SIGIR Forum,* vol. 23 issue 3-4, pp. 89-99, Spring 1989. [https://dl.acm.org/doi/pdf/10.1145/74697.74702](https://dl.acm.org/doi/pdf/10.1145/74697.74702)

I'll refer to these algorithms as HM, HS, and KC respectively.

The paper by Hunt and McIlroy describes HM better than I can, and KC (as modified below) works as well or better, so I won't discuss HM here (but [see my Python code for Hunt/McIlroy](https://github.com/raygard/lcs_diff_demo/blob/main/src/python/hm.py)). I'll start with trying to explain HS.

Assume length of *A* is *m* and length of *B* is *n*. *LLCS(A, B)* denotes the length of *LCS(A, B)*.

Sequences will be referred to with 1-based indexes. A *string* is a contiguous sequence of symbols. A *prefix* is a string that begins at the first symbol in a sequence, e.g. *acb* is a prefix of *acbdef*. So a prefix *A[1], A[2], ... A[i]* of *A* of length *i* can be denoted *A[1..i]*.

A *match* is a pair *(i, j)* where *A[i] == B[j]*; the prefixes *A[1..i]* and *B[1..j]* will end with the same symbol.

We'll examine *A* from left to right, one symbol at a time. For each symbol in *A*, we look at matches in *B*. Not all matches will be part of an LCS. Determining the right set of matches is the problem.

#### HS (Hunt-Szymanski) algorithm

The central idea is to maintain an array *T* (called *THRESH* in the HS and KC papers) of lengths of prefixes of *B*. After examining a symbol of *A*, we update *T* so that *T[k]* is the length of the shortest prefix of *B* that has an LCS of length *k* with the prefix of *A* examined so far.

If that's not clear, study this tedious example (or [skip ahead](#continue1)):

Using Hunt/Szymanski's example strings: *A = abcbdda*, *B = badbabd*

After examining A[1..1] (*a*),
T[1] is 2 because there is a CS of length 1 (*a*) between A[1..1] (*a*) and B[1..2] (*ba*).

After examining A[1..2] (*ab*),
T[1] is 1 because there is a CS of length 1 (*b*) between A[1..2] (*ab*) and B[1..1] (*b*),
and T[2] is 4 because there is a CS of length 2 (*ab*) between A[1..2] (*ab*) and B[1..4] (*badb*).

After examining A[1..3] (*abc*) (*c* does not match anything in *B*, nothing changes),
T[1] is 1 because there is a CS of length 1 (*b*) between A[1..3] (*abc*) and B[1..1] (*b*),
and T[2] is 4 because there is a CS of length 2 (*ab*) between A[1..3] (*abc*) and B[1..4] (*badb*).

After examining A[1..4] (*abcb*),
T[1] is 1 because there is a CS of length 1 (*b*) between A[1..4] (*abcb*) and B[1..1] (*b*),
and T[2] is 4 because there is a CS of length 2 (*ab*) between A[1..4] (*abcb*) and B[1..4] (*badb*)
and T[3] is 6 because there is a CS of length 3 (*abb*) between A[1..4] (*abcb*) and B[1..6] (*badbab*).

After examining A[1..5] (*abcbd*),
T[1] is 1 because there is a CS of length 1 (*b*) between A[1..5] (*abcbd*) and B[1..1] (*b*),
and T[2] is 3 because there is a CS of length 2 (*ad*) between A[1..5] (*abcbd*) and B[1..3] (*bad*)
and T[3] is 6 because there is a CS of length 3 (*abb*) between A[1..5] (*abcbd*) and B[1..6] (*badbab*).
and T[4] is 7 because there is a CS of length 4 (*abbd*) between A[1..5] (*abcbd*) and B[1..7] (*badbabd*).

After examining A[1..6] (*abcbdd*),
T[1] is 1 because there is a CS of length 1 (*b*) between A[1..6] (*abcbdd*) and B[1..1] (*b*),
and T[2] is 3 because there is a CS of length 2 (*ad*) between A[1..6] (*abcbdd*) and B[1..3] (*bad*)
and T[3] is 6 because there is a CS of length 3 (*abb*) between A[1..6] (*abcbdd*) and B[1..6] (*badbab*).
and T[4] is 7 because there is a CS of length 4 (*abbd*) between A[1..6] (*abcbdd*) and B[1..7] (*badbabd*).

After examining A[1..7] (*abcbdda*),
T[1] is 1 because there is a CS of length 1 (*b*) between A[1..7] (*abcbdda*) and B[1..1] (*b*),
and T[2] is 3 because there is a CS of length 2 (*ad*) between A[1..7] (*abcbdda*) and B[1..3] (*bad*)
and T[3] is 5 because there is a CS of length 3 (*bda*) between A[1..7] (*abcbdda*) and B[1..5] (*badba*).
and T[4] is 7 because there is a CS of length 4 (*abbd*) between A[1..7] (*abcbdda*) and B[1..7] (*badbabd*).

<a name="continue1"></a>
(End of example.)

Note that the *T* values are always strictly ascending after each step, and that for any *k*, *T[k]* can decrease (but never increase) from one step to the next.

Start with *T* of length *m + 1*. We set *T[0] = 0* and set all other elements to *n + 1*.

So for our example *A = abcbdda*, *B = badbabd*, we initialize *T* as [0, 8, 8, 8, 8, 8, 8, 8].

We'll say values of *T* changed after this initialization are *assigned* to *T*.

The general structure of the algorithm is:
```
for i from 1 to m:
    for B[j] where A[i] == B[j]:
        Set T[k] to lowest j such that LLCS(A[1..i], B[1..j]) is k
```

When we finish, the index of the highest element of *T* assigned to will be the length of *LCS(A, B)*. We'll see later how the actual LCS (and the matching pairs from *A* and *B*) can be obtained.

This doesn't tell us much, other than that we maintain the invariant status of *T* (*T[k]* is lowest *j* such that *A[1..i]* and *B[1..j]* have an LCS of length *k*). How can we maintain this invariant?

We use a structure of matches (called *MATCHLIST* in Hunt_Szymanski_1977) that tells, for each symbol in *A*, locations where the corresponding symbol is found in *B*. (This can be done with sorting and/or searching operations in O(n log n) time.) Then as each new *A* symbol is processed, the corresponding *B* symbols are found in *MATCHLIST* and their locations "merged" into the *T* array. The correct location in *T* for each match can be found using a binary search.

Say we've established *T* after looking at *A[1..i-1]* and we now look at *A[i]*. If there are no matches in *B* then there's nothing to do at this step. If we find one or more matches in *B* we need to update *T*. The HS algorithm has the MATCHLIST locations for *B* in descending order. We take each one and find where it should fall in *T*.

For each *j* in *MATCHLIST(i)*, we use a binary search to locate *T[k]* such that *T[k-1] < j <= T[k]*. Note that with the values we initialized *T* to, there will always be such a *T[k]*.

Here is the critical observation: After the previous iteration where we examined *A[i-1]*, we had *B[1..T[k-1]]* as the shortest prefix of *B* that had *LLCS(A[1..i-1], B[1..T[k-1]]) == k-1*. Now *B[j]* extends that prefix by matching *A[i]*, so *LLCS(A[1..i], B[1..j])* is *k*. But if *j* is less than *T[k]*, then *T[k]* is no longer the shortest prefix of *B* with *LLCS(A[1..i], B[1..j])*. So if *j < T[k]* we assign *T[k] = j*.

Due to the way *T* is initialized, this works for every case.

Now the HS algorithm looks like this:
```
for i from 1 to m:
    for each descending j where A[i] == B[j]:
        find k such that T[k-1] < j <= T[k]
            if j < T[k]
                T[k] = j
```

The length of the LCS is the largest *k* such that *T[k] != n + 1*.

Note that we need to have the MATCHLIST values in descending order. For example, if we have two matches *B[p]* and *B[q]* with *A[i]*, and *T[k-1] < p < q < T[k]*, then if we update *T[k] = p* first, the *q* will no longer fall between *T[k-1]* and *T[k]* and will be incorrectly be assigned to *T[k+1]*.

To get the actual *(i, j)* pairs of the LCS matches, save the pairs at each assignment to *T* in a linked list, with a list head for each *k*.

Create an array *link* of *m+1* pointers and set *link[0] = NULL*. Now on each assignment to *T*, create a struct *dmatch = (i, j, ptr)*, set *ptr = link[k-1]* and *link[k] = dmatch*, linking the new struct into the list headed at *link[k]* and pointing to the next shorter match referenced by *link[k-1]*. 

Now the HS algorithm looks like this:
```
for i from 1 to m:
    for each descending j where A[i] == B[j]:
        find k such that T[k-1] < j <= T[k]
            if j < T[k]
                T[k] = j
                link[k] = new_dmatch(i, j, link[k-1])
```

After this finishes find the largest *k* where *T[k] != n + 1*, and this is the length of *LCS(A, B)*.

Now *link[k]* will point to a chain of *dmatch* nodes with the actual LCS pairs in descending order.

This is essentially all there is to the HS algorithm.

#### KC (Kuo-Cross) algorithm

In 1989, Kuo and Cross published an improvement. They noted that when there are many matches between *T[k-1]* and *T[k]*, the *T[k] =j* assignment happens needlessly often. (Note also that the Hunt-McIlroy paper says "By handling the equivalence class in reverse order, Szymanski[10] circumvents the need to delay updating K[r], but generates extra 'candidates' that waste space." This observes the same problem.)

They proposed keeping the matchlists in ascending order, and changing the algorithm above to:
```
for i from 1 to m:
    temp = 0, k = 0
    for each ascending j where A[i] == B[j]:
        if j > temp
            do k = k + 1 while j > T[k]
            // Now T[k-1] < j <= T[k]
            temp = T[k]
            T[k] = j
```

The Kuo/Cross paper addresses only the *length* of the LCS and does not at all explain how to get the actual LCS matches. I modeled the method below based on the Hunt-McIlroy paper but I've not seen it published anywhere.

Since the *T* array is updated a bit differently, the linking of the *dmatch* nodes is slightly trickier:
```
for i from 1 to m:
    temp = 0, k = 0, r = 0
    c = link[0]
    for each ascending j where A[i] == B[j]:
        if j > temp
            do k = k + 1 while j > T[k]
            // Now T[k-1] < j <= T[k]
            temp = T[k]
            T[k] = j
            prev = link[k-1]
            link[r] = c
            r = k
            c = new_dmatch(i, j, prev)
    link[r] = c
```

I find that this runs quite a bit faster than HS in most cases. Since Kuo/Cross do not mention the extraneous creations of the *dmatch* nodes, they don't mention the space wasted that was mentioned by the Hunt/McIlroy paper, but I'm pretty sure the extra node creation and cleanup account for much of the performance difference.

How many "extra" *dmatch* nodes does HS create? Comparing source files of 963 and 798 lines, KC created 877 nodes while HS created 4146! Not a small difference. Comparing two files of about 5 MB, KC created 46010362 notes while HS created 259351879, about 5.6 times as many.

Note that the handling of the *dmatch* nodes will cause some nodes to have no pointer to them, so you'll need to have a strategy for freeing them if you need to free them. In my demo programs I added a pointer to each node and keep all the nodes in that chain, so after recovering the LCS matches I walk that chain and free the memory.

#### My modified KC algorithm

I tried a modification of KC that I call KCM or KCMOD. In place of the line `do k = k + 1 while j > T[k]`, I use a binary search between *k* and *m* (inclusive) to find the insertion point. This variant is rarely much slower and often faster than the original KC, especially if there are many matches between *A* and *B*.

#### Additional factoids

A *dominant match* (also called *minimal match* or *candidate*) *(i, j)* is one where removing the last symbol of either prefix will reduce the length of the the LCS of the prefixes, and in fact *LLCS(A[1..i], B[1..j]) - 1 == LLCS(A[1..i-1], B[1..j]) == LLCS(A[1..i], B[1..j-1])*. If the LLCS of the prefixes is *k* this match is also called *k-candidate* in some of the literature. All the matches in an LCS are dominant matches, but not all dominant matches need be part of an LCS.

The assignments to the *T* (THRESH) array occur as dominant matches are found. This is why I called the struct nodes holding the matches *dmatch*. When more than one match falls between T elements, HS will make bogus assignments for the matches before the lowest one, and the corresponding *dmatch* nodes are not really dominant matches.

An interesting discovery: Recall that Hunt/McIlroy said the HS algorithm "generates extra 'candidates' that waste space." I found in testing that the KC and KCMOD algorithms find *exactly* the same candidates as the original HM algorithm and in the same order. I suspect they are essentially the same algorithm, but that KC is easier to understand than HM (given the explanation in the HM paper at least.)

#### Demo code <a name="democode"></a>

See my repo at [https://github.com/raygard/lcs_diff_demo](https://github.com/raygard/lcs_diff_demo) for code. I have included Hunt/McIlroy in Python and also C and Python of Hunt/Szymanski and Kuo/Cross and my modified Kuo/Cross, along with a diff.py and a diff.c that create a unified diff using these algorithms.
