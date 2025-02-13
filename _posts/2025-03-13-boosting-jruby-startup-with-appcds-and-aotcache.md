---
layout: post
title: "Boosting JRuby Startup with AppCDS and AOT caching"
date: "2025-02-13"
author: Charles Oliver Nutter
---

Hello again friends! Today I'll show how you can improve JRuby startup right today, as well as a sneak preview of some exciting new developments in this area!

Application Class Data Store (AppCDS)
-------------------------------------

I'm still ramping up my blogging efforts, but I wanted to share the results of recent experiments using JRuby with Application Class Data Store and the new AOTCache feature being previewed in JDK 24

Around the times of Java 1.7 or 1.8, teams at Sun (and then Oracle) started shipping a JDK feature called [Class Data Sharing](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/class-data-sharing.html) (CDS). This was a new, specialized cache file that could be used to "pre-load" much of the runtime metadata from starting a JVM-based application. Initially, this only included the JDK's own class metadata, but by JDK 10 a new version of the feature was made available: [Application CDS](https://openjdk.org/jeps/310), basically a Class Data Store for your own applications.

We have experimented with this feature in JRuby over the years, but the user experience has been cumbersome and the file can get stale if you reinstall JRuby or switch to a different JDK. That appears to be improving with recent JDK releases.

### Automatic CDS with JRuby 10

In JDK 21, we can now request that the AppCDS archive be generated automatically, and then use it each time we start up. Here's JRuby without any help from AppCDS:

```text
$ time jruby -e 1                   
  2.35s user 0.16s system 165% cpu 1.518 total
$ time jruby -e 1
  2.73s user 0.21s system 183% cpu 1.603 total
$ time jruby -e 1
  2.37s user 0.14s system 180% cpu 1.385 total
$ time jruby --dev -e 1
  1.14s user 0.10s system 118% cpu 1.045 total
$ time jruby --dev -e 1
  1.19s user 0.10s system 120% cpu 1.067 total
$ time jruby --dev -e 1
  1.17s user 0.11s system 123% cpu 1.034 total
```

And JRuby with experimental "automatic" CDS support:

```text
$ time jruby -e 1
  1.81s user 0.11s system 184% cpu 1.044 total
$ time jruby -e 1
  1.95s user 0.12s system 192% cpu 1.078 total
$ time jruby -e 1
  1.80s user 0.11s system 188% cpu 1.011 total
$ time jruby --dev -e 1
  0.93s user 0.09s system 121% cpu 0.837 total
$ time jruby --dev -e 1
  0.95s user 0.09s system 119% cpu 0.878 total
$ time jruby --dev -e 1
  0.94s user 0.09s system 121% cpu 0.844 total
```

That's a huge improvement you'll get for free on JRuby 10! But today I also want to look at the new preview of AOT caching (Ahead Of Time data and code caching) in JDK24.

AOT Caching from Project Leyden
-------------------------------

OpenJDK's [Project Leyden](https://openjdk.org/projects/leyden/) has been working for the past several years on bringing AOT (Ahead-of-Time compilation) to the JVM without the limitations of a closed-world system like GraalVM's Native Image. The general feature is being called the "AOT cache", but there's a lot to unpack here.

AOT caching is similar to AppCDS in that it caches class files and their metadata, but it will eventually include optimization artifacts such as:

* Code profiles from previous runs, so we can jump into optimization earlier;
* Native code produced by the JIT for hot classes and methods during a training run;

