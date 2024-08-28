---
layout: post
title: "CRaCing the JRuby Startup Nut"
date: "2024-08-16"
author: Charles Oliver Nutter
---

Hello friends! I'm reentering the blogspace with a quick post about improving JRuby startup time with CRaC (Coordinated Restore at Checkpoint)! If that sounds intriguing, read on!

The Hardest Problem in JRuby
----------------------------

JRuby is the most successful and widely-deployed alternative Ruby implementation, but is frequently overlooked as an option due to several misconceptions:

### "JRuby won't work for my app because we use C extensions"

Most popular C extensions have JRuby equivalents (using a similar JRuby extension API), and others can be replaced by equivalent JVM libraries. Many applications will work out of the box with a simple "bundle install".

### "A Java-based Ruby can't possibly be faster than one implemented in C"

JRuby generally performs and scales better than even the latest Ruby versions with JIT capabities, and the legendary JVM garbage collectors prevent memory management from being a bottleneck. In fact, JRuby's JVM-based extension API means that even "native" libraries frequently outperform the C version of Ruby.

### "My development workflow will have to change if I move to JRuby"

The development workflow when using JRuby is largely identical to using CRuby, and we've worked very hard to make sure command-line tools behave the same.

However, hidden in this last item is a clue to the real JRuby challenge: **startup time**.

Why Is This So Hard?
--------------------

There's many factors impacting startup time for JVM-based applications, and JRuby is no different.

* JVM applications start out as interpreted bytecode. The JVM needs to execute and profile that code before it can be converted to native code.
* Most of the core JDK classes also start out the same way, requiring some amount of warm-up time before they are fully optimized.
* JRuby itself starts out cold, which means our Ruby parser, compiler, interpreters, and core libraries take a bit of time to get going. We eventually convert Ruby scripts to bytecode, which is then interpreted and profiled like other JVM classes.
* Simply loading all those classes and all that bytecode every time the JVM starts has a significant cost.

Many projects have been launched with varying degrees of success to reduce the time between launching your application and running optimized code:

* Projects like **AppCDS** (Class Data Store for Applications) save some of this "warm-up" work between runs so that future invocations get start more quickly.
* AOT compilers like **GraalVM Native Image** produce a pre-optimized native binary based on a static snapshot of your application and its dependencies. But these options frequently cripple core JDK APIs like MethodHandles (used heavily by dynamic languages like JRuby) or require extra configuration to allow reflection and runtime loading of code.
* Pre-loading frameworks like **Drip**, **Nailgun**, and similar tools for Ruby like **Theine** or **Spring** attempt to keep a background process "warm" and ready to be used. Unfortunately hooking up an active TTY to a background process can be buggy, and tools like Nailgun that keep a single persistent JVM running can leak threads and other resources.

