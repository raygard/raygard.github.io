---
layout: page
title: My projects
---

## About some of my projects

### An awk implementation

This is a work in progress, with code at [github](https://github.com/raygard/wak) (`wak` is a working name that is subject to change if I think of one I like better) and some fragmentary documentation [here](/awkdoc).

### Longest Common Subsequence algorithms

I've been interested in `diff` algorithms for a very long time, ever since a guy I worked with had written one for the Univac 1108. There have been many, the "formal" ones using solutions of the "longest common subsequence" problem (for two sequences). There are many solutions for that also, the earliest published being Hunt and Szymanski and/or Hunt and McIlroy (the latter being available only as a Bell Labs internal publication at first). I wrote about that on my [blog](/2022/08/26/diff-LCS-Hunt-Szymanski-Kuo-Cross/), including some later refinements, and have demo code at [github](https://github.com/raygard/lcs_diff_demo).

### `tsort` implementations

There is a POSIX spec for `tsort`, a utility that does a topological sort on pairs of alphanumeric symbols. I wrote a version that is straight out of Knuth TAOCP volume 1, with detailed comments linking the code to Knuth's informal description of his algorithm T. The result is at [github](https://github.com/raygard/tsort).

### `qsort()` implementations

Many years ago I wrote a `qsort()` implementation. It did not have enough stack space for today's potentially huge arrays. When I started fix that I found that it was not as performant as some of today's implementations, so I set out to write the best versions I could. The results are at [github](https://github.com/raygard/qsort_dev) with some longish exposition on it at my blog starting [here](/2022/01/17/Re-engineering-a-qsort-part-1/).
I thought it was about as good as can be done, but some recent research seems to indicate that a multiple-pivot version can do better. I tried that without much success, but may follow up on it again in the future.

### giflzw

I contributed a GIF encoder to the Python Imaging Library (PIL, now Pillow). It was an interesting project, and seems to be working without any issues for a few years now. The project files are at [github](https://github.com/raygard/giflzw); also see the [documentation](/giflzw/). (This is code for doing the same kind of incremental coding that is in the GifEncode.c module in Pillow, but that module is not included here.)

### readability-rg

This Python package computes common readability measures on text files. It computes Flesch reading ease, SMOG index, and others. The code is at [github](https://github.com/raygard/readability-rg); also see the [documentation](/readability-rg/).
