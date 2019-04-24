---
title: Another update
index: true
layout: post
createdAt: 2019-04-23T22:47
updatedAt: 2019-04-23T22:47
---
It has been too long since my last post in which I [announced mgrt](/programming/2019/01/27/mgrt-simple-sql-migrations), the simple SQL migration tool. Has much happened between then and now? Well, I showed off mgrt to some people with little fanfare, and got around to implementing TLS support for the tool. Right now it supports TLS for PostgreSQL, and to a certain extent MySQL/MariaDB. And during my further developments of the tool itself I have come to realise how much I truly despise working with MySQL. My feelings for it are all superficial, so they hardly warrant a tangent.

Going forward however I do intend on wanting to write more frequently, ideally once a month at least. Thereforce I can actually use this blog as I [originally intended](/ramblings/2018/12/28/2019-and-the-purpose-of-this-blog). I feel like I have padded out this post with too much fluff for now, so I will get to the more somewhat meaningful updates.


## Reading

I have finally finished reading Dune, it took me only four months. What can I say, I am known to be a slow reader. I like to mull over the details, and let the story sit for a while before I proceed. Ok, that's half of it, the other half was just due to laziness. Needless to say I am now ever more excited for Villeneuve's adaptation of the book, it will be a challenging story to adapt to the screen, but I have faith in his ability nonetheless. I sincerely hope when the movie does hit theatres, it will do for sci-fi, what Lord of the Rings and Game of Thrones both did for fantasy.

The Witcher series is slowly approaching its end. The Tower of the Swallow is currently sitting on my desk with only a single chapter to go. Then comes The Lady of the Lake. Similar to Dune, my hope is to finish these books before the TV series drops. And, to be brutally honest, I am not as excited for this adaptation as I am Dune. I am approaching it with cautious optimism. I am not familiar with the showrunner's previous work, so I'm trying to not set myself up for dissapointment.

## Work

The past month of work has been somewhat stressful for me. This has lead me to take a week off to focus on recuperating, and will give me ample time to work on both [mgrt](https://github.com/andrewpillar/mgrt) and [jrnl](https://github.com/andrewpillar/jrnl). Primarily, adding unit tests to each of these. Yes, unit tests. If you read through the commit logs for each you will notice that I originall nuked them in favour of some shell scripts that tested the functionality of the programs themselves. Well, I decided that was a fairly naive thing to do, and reverted back to writing good old fashioned unit tests, where applicated. Also as mentioned previously I need to iron out the TLS support for MySQL/MariaDB in mgrt, and update the documentation surrounding the TLS capabilities.

This also leads me into my next announcement -- of sorts. The side-project I have been working on, as mentioned briefly [here](/ramblings/2018/12/28/2019-and-the-purpose-of-this-blog), is coming along nicely. At the end of this week I would hope to have an MMMVP (minimum-minimum-minium viable product) of this ready to show to some people. At this point I honestly don't know who. Perhaps my avid readership of this blog, who, judging by the NGINX access logs, mainly consist of web crawlers.

## End

Well, that about covers it for now. Come May I hope to have another post be made, and by then I would hope to have made some substantial progress on my side-project. I have big hopes for this thing, the only problem with actually getting it out there is my incredibly limited reach, something I should work on.

Also, yes. I do realise I said I should stop doing these last minute late night blog posts. Well, old habbits and all that. No doubt this text is riddled with typos. But in the interest of not wanting to keep the web crawlers waiting I'll deal with editing out the typos at some other time.
