---
layout: post
title:  wak - An awk implementation released
date:   2024-10-10 05:00:00 -0600
---

### A production release

I posted over a year ago that an awk implementation called wak was "coming soon?"

It's finally in good enough shape I feel OK saying it's here. Of course it's not finished; nothing like that is ever quite finished. Brian Kernighan maintained his original awk up until last year (with over 200 fixes over nearly 40 years!), and it's still being maintained. I do hope to keep improving wak.

The code is in https://github.com/raygard/wak and a writeup on its development is at https://www.raygard.net/awkdoc/.

<!-- more -->

This was written primarily to provide an awk implementation for [Rob Landley's toybox project](http://landley.net/toybox) (repo at https://github.com/landley/toybox). He has placed it in the "pending" folder, and it remains to be seen if he really wants it or will promote it eventually to the "posix" folder. If you use toybox, you may want to get the version here, as the version in the toybox repo may not always be up to date.

I developed wak as a standalone project initially because it was much easier to run compile/test cycles that way. Since it's under Landley's [public-domain-adjacent 0BSD license](https://spdx.org/licenses/0BSD.html), it can be freely used and adapted by anyone. I think if you find gawk or mawk satisfactory, you will probably have no need for wak. But if you want or need a less-encumbered and maybe more compact implementation, wak may be what you're looking for.

Despite having not yet made any attempt at publicising wak, it seems to have been found by several people, including gawk implementer Arnold Robbins. Arnold mentioned wak to Prof. Nelson H. F. Beebe (University of Utah), a noted awk expert who has written thousands of awk scripts over the past 40 or so years. Prof. Beebe then attempted installing wak on a variety of systems and found it can be built on most of them. He informed me of the results, pointed out some issues, and advised me of the need for a set of tests to run as part of the build procedure.

I have a testing regime I use during development (documented in https://www.raygard.net/awkdoc/) but it's not suitable for use as part of the distribution. So I adapted about 200 tests from gawk 5.3.1 and incorporated an automated test runner nearly verbatim from Prof. Beebe's suggestions. The tests, along with a new record-reading routine, are the major improvements since the first versioned release a couple months ago. I am still working on incorporating more of Prof. Beebe's advice.

Please feel free to send me feedback, either as issues at the github repo or directly to raygard at gmail dot com.
