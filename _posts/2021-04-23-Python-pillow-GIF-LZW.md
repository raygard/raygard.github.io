---
layout: post
title:  LZW encoding for Python Pillow (PIL) 
date:   2021-04-23 17:00:00 -0600
---

I have recently made my first contribution to a major (?) open source project.

The Python Imaging Library (PIL), now known as [Pillow](https://python-pillow.org/), is probably the best-known library for image processing in Python. (See [documentation](https://pillow.readthedocs.io/en/stable/) and [source](https://github.com/python-pillow/Pillow).)

I had a need to handle GIF files in Python, and I noticed some time ago that they were not compressed very well. The library was originally written by Fredrik Lundh starting in the late 1990s. Fredrik is a brilliant programmer (also created the ElementTree system for handling XML in Python, among other projects). But when he created PIL, GIFs were under a bit of a cloud.

<!-- more -->

The [GIF file format](https://en.wikipedia.org/wiki/GIF) (Graphics Image Format) was created by [Compuserve](https://en.wikipedia.org/wiki/CompuServe) in 1987 and updated in 1989. Though now over 30 years old, it is still widely used on the World Wide Web. It is especially popular for animated images. The GIF format uses the [Lempel-Ziv-Welch (LZW)](https://en.wikipedia.org/wiki/Lempel%E2%80%93Ziv%E2%80%93Welch) algorithm to encode the image pixel data, usually saving considerable space over what would be used by the raw pixel data. The use of LZW is specified in Appendix F of the [GIF specification](https://www.w3.org/Graphics/GIF/spec-gif89a.txt) (and Appendix C of the earlier [1987 specification](https://www.w3.org/Graphics/GIF/spec-gif87.txt)).

It happened that [Sperry Corp. (later named Unisys)](https://en.wikipedia.org/wiki/Lempel%E2%80%93Ziv%E2%80%93Welch#Patents) held [U.S. Patent 4,558,302](https://patents.google.com/patent/US4558302) on the LZW algorithm. You can read on the Web all about the ensuing confusion caused by Unisys's ever-shifting stance on enforcing the patent against software that handled GIF files. Compuserve claimed it was unaware of the patent when LZW was incorporated in the GIF spec. Suffice it to say that it caused enough [FUD](https://en.wikipedia.org/wiki/Fear,_uncertainty,_and_doubt) to lead to GIF writing being abandoned by some image software. (It is widely believed that the patent only covered LZW encoding, not decoding.)

I have no inside information about this, but my guess is that this patent hassle did lead Fredrik Lundh to avoid using LZW encoding in the PIL GIF routines. It is possible to encode the pixel data in a way that does not use LZW encoding but will be correctly decoded by the LZW decoding algorithm. The comments in the code seem to indicate that he originally wrote the pixel data using essentially no encoding, which can be done by writing each 8-bit pixel byte in a 9-bit field. This actually expands the file size. He later (February 1999) devised a clever way to get a sort of [run-length-encoding](https://en.wikipedia.org/wiki/Run-length_encoding)-like encoding that does achieve some compression on most image files, but falls considerably short of what LZW encoding would accomplish.

The LZW patents all [expired in July 2004](https://en.wikipedia.org/wiki/GIF#Unisys_and_LZW_patent_enforcement), so GIF-writing programs have been free to use the LZW algorithm unencumbered since then.

More than 16 years later, why was Pillow still not using LZW encoding for GIF files? I have no answer for that. But last summer I had a need to write GIF files from Python and found Pillow unsatisfactory because of the needlessly large files it was writing. So I dragged out a LZW demo encoder and decoder I wrote some years ago and started hacking. You can find the results in <https://github.com/raygard/giflzw/> and some documentation at <https://raygard.github.io/giflzw/>.

I worked some more on this code and incorporated it into Pillow's [``GifEncode.c``](https://github.com/python-pillow/Pillow/blob/master/src/libImaging/GifEncode.c) module. This involved adding my encoder to [``GifEncode.c``](https://github.com/python-pillow/Pillow/blob/master/src/libImaging/GifEncode.c) and also reworking the rest of the code that accepts unencoded pixel bytes and uses the encoder to emit the encoded bytes in a fashion that is transparent to the caller. Aside from some changes to the [``Gif.h``](https://github.com/python-pillow/Pillow/blob/master/src/libImaging/Gif.h) header, no other changes were needed to the Pillow source code.

I am pleased that my pull request was accepted with only a few slight tweaks asked of me by the Pillow maintainers. They don't know me at all, so I'm gratified that they have some faith in my code. I tested it really well and have pored over every line looking for possible flaws, but it's hard to be sure that it's truly correct. I have my fingers crossed that no one ever does find a defect in it.

So is this the end of the Pillow / PIL / GIF story? Not for me. There are still a lot of problems with the way Pillow handles animated GIF files. I am hoping to help remedy that, but I think it will require a major overhaul of the [``GifImagePlugin.py``](https://github.com/python-pillow/Pillow/blob/master/src/PIL/GifImagePlugin.py) module.

Also, Pillow currently writes all GIF files with 8-bit pixel data fields, as would be necessary if the palettes (color tables) are the maximum size of 256 entries. But GIF files can have as few as two colors (e.g. monochrome black and white image files), and the GIF color tables can have 2, 4, 8, 16, 32, 64, 128, or 256 entries. This means the unencoded pixel fields can be from one to eight bits in length. My encoder is ready to go for that. Some changes will be needed elsewhere in Pillow to accommodate that, but it will result in some GIFs being quite a bit smaller than they are with the current code. I intend to look into what it will take to do this.
