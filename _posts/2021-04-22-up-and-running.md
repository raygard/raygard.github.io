---
layout: post
title:  Up and running
date:   2021-04-22 13:00:00 -0600
---

<!-- categories: do not work yet -->

Maybe I'm wrong, but it seems many blogs start with a post about what was involved in getting the blog going. This [isn't really a blog], but anyway...

[isn't really a blog]: {{ site.baseurl }}/about/index.html#not-a-blog

This site is hosted on GitHub at <https://raygard.github.io/>, but using a custom domain <https://raygard.net>.

I had already done a little with Jekyll and GitHub Pages using the [Just the Docs](https://github.com/pmarsceill/just-the-docs) theme. I wanted to set up a site or page where I could put any sort of stuff, initially planning to point to some work I've put up in GitHub repositories and some open source contributions I've made or will make. Just the Docs is aimed at documentation and it seemed fine for some [work I've done so far](https://raygard.github.io/giflzw/) (more about that in the next post). I needed something better suited for blogging.

<!-- more -->

So I spent more time than I should have, looking for a good Jekyll theme and figuring out how to use it, coming from little Jekyll and Markdown experience, and none with Ruby. I poked around for a theme that works with GitHub Pages, decided the "supported" themes were not for me. I looked at [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/), very popular and capable. But it seemed a bit too much for me and I wasn't sure I wanted a two-column theme. I looked for something more minimal, and looked at several before I found [Whiteglass](https://github.com/yous/whiteglass) by Chayoung You.

This seems to be a good fit, at least for now. I did wrestle with getting it set up so that I could run it locally and also have it work at GitHub Pages. A couple keys were getting things right in ``_config.yml`` and ``Gemfile``. Specifically, I think, putting ``remote_theme: yous/whiteglass`` in ``_config.yml`` and _not_ putting in ``theme: yous/whiteglass``, and putting 
``gem "jekyll-remote-theme"`` and ``gem "jekyll-whiteglass"`` in ``Gemfile``.

This now builds and serves locally and at GitHub Pages, without giving me that email "Page build warning" saying "The page build completed successfully, but returned the following warning ...  You are attempting to use a Jekyll theme, "jekyll-whiteglass", which is not supported by GitHub Pages."

It took me a while to figure out how to set up the DNS records to make <https://raygard.net> work right. I put ``www.raygard.net`` in the Custom domain field in the Pages section of the Settings page for <https://github.com/raygard/raygard.github.io>. I am using Namecheap as my domain name registrar and DNS provider. At Namecheap, under Advanced DNS, I set up a CNAME record to point ``www.raygard.net`` to ``raygard.github.io``, and four A records to point ``raygard.net`` to 185.199.108.153, 185.199.109.153, 185.199.110.153, and 185.199.111.153. I don't know how other DNS providers' interfaces work, but here, I had to use ``@`` to represent ``raygard.net`` in the Host field for the A records, and just a bare ``www`` (not ``www.raygard.net``) under Host for the CNAME.

I changed or customized a few other things. I downloaded the entire whiteglass repo and copied and modified these:

``assets/main.scss``: I changed the ``$brand-color`` to make it stand out a bit more on my monitor, and hopefully on yours too.

``_data/navigation.yml``: Modified for my own nav header.

``_includes/head_custom.html``: Inserted ``<link rel="icon" type="image/x-icon" href="{{ "/favicon.ico" | relative_url }}">`` to get a simple favicon. Still need a better one...

``_layouts/archive.html``: Made the "archive" page heading read "Posts" instead of "Blog Archive". 

``_sass/whiteglass/_layout.scss``: under ``.site-footer``, I made it smaller and left justified, and under ``.post-title``, I used a 32px font instead of 42px. I hope it still looks OK on all devices.

``_sass/whiteglass.scss``: Had to add this even though I didn't change it, to make the ``_layout.scss`` mods work.

That's all I've customized so far. Feel free to copy the repo and make appropriate modifications (using only your own content of course!)

