---
title: Patience in Programming
index: true
layout: post
createdAt: 2019-05-11T19:46
updatedAt: 2019-05-11T19:46
---
I am an impatient man (if my sporadic typos, and last minute edits late at night
on my blog didn't make it obvious already). Recently this has started to stick
out to me much more, especially in regards to programming. More often than not I
would find myself just writing code haphazardly, then let the
compiler/interpreter point out the errors for me, some not so obvious, others
more obvious than I would have liked. This is a rather reckless approach to
take when it comes to development. I find that it prevents me from actually
thinking about what I am doing, and results in me producing tools and code that
are in unsatisfactory state.

Take [mgrt](https://github.com/andrewpillar/mgrt) for example, just today I
added some small features to it. Nothing to special, just storing the revision
queries in the database and better supporing MySQL. And that was what stuck out
to me. The support for MySQL in this tool was completely broken. It failed to
parse the timestamps from the database, and didn't even have SSL supported
properly either.

This came down to two things, the first being that I developed this tool for
myself, and I'm more of a PostgreSQL person, and the second being that I was
impatient with the development of this tool. I never took the time to properly
test it using the different databases it claims to have supported. There were
other things that were poorly implemented too, such as the SSL support for
MySQL. I have fixed these today, however it still bothers me that I saw fit to
release this tool, to little fanfare, in such a broken state. Had I taken my
time with it, then this could have been avoided. Even the feature that I added
today, whereby the queries are now stored in the database, is something I
initially considered when I started writing it. However I neglected it due to
my impatience.

Going forward, I do not expect to be the sort of person who will write code that
will compile/run on its first time. But it's an exercise I would like to
practice. Write some code, or make some modifications to a code base, but before
compiling or running the program I should stop and think about the changes I've
made instead of letting the compiler/interpreter hold my hand.

I realise that this post doesn't have much substance, and in retrospect might
come off as being common sense. But, the fact that I showed off
[mgrt](https://github.com/andrewpillar/mgrt) to some people in such a broken
state really bothered me. I'm dissapointed that I didn't hold myself to a higher
standard. On the bright side, I appear to be the only one using this tool so the
only thing damaged was my self-esteem. No doubt if someone else used it for a
MySQL database they would have opened up an issue on the GitHub repository in
no time.
