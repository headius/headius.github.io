---
layout: post
title: "JRuby and Leyden: Even Better Startup"
date: "2025-09-26"
author: Charles Oliver Nutter
published: true

---

At the end of my post on JRuby and JDK 25 startup time features, I teased a bit of the unreleased improvements from [Project Leyden](https://openjdk.org/projects/leyden/). It turns out the latest commits improve startup time **even more**, so it seems worth posting a quick follow-up!

Project Leyden is LIT
=====================

Of the many OpenJDK projects I follow, Leyden has been near the top as far as activity and interest. In the past month, there's been 527 commits to all branches... over 15 commits per day. And this doesn't include commits being done by contributors on their own repositories. It's exciting to watch!

After my recent post, [Aleksey Shipilëv](https://bsky.app/profile/shipilev.bsky.social) reached out to me on Bluesky:

[![Aleksey Shipilëv Bluesky post about recent Leyden improvements](/images/aleksey_shipilëv_leyden_followup.png)](https://bsky.app/profile/shipilev.bsky.social/post/3lzm3ss3fjs22)

If you know Aleksey, you know to listen when he makes a suggestion. Naturally, I got to work building my own "premain" Leyden JDK build to try things out.

Baseline Startup
----------------

To refresh your memory, our previous best time — with AOTCache and JRuby's "dev" mode (which disables many optimizations) — was a lovely little **423ms**, down from an unaided 943ms.

We'll try our baseline startup time training again, but this time I'll leave off the `--dev` flag and let JRuby and the JVM fully optimize and record everything.

(Recall that the `--nocache` flag turns off JRuby's automatic use of AppCDS, which is incompatible with AOTCache.)

Same training run:

```text
$ jruby --nocache -J-XX:AOTCacheOutput=jruby.aot baseline_training.rb 
...
Reading AOTConfiguration jruby.aot.config and writing AOTCache jruby.aot
AOTCache creation is complete: jruby.aot 110788608 bytes
Removed temporary AOT configuration file jruby.aot.config
```

Note this new AOTCache is 110MB, a full 50MB larger than the 60MB from before. There must be something good in there... let's try it.

(To make things a bit easier to read, I'll be using `TIMEFMT="run time: %mE"` to format zsh's `time` command.)

```text
$ time jruby --nocache -J-XX:AOTCache=jruby.aot -e 1
run time: 353ms
```

With the latest Leyden "premain" patches, JRuby's base startup — sacrificing no features and no optimizations — is down to **353ms**, over 16% faster than our previous best!

gem list
--------

One thing I didn't show in my previous post was the performance of `gem list` without explicitly training the AOTCache for `gem list`. Recall that without any help, `gem list` for 88 gems took **1546ms**, and our best time was **815ms**.

Here's a `gem list` time with the latest Leyden "premain" AOTCache patches, trained only for baseline JRuby startup:

```text
$ time jruby --nocache -J-XX:AOTCache=jruby.aot -S gem list > /dev/null
run time: 919ms
```

Not bad! A single, simple training run has reduced `gem list` time by 40%, and we've achieved a sub-second run time without specifically training the AOTCache for this command. This means you can use a simple training run for your JRuby development environment and still see a massive improvement in startup time for arbitrary Ruby commands.

Of course, we want to see what happens if we specifically train the updated AOTCache for `gem list`. Targeted training like is useful when you have one type of command you must run repeatedly, or in a serverless Amazon Lambda-style environment that needs to quickly launch a small tool.

The training run proceeds as before:

```text
$ jruby --nocache -J-XX:AOTCacheOutput=jruby.aot gem_list_training.rb
...
Reading AOTConfiguration jruby.aot.config and writing AOTCache jruby.aot
AOTCache creation is complete: jruby.aot 175816704 bytes
Removed temporary AOT configuration file jruby.aot.config
```

And here's the `gem list` run time with our custom `AOTCache`:

```text
$ time jruby --nocache -J-XX:AOTCache=jruby.aot -S gem list > /dev/null
run time: 714ms
```

Another new record: we're down to **714ms**, beating the old record of 825ms by 13%, and we **haven't disabled any optimizations**.

What's Next?
============

Watch This Space
----------------

Even with these incredible results, I want to reiterate that it's early days for Project Leyden.

We still haven't achieved a "cold" `gem list` run time comparable to the 280ms from running it in-process in a loop, so clearly there's optimizations happening that aren't getting saved and restored.

And some of these AOTCache sizes might be prohibitively large; an uncached filesystem will take time to load up 100-200MB of AOTCache data, and that will be even worse if it's loaded across the network.

But wow it's exciting stuff. I have been telling JRuby users for years that "it's not our fault" and "the JVM folks are working on it". Finally, the dream of fast JRuby startup is coming true.

Other Options
-------------

My previous post neglected to mention other options for improving JRuby startup, like the [Project CRaC "checkpoint and restore" feature](https://blog.headius.com/2024/09/jruby-on-crac-part-1-lets-get-cracking.html) I blogged about previously and alternative JVMs like [IBM Semeru](https://developer.ibm.com/languages/semeru-runtimes/) (aka "OpenJ9") and its [Shared Class Cache](https://www.ibm.com/docs/en/semeru-runtime-ce-z/17.0.0?topic=sharing-introduction) and [JITServer](https://www.ibm.com/docs/en/semeru-runtime-ce-z/17.0.0?topic=documentation-jitserver-technology) features. We'll explore those and other options in an upcoming post!

## [Join the discussion on Reddit!](https://www.reddit.com/r/ruby/comments/1nr630x/jruby_and_leyden_even_better_startup/)

_JRuby Support for Your Project_
================================

_This is a call to action!_
---------------------------

JRuby development is made possible by our primary sponsor, [Headius Enterprises](https://headius.com), offering a range of professional development and support resources for your team. You can choose JRuby for your next project knowing that you've got the world's best JRuby experts standing by to help.

If you are interested in **scaling Ruby** to new heights, deploying your applications in **large enterprises**, or taking advantage of JVM features like **world-class JIT compilers**, **battle-tested libraries**, and **leading-edge AI tools**, you need to be using JRuby. Let us help you.

[Sign up for expert JRuby support from Headius Enterprises!](https://www.headius.com/jruby-support)

If you find this article or my work on JRuby to be interesting and useful, you can also sponsor me directly on GitHub. Your contributions help ensure the JRuby project keeps moving forward!

[Sponsor Charles Oliver Nutter on GitHub!](https://github.com/sponsors/headius)