I covered some of the trade-offs of AOT compilation and other tricks for startup time in my previous post (five years ago, whew!), [Start It Up: Improving JRuby's Startup Time](http://blog.headius.com/2019/09/jruby-startup-time-exploration.html).

I'm happy to report that after years of projects like JRuby complaining about startup time, serious effort are being made to solve this long-standing JVM problem. This post will cover one such effort: OpenJDK's **Project CRaC**!

A Powerful Project with an Unfortunate Name
-------------------------------------------

Project CRaC (Coordinated Restore at Checkpoint) is an effort to bring checkpoint-and-restore technology to JVM applications.

Checkpoint-and-restore is based on a Linux feature called Checkpoint and Restore In Userspace (CRIU) and allows users to capture an image, or "checkpoint", of a currently-executing process. This image can then be "restored" as a new process, continuing to execute as though nothing happened. UNIX fiends might think of it as a "deferred fork" where the parent immediately executes and the child can be continued later on, as many times as you want.

There's obviously a number of challenges for such an invasive operation (some that are similar to forking): open files and sockets must be restored, system resources claimed by the original process must be duplicated or re-acquired, threads can't survive the transition, and the new process may have little awareness that it has been restored from a previously-acquired image. Applied to the JVM, many of these issues are amplified: the JVM itself spins up many threads for GC and other bookkeeping, JVM apps may not realize how many files or sockets they have open, and a JVM is a big process with lots of system requirements.

That's where CRaC comes in.

Project CRaC aligns JVM behavior with the requirements of a CRIU checkpoint by restarting runtime-level threads, patching up JDK internal state after the restore, and providing mechanisms for users to explicitly handle releasing and reacquiring system resources like open files and sockets or native OS libraries and memory.

A short introduction to CRaC is provided by Azul, one of the primary drivers of this technology: [What is CRaC?](https://docs.azul.com/core/crac/crac-introduction). Numbers provided in that post demonstrate the enormous potential; applications based on Spring Boot, Quarkus, and Micronaut can startup in a fraction of the time of a full, cold JVM process.

Obviously, we want JRuby users to benefit from CRaC too!

JRuby on CRaC
-------------

While it is true that JRuby compounds the startup challenges of the JVM, most of what we do at startup can fit well into a CRaC image:

* Loading JRuby and its many component classes; this works like any other JVM framework.
* JRuby's cold parser, compiler, and interpreter can be warmed up prior to checkpointing.
* Ruby can also be pre-compiled to JVM bytecode and included in the image. When combined with a few warmup "training" cycles, you could see a JRuby CRaC restore leap directly into optimized Ruby code.

We'll walk through a simple example of using CRaC to checkpoint and restore a JRuby command-line operation.

Prerequisites
-------------

This first example will be on Linux, to allow CRIU to work natively. We'll explore options for MacOS and Windows later.

You'll want to download:

* A modern Linux environment.
* Azul's "Zulu" build of OpenJDK with CRaC support.
* https://cdn.azul.com/zulu/bin/zulu22.32.17-ca-crac-jdk22.0.2-linux_x64.tar.gz
* A clone of JRuby's `crac` branch, which adds runtime and command-line support for checkpointing.

### Linux

The only requirement is that the Linux you use support CRIU. There's information about compatibility on the CRIU site here: LINK

For this experiment I'm using a current release of Ubuntu Server on x86_64.

### JDK

The currently available builds of Azul Zulu with CRaC will work for most of this example, but there's a more recent build that fixes BLAH

You can do all JRuby build steps of this example using the same Zulu JDK you just installed. Just set env JAVA_HOME to the Zulu path, wherever you unpacked or installed Zulu.

Your clone of JRuby can be built using the provided Maven launcher:

* The provided Maven wrapper `./mvnw`. Run it without any arguments and it will build your JRuby distribution.
* Put the `bin/` in PATH and you're ready to try out JRuby with CRaC!

```
~/work/jruby $ jruby -v
jruby 9.4.9.0-SNAPSHOT (3.1.4) 2024-08-27 b3ad13d90d OpenJDK 64-Bit Server VM 22.0.2+9 on 22.0.2+9 +jit [x86_64-linux]
```

It Works! But At What Cost?
---------------------------

We'll start by getting a baseline time to print out version strings from the JDK and from JRuby.

First, `java -version`, which will mostly be native code:

```
~/work/jruby $ time java -version
openjdk version "22.0.2" 2024-07-16
OpenJDK Runtime Environment Zulu22.32+17-CRaC-CA (build 22.0.2+9)
OpenJDK 64-Bit Server VM Zulu22.32+17-CRaC-CA (build 22.0.2+9, mixed mode, sharing)

real	0m0.037s
user	0m0.017s
sys	0m0.012s
```

And then `jruby -v`, which loads only a few classes in JRuby, but is all Java code:

```
~/work/jruby $ time jruby -v
jruby 9.4.9.0-SNAPSHOT (3.1.4) 2024-08-27 b3ad13d90d OpenJDK 64-Bit Server VM 22.0.2+9 on 22.0.2+9 +jit [x86_64-linux]

real	0m0.261s
user	0m0.649s
sys	0m0.039s
```

So booting up the rest of OpenJDK and a small part of JRuby adds nearly a quarter-second.

How long does it take to fully boot JRuby and print "hello"?

```
~/work/jruby $ time jruby -e 'puts("hello")'
hello

real	0m1.804s
user	0m5.569s
sys	0m0.161s
```

First, JRuby with no extra flags:

```
~/work/jruby $ time jruby -e 'puts("hello")'
hello

real	0m1.804s
user	0m5.569s
sys	0m0.161s
```

This is a pretty typical result on most local Linux dev systems, and when compared with the standard C implementation of
Ruby, you can see why faster startup time is our most often-requested feature:

```
~/work/jruby $ time ~/.rubies/ruby-3.3.4/bin/ruby -v
ruby 3.3.4 (2024-07-09 revision be1089c8ec) [x86_64-linux]

real	0m0.014s
user	0m0.005s
sys	0m0.008s

~/work/jruby $ time ~/.rubies/ruby-3.3.4/bin/ruby -e 'puts("hello")'
hello

real	0m0.062s
user	0m0.047s
sys	0m0.014s
```

So that's about 30 times faster. But remember we're including the cost of booting OpenJDK and JRuby. How much time is actually spent printing?

```
~/work/jruby $ time jruby -e 't = Time.now; puts("hello"); puts Time.now - t'
hello
0.002096

real	0m1.843s
user	0m5.535s
sys	0m0.195s
```

The JRuby time varied on my machine from as low as 0.00197 up to 0.0022, but it's clearly a small amount for both implementations and shows the impact of booting each runtime.

So, we have two options to improve the situation on a typical JVM:

* Boot faster
* Don't boot.

Of course we are always looking for ways to speed up JRuby's boot time, but there's only so much we can do. A lot of code just has to load and run cold to set up a fully-prepared Ruby runtime on the JVM.

We have a few ways to avoid doing as much work at boot time, but anything less than a full boot generally impacts the completeness of the JRuby runtime environment.

### Improving Startup on a non-CRaC JDK

My article on JRUBY STARTUP ARTICLE AND LINK

Since we want to have a full-booted JRuby runtime, we will just time JRuby's `--dev` mode for comparison:

```
~/work/jruby $ time jruby --dev -e 'puts("hello")'
hello

real	0m1.255s
user	0m1.944s
sys	0m0.157s
```

We can also use AppCDS to get the best-possible JRuby startup time on the current JDK:

```
~/work/jruby $ gem install jruby-startup
Successfully installed jruby-startup-0.0.6
Parsing documentation for jruby-startup-0.0.6
Done installing documentation for jruby-startup after 0 seconds
1 gem installed

~/work/jruby $ generate-appcds 
*** Outputting list of classes at /home/headius/work/jruby/lib/jruby.list


*** Generating shared AppCDS archive at /home/headius/work/jruby/lib/jruby.jsa

<lots of output omitted>

*** Success!

JRuby versions 9.2.1 or higher should detect /home/headius/work/jruby/lib/jruby.jsa and use it automatically.
For versions of JRuby 9.2 or earlier, set the following environment variables:

VERIFY_JRUBY=1
JAVA_OPTS="-XX:G1HeapRegionSize=2m -XX:SharedArchiveFile=/home/headius/work/jruby/lib/jruby.jsa"

~/work/jruby $ time jruby --dev -e 'puts("hello")'
hello

real	0m0.834s
user	0m1.439s
sys	0m0.124s
```

So with all our best tricks, we've reduced the startup gap with C Ruby to around thirteen times. It's better, but still noticeably slower, and this effect is magnified when more Ruby code must be loaded to run a command line.

But what if we could pre-boot the JVM and JRuby, since that's largely the same every time, and then save that moment in time for future runs? Actually let's just do exactly that using CRaC!

Capturing a checkpoint
----------------------

The `crac` branch of JRuby contains some modifications to our shell script launcher to support checkpointing:

```
    --checkpoint[=path]     Save a CRaC checkpoint to path, defaulting to ./.jruby.checkpoint
```

A bit of logging provides some information as the checkpoint capture proceeds:

```
~/work/jruby $ jruby --checkpoint
Aug 28, 2024 8:34:52 AM jdk.internal.crac.LoggerContainer info
INFO: Starting checkpoint
Aug 28, 2024 8:34:52 AM jdk.internal.crac.LoggerContainer info
INFO: /home/headius/work/jruby/lib/jruby.jar is recorded as always available on restore
CR: Checkpoint ...
Killed
```

A checkpoint has been captured and the original process killed. Can you guess how we run JRuby from a restored checkpoint?

```
    --restore[=path]     Restore a CRaC checkpoint from path, defaulting to ./.jruby.checkpoint
```

Let's give it a try with our print example!

```
~/work/jruby $ time jruby --restore -e 'puts("hello")'
hello

real	0m0.116s
user	0m0.172s
sys	0m0.092s
```

Holy toledo! We're now less than twice the startup time of C Ruby, and we didn't even have to use `--dev` mode.

What's actually happening here?

CRaC support in JRuby
---------------------

Luckily most of the heavy lifting has been done by the excellent CRaC engineers.

The `--checkpoint` flag just points the JRuby launcher script at a different `main` class: `CheckpointMain.java` LINK

This is a work-in-progress; we'll explore how the pieces fit together in Part 2 but I provide a high-level overview here.

We use the CRaC API to do a few things:

* Run some code before capturing the checkpoint

  This is what you might expect: boot JRuby right up to the point at which we're about to execute code. Code in the base `PrebootMain.java` class runs some code through a throw-away JRuby instance to make sure all the critical classes have been loaded, and then boots up the JRuby instance we plan to capture.

https://github.com/jruby/jruby/blob/5b417f5ab90c9db94e93140a41479667fe95ec7e/core/src/main/java/org/jruby/main/PrebootMain.java#L35-L45

* Prepare JRuby for checkpointing

  Because the checkpoint will be restored in a completely new process, there's some bookkeeping necessary to ensure JRuby adapts to the new environment. We tuck away a reference to the pre-booted JRuby runtime before the checkpoint, and fix it up after the restore.

https://github.com/jruby/jruby/blob/c45c462b486a9ebaa0d5774e62e1fdac844632bd/core/src/main/java/org/jruby/main/PrebootMain.java#L47-L54

* Request the checkpoint

  At this point CRaC steps in, performing its own graceful hand-holding for the JVM before requesting a CRIU checkpoint be captured.

https://github.com/jruby/jruby/blob/c45c462b486a9ebaa0d5774e62e1fdac844632bd/core/src/main/java/org/jruby/main/CheckpointMain.java#L20-L29

The `--restore` flag uses the normal JRuby `Main` class, but when it detects our pre-booted JRuby instance in memory, it uses that rather than start a new one:

https://github.com/jruby/jruby/blob/e3de925818ab356374b7c53a5154802e791da141/core/src/main/java/org/jruby/Main.java#L259-L267

And execution proceeds as normal from this point, except we've just skipped most of the expensive part of JRuby's startup!

Digging Deeper
--------------

This is just a first taste of how checkpointing can help JRuby finally approach parity with C Ruby's startup time, without sacrificing any functionality in JRuby itself. In Part 2, I'll dig deeper into using CRaC to speed up larger commands like `gem` and `rails` and we'll explore a few ways to tweak the checkpoint for specific profiles.

If you like this article
------------------------

The work to finally, maybe, hopefully fix JRuby's startup-time woes using CRaC has been done by me as part of my work to support JRuby users at Headius Enterprises. We offer multiple support tiers for everyone from curious JRuby first-timers to professional JRuby application developers. I also depend on sponsorships from readers and users like you. If you appreciate this article or the JRuby project, sign up for support or sponsorship today!

