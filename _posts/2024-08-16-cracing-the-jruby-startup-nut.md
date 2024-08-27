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

For Linux, I'm using a current release of Ubuntu (ADD VERSION AND LINK) running on x86_64.

The currently available builds of Azul Zulu with CRaC will work for most of this example, but there's a more recent build that fixes BLAH

You can do all JRuby build steps of this example using the Azul build; other than providing CRaC support, it is a standard JVM.

Set JAVA_HOME to the JDK path, wherever you unpacked or installed Zulu.

Your clone of JRuby can be built using the provided Maven launcher: `./mvn` is enough to get a fully-working JRuby distribution. Put `bin/` in PATH and you're ready to try out JRuby with CRaC!

Something Simple
----------------

We'll start by exploring the simplest example, evaluating the number `1` and exiting. We'll just use `time` because we want full end-to-end process time.

First, JRuby with no extra flags:

headius@nuttergros-server:~/work/jruby$ time jruby -e 1

real	0m1.847s
user	0m5.686s
sys	0m0.214s
headius@nuttergros-server:~/work/jruby$ time jruby -e 1

real	0m1.855s
user	0m5.521s
sys	0m0.201s
headius@nuttergros-server:~/work/jruby$ time jruby --dev -e 1

real	0m1.223s
user	0m1.847s
sys	0m0.167s
headius@nuttergros-server:~/work/jruby$ time jruby --dev -e 1

real	0m1.254s
user	0m1.891s
sys	0m0.183s
headius@nuttergros-server:~/work/jruby$ time jruby --dev --disable-gems -e 1

real	0m0.883s
user	0m1.235s
sys	0m0.118s
headius@nuttergros-server:~/work/jruby$ time jruby --disable-gems -e 1

real	0m1.293s
user	0m3.188s
sys	0m0.170s
headius@nuttergros-server:~/work/jruby$ time jruby --dev --disable-gems -e 1

real	0m0.864s
user	0m1.235s
sys	0m0.115s
headius@nuttergros-server:~/work/jruby$ time jruby -e "t = Time.now; 1; puts Time.now - t"
0.000171

real	0m1.862s
user	0m5.594s
sys	0m0.179s
headius@nuttergros-server:~/work/jruby$ time jruby -e "t = Time.now; 1; puts Time.now - t"
0.000163

real	0m1.852s
user	0m5.706s
sys	0m0.202s


For Rails show base bootup is still not fast

need flag for checkpoint files to reopen? logs in rails kill restore

TODO:

* CRaC image size comparison with and without compression, perhaps trying some GC runs to clean things up
* Docker instructions based on MacOS and Windows if possible.
* Bake JRuby crac data into the image for faster startup, rather than loading it from a virtual volume.
* Call for sponsors.
* SnapStart on Lambda uses CRaC