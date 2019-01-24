---
title: An Update
index: true
layout: post
createdAt: 2019-01-24T21:24
updatedAt: 2019-01-24T21:24
---
It's been almost a month since my last [post](/2018/12/28/2019-and-the-purpose-of-this-blog), in which I detailed what I hoped to achieve in regards to this blog, and to a slightly further extent this year.

Firstly let's talk about [jrnl](https://github.com/andrewpillar/jrnl). In the previous post I mentioned how jrnl was pretty much feature complete, and how I was fairly happy with the quality of the code. Turns out this was not the case as I performed a rather large [refactoring](https://github.com/andrewpillar/jrnl/commit/f8aaa246fe4480129650c5a8817b3ad4780cf3bf) of the entire code base. And yes, you read that commit summary correctly, I decided to axe all the unit tests. Instead they have been replaced with a collection of shell scripts that test the tool's functionality. After that, the rest of the commits were documentation/testing related, what with the final few being for implementing support for Atom and RSS feed generation. During this process I was able to actually fix lots of incredibly stupid mistakes I had made in my [cli](https://github.com/andrewpillar/cli) library. In retrospect I should really be more careful when it comes to programming, and take my time.

[mgrt](https://github.com/andrewpillar/mgrt) is another tool I have been working on, as I also mentioned in the previous post. This is a tool for managing migrations on SQL databases. Right now it only supports SQLite, PostgreSQL, and MySQL. It is coming along nicely, and I am so far pleased with how it is turning out. I think all that is left for me to do is clean up some of the code, thoroughly test it, and document it.

And with regards to my personal life outside of work, and my hobbyist programming, things are going fine. I've been reading more, which is nice. However I still haven't made it as far through Dune as I would have hoped. My intention is to finish reading the first book in the trilogy before Villeneuve's adaptation hits theatres, because I just have to be that jackass who enters a conversation saying something along the lines of, "Well, in the books...,"

Anyway, I have this side-project I've been working on these past few months on and off. Progress is being made slowly, and I am pleased with how it is turning out. When it comes further along I'll perhaps talk about it more here.