These efforts are starting to bear fruit, and JDK24 includes the first version of AOT caching: [Ahead-of-Time Class Loading and Linking](https://openjdk.org/jeps/483).

How does this early version of AOT caching stack up against AppCDS?

### JRuby and JDK24 AOTCache

So given our numbers above for JRuby startup, let's try JRuby with JDK24's AOT caching.

First we need to run JDK24 with AOT caching flags that record training data. In this case, I make JRuby start up additional instances of itself repeatedly in a loop. (The `--nocache` flag here disables automatic AppCDS)

```text
$ jruby --nocache -J-XX:AOTMode=record -J-XX:AOTConfiguration=lib/jruby.aotconf -e '1000.times { r = org.jruby.Ruby.newInstance; r.evalScriptlet(%[require "rubygems"]); r.tearDown }'
$ jruby --nocache -J-XX:AOTMode=create -J-XX:AOTConfiguration=lib/jruby.aotconf -J-XX:AOTCache=lib/jruby.aot -e '100.times { org.jruby.Ruby.newInstance.tearDown }'
[2.506s][warning][cds] java.lang.ClassNotFoundException: DashE
[2.506s][warning][cds] Preload Warning: Cannot find DashE
[2.809s][warning][cds] Skipping jdk/internal/event/Event: JFR event class
AOTCache creation is complete: lib/jruby.aot
```

A few warnings later and we have an AOT cache of JRuby startup code. Here's the same baseline `-e 1` calls from above, but this time using the AOT cache:

```text
$ time jruby --nocache -J-XX:AOTCache=lib/jruby.aot -e 1                                                           
  2.73s user 0.17s system 277% cpu 1.047 total
$ time jruby --nocache -J-XX:AOTCache=lib/jruby.aot -e 1
  2.87s user 0.16s system 286% cpu 1.058 total
$ time jruby --nocache -J-XX:AOTCache=lib/jruby.aot -e 1
  2.82s user 0.14s system 279% cpu 1.057 total
$ time jruby --nocache -J-XX:AOTCache=lib/jruby.aot --dev -e 1
  0.84s user 0.08s system 126% cpu 0.732 total
$ time jruby --nocache -J-XX:AOTCache=lib/jruby.aot --dev -e 1
  0.85s user 0.07s system 127% cpu 0.722 total
$ time jruby --nocache -J-XX:AOTCache=lib/jruby.aot --dev -e 1
  0.85s user 0.07s system 127% cpu 0.722 total
```

We have a new winner for JRuby startup time! (ignoring more restricted options like [JRuby on CRaC](https://blog.headius.com/2024/09/jruby-on-crac-part-1-lets-get-cracking.html), which will see a **Part 2** post real soon!)

How Can I Use This Today?
-------------------------

JRuby 9.4 users on Java 11 or higher can use AppCDS today, and with JDK24 you can also experiment with AOT caching.

### JRuby 9.4 and AppCDS

All you need to try AppCDS on JRuby is to generate the AppCDS archive (`lib/jruby.jsa`):

* Make sure you're running JDK11 or higher.
* `gem install jruby-startup`
* `generate-appcds`

```text
$ gem install jruby-startup
Successfully installed jruby-startup-0.0.6
1 gem installed
$ generate-appcds
...
*** Success!

JRuby versions 9.2.1 or higher should detect /Users/headius/work/jruby/lib/jruby.jsa and use it automatically.
For versions of JRuby 9.2 or earlier, set the following environment variables:

VERIFY_JRUBY=1
JAVA_OPTS="-XX:G1HeapRegionSize=2m -XX:SharedArchiveFile=/Users/headius/work/jruby/lib/jruby.jsa"
$ time jruby --dev -e 1
  0.92s user 0.09s system 118% cpu 0.807 total
$ time jruby --dev -e 1
  0.93s user 0.09s system 121% cpu 0.838 total
...
```

The `jruby.jsa` file is automatically passed to the JVM when you start JRuby, boosting startup as seen at the top of this post. Newer JDKs will see a bigger improvement!

### JRuby 9.4 and JDK24 AOT Caching

You'll need to install a build of JDK24, which just dropped a [release candidate](https://jdk.java.net/24/) last week (Feb 6).

The same flags shown above will work with JRuby 9.4, but there's no `--nocache` flag to disable automatic CDS. Just make sure you delete the `lib/jruby.jsa` file before playing with AOT caching on JRuby 9.4.

Have fun, and let us know how it goes!

## [Join the discussion on Reddit!](https://www.reddit.com/r/javavirtualmachine/comments/1ioncqe/boosting_jruby_startup_with_appcds_and_aot_caching/)

_JRuby Support and Sponsorship_
===============================

_This is a call to action!_

_JRuby development is funded entirely through your generous sponsorships and the sale of commercial support contracts for JRuby developers and enterprises around the world. If you find my work exciting or believe it is important your company or your projects, please consider partnering with me to keep JRuby strong and moving forward!_

[Sponsor Charles Oliver Nutter on GitHub!](https://github.com/sponsors/headius)

[Sign up for expert JRuby support from Headius Enterprises!](https://www.headius.com/jruby-support)
