---
layout: post
title: "JRuby on CRaC Part 1: Let's Get CRaCking!"
date: "2024-09-11"
author: Charles Oliver Nutter
---

Hello friends! Today we start a series on improving JRuby startup time using the new OpenJDK [Project CRaC](https://openjdk.org/projects/crac/) (Coordinated Restore at Checkpoint)! If that sounds intriguing, read on!

The Hardest Problem in JRuby
----------------------------

JRuby is the most successful and widely-deployed alternative Ruby implementation, but is frequently overlooked as an option due to several misconceptions:

### _"JRuby won't work for my app because we use C extensions"_

Most popular C extensions have JRuby equivalents (using pure-Ruby or our similar JRuby extension API), and others can be replaced by equivalent JVM libraries. Many applications will work out of the box with a simple `bundle install`.

### _"A Java-based Ruby can't possibly be faster than one implemented in C"_

JRuby frequently performs and scales better than even the latest Ruby versions with JIT capabities, and the legendary JVM garbage collectors generally prevent memory management from being a bottleneck. In fact, JRuby extensions frequently optimize _better_ than C extensions because it's all just bytecode to the JVM.

### _"My development workflow will have to change if I move to JRuby"_

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

* Projects like **AppCDS** (Class Data Store for Applications) save some of this "warm-up" work between runs so that future invocations get start more quickly. This is generally limited to static class data, but can be "trained" with specific sets of classes for specific scenarios.
* AOT compilers like **GraalVM Native Image** produce a pre-optimized native binary based on a closed-world view of your application and its dependencies. Unfortunately this means giving up dynamic code loading and invokedynamic support APIs like MethodHandles (both used heavily by JRuby).
* Pre-loading and background-process launchers like **Drip**, **Nailgun**, and similar tools for Ruby like **Theine** or **Spring** attempt to keep a pre-booted process "warm" and ready to be used. Unfortunately hooking up an active TTY to a background process can be buggy, and tools like Nailgun that keep a single persistent JVM running can leak threads and other resources.

I covered some of the trade-offs of AOT compilation and other tricks for startup time in a previous post (five years ago, whew!), [Start It Up: Improving JRuby's Startup Time](http://blog.headius.com/2019/09/jruby-startup-time-exploration.html).

I'm happy to report that after years of projects like JRuby struggling with startup time, serious effort are being made to solve this long-standing JVM problem. This series will cover one such effort: OpenJDK's **Project CRaC**!

A Powerful Project with an Unfortunate Name
-------------------------------------------

Project CRaC (Coordinated Restore at Checkpoint) is an effort to bring checkpoint-and-restore technology to JVM applications.

Checkpoint-and-restore is based on a Linux feature called [Checkpoint and Restore In Userspace](https://criu.org/Main_Page) (CRIU) and allows programs to capture an image, or "checkpoint", of a currently-executing process. This checkpoint can then be "restored" as a new process (repeatedly), continuing to execute as though nothing happened. UNIX fiends might think of it as a delayed `fork` where the parent immediately exits and the child can be continued later on, as many times as you want.

There's obviously a number of challenges for such an invasive operation (some that are similar to forking): open files and sockets must be restored, system resources claimed by the original process must be duplicated or re-acquired, threads can't survive the transition, and the new process may have little awareness that it has been restored from a previously-acquired image. On the JVM, many of these issues are amplified: most JVMs spin up many threads for GC and other bookkeeping, JVM apps may not track how many files or sockets they have open, and the JVM itself is a big process with lots of system requirements.

That's where CRaC comes in.

Project CRaC aligns JVM behavior with the requirements of a CRIU checkpoint by restarting runtime-level threads, patching up JDK internal state after the restore, and providing mechanisms for users to explicitly handle releasing and reacquiring system resources like open files and sockets or native OS libraries and memory.

A short introduction to CRaC is provided by Azul, one of the primary drivers of this technology: [What is CRaC?](https://docs.azul.com/core/crac/crac-introduction). Numbers provided in that post demonstrate the enormous potential; applications based on Spring Boot, Quarkus, and Micronaut can start much faster with CRaC than they ever could with a cold JVM process.

Obviously, we want JRuby users to benefit from CRaC too!

Prerequisites
-------------

For simplicity, these early examples will be on Linux to allow CRIU to work natively. We'll explore options for MacOS and Windows in a future post.

You'll want to download:

* A modern Linux environment.
* A version of OpenJDK with CRaC support.
* A clone of [JRuby's "crac" branch](https://github.com/jruby/jruby/tree/crac), which adds runtime and command-line support for checkpointing. (This post will be updated to a release version once available.)

### Linux

The only requirement is that the Linux you use support CRIU. Any kernel version 3.11 or later should support CRIU, though newer kernels may have an improved set of features.

For these examples, I used the latest LTS version of [Ubuntu Server, 24.04.1 (Noble Numbat)](https://ubuntu.com/download/server)

### JDK

The easiest option is to download a build of [Azul Zulu with CRaC support](https://www.azul.com/downloads/?package=jdk-crac#zulu). Alternatively you can build it yourself from the [openjdk/crac repository](https://github.com/openjdk/crac), but you'll need recent patches from master for this demo.

Ensure the `JAVA_HOME` environment variable points at your installed JDK.

### JRuby

From here, getting a working JRuby is trivial.

```shell
git clone https://github.com/jruby/jruby.git
cd jruby
git checkout crac
```

Your clone of JRuby can be built using the provided Maven launcher:

```shell
./mvnw
```

Put the `bin/` dir in your `PATH` and you're ready to try out JRuby with CRaC!

```
$ export PATH=$PATH:`pwd`/bin
$ jruby -v
jruby 9.4.9.0-SNAPSHOT (3.1.4) 2024-08-27 b3ad13d90d OpenJDK 64-Bit Server VM 22.0.2+9 on 22.0.2+9 +jit [x86_64-linux]
```

JRuby Without CRaC
------------------

My previous article talked about [JRuby startup time](http://blog.headius.com/2019/09/jruby-startup-time-exploration.html) and how to get the most (least?) out of it with various strategies, but we'll revisit a few key examples here.

We'll start by getting a baseline time to print out version strings from the JDK and from JRuby.

First, `java -version`, which will mostly be native code:

```
$ time java -version
openjdk version "22.0.2" 2024-07-16
OpenJDK Runtime Environment Zulu22.32+1005-CRaC-BETA (build 22.0.2+9-BETA)
OpenJDK 64-Bit Server VM Zulu22.32+1005-CRaC-BETA (build 22.0.2+9-BETA, mixed mode, sharing)

real	0m0.037s
user	0m0.017s
sys	0m0.012s
```

And then `jruby -v`, which loads only a few classes in JRuby, but is all Java code:

```
$ time jruby -v
jruby 9.4.9.0-SNAPSHOT (3.1.4) 2024-09-11 3de80c34d4 OpenJDK 64-Bit Server VM 22.0.2+9-BETA on 22.0.2+9-BETA +jit [x86_64-linux]

real	0m0.261s
user	0m0.649s
sys	0m0.039s
```

So booting up the rest of OpenJDK and a small part of JRuby adds nearly a quarter-second.

How long does it take to fully boot JRuby and print "hello"?

```
$ time jruby -e 'puts("hello")'
hello

real	0m1.804s
user	0m5.569s
sys	0m0.161s
```

This is a pretty typical result on most local Linux dev systems, but when compared with the standard C implementation of
Ruby, you can see why startup time is our most often-requested improvement:

```
$ time ~/.rubies/ruby-3.3.4/bin/ruby -v
ruby 3.3.4 (2024-07-09 revision be1089c8ec) [x86_64-linux]

real	0m0.014s
user	0m0.005s
sys	0m0.008s

$ time ~/.rubies/ruby-3.3.4/bin/ruby -e 'puts("hello")'
hello

real	0m0.062s
user	0m0.047s
sys	0m0.014s
```

So that's about 30 times faster. But remember we're including the cost of booting OpenJDK and JRuby. How much time is actually spent printing?

```
$ jruby -e 't = Time.now; puts("hello"); puts Time.now - t'
hello
0.002096
```

The JRuby `puts` timing (including the `Time.now` and subtraction calls) varied on my machine from as low as 0.00197 up to 0.0022. It seems that over 98% of the total execution time is just getting the JVM and JRuby ready to run Ruby code.

Finally, here's the full execution time using the two best startup tricks available on standard OpenJDK releases today:

* JRuby's `--dev` flag, which turns off several JRuby and JVM optimizations to boot faster
* A preloaded AppCDS archive generated by the handy gem "jruby-startup"

```
$ gem install jruby-startup
Successfully installed jruby-startup-0.0.6
Parsing documentation for jruby-startup-0.0.6
Done installing documentation for jruby-startup after 0 seconds
1 gem installed

$ generate-appcds 

<lots of output omitted>

*** Success!

$ time jruby --dev -e 'puts("hello")'
hello

real	0m0.834s
user	0m1.439s
sys	0m0.124s
```

We've reduced the startup gap with C Ruby to around thirteen times. It's better, but still noticeably slower, and this effect is magnified when more Ruby code must be loaded to run a command line. We're still not booting up as fast as we'd like, and we've disabled many optimizations in the process.

So, we want to skip all of that overhead and go straight to executing code, but we can't avoid booting up JRuby and we don't want to disable critical optimizations that might help later on. In other words, we want to save a checkpoint immediately after JRuby has finished booting, and restore from that point every time we want to run code.

Checkpoint and restore.

JRuby on CRaC
-------------

Although JRuby compounds the many startup challenges of the JVM, most of what we do at boot can be performed before acquiring a CRaC checkpoint:

* Load JRuby and its many component libraries; this works like any other JVM framework.
* Warm up JRuby's cold parser, compiler, and interpreter.
* Optionally, pre-compile Ruby sources to JVM bytecode, and include that code in the image. When combined with a few warmup "training" cycles, you could see a restored JRuby CRaC image leap directly into optimized Ruby code.

For the rest of this article, we'll walk through a simple example of using CRaC to checkpoint and restore a JRuby command-line operation.

Capturing a checkpoint
----------------------

CRIU and CRaC both have the concept of a "checkpoint". This is the point at which the process (the JVM running JRuby in our case) is frozen in time and saved to disk. The resulting set of files reflects the memory space of that process and metadata about the resources it had acquired and can be "restored" as a new process later on.

The `crac` branch of JRuby contains some modifications to our shell script launcher to support checkpointing:

```
    --checkpoint[=path]     Save a CRaC checkpoint to path, defaulting to ./.jruby.checkpoint
```

A bit of CRaC logging provides some information as the checkpoint capture proceeds:

```
$ jruby --checkpoint
Aug 28, 2024 8:34:52 AM jdk.internal.crac.LoggerContainer info
INFO: Starting checkpoint
Aug 28, 2024 8:34:52 AM jdk.internal.crac.LoggerContainer info
INFO: /home/headius/work/jruby/lib/jruby.jar is recorded as always available on restore
CR: Checkpoint ...
Killed
```

A checkpoint has been captured and the original process killed. Can you guess how we run JRuby from a restored checkpoint?

Restoring a checkpoint
----------------------

My branch also contains modifications for restoring from a checkpoint:

```
    --restore[=path]     Restore a CRaC checkpoint from path, defaulting to ./.jruby.checkpoint
```

Let's give it a try with our print example!

```
$ time jruby --restore -e 'puts("hello")'
hello

real	0m0.116s
user	0m0.172s
sys	0m0.092s
```

Holy toledo! We're now less than twice the startup time of C Ruby, and we didn't even have to use `--dev` mode.

What's actually happening here?

CRaC support in JRuby
---------------------

Support for CRaC in JRuby is a work-in-progress; we'll explore how the pieces fit together in future posts but I provide a high-level overview here. Luckily most of the heavy lifting has been done by the excellent CRaC engineers.

The `--checkpoint` flag adds a JVM flag to enable checkpointing, `-XX:CRaCCheckpointTo=path` and points the JRuby launcher script at a checkpoint-aware "main" class, [CheckpointMain](https://github.com/jruby/jruby/blob/3de80c34d4dc3e43e429e50f2a097f4bf9decbe2/core/src/main/java/org/jruby/main/CheckpointMain.java). CheckpointMain then uses the [CRaC API](https://crac.github.io/openjdk-builds/javadoc/api/java.base/jdk/crac/package-summary.html) and some JRuby-specific code to capture a checkpoint for us. There's three steps involved:

* Run some code before capturing the checkpoint

  This does what you might expect: boot JRuby right up to the point at which we're about to execute code. Code in the base [PrebootMain](https://github.com/jruby/jruby/blob/826fc635d363e88974df00bcbebf0b95d3e13c10/core/src/main/java/org/jruby/main/PrebootMain.java) class runs some simple code through a throw-away JRuby instance to make sure all the critical classes have been loaded, and then boots up the JRuby instance we plan to capture. This can be configured (and I'm open to suggestions on how to make it as clean as possible).

```java
protected String[] warmup(String[] args) {
    Ruby ruby = Ruby.newInstance();
    ruby.evalScriptlet("1 + 1");
    Ruby.clearGlobalRuntime();
    return args;
}
```

* Prepare JRuby for checkpointing

  Because the checkpoint will be restored in a completely new process, there's some bookkeeping necessary to ensure JRuby adapts to the new environment. We tuck away a reference to the pre-booted JRuby runtime before the checkpoint, and fix it up after the restore.

```java
protected Ruby prepareRuntime(RubyInstanceConfig config, String[] args) {
  Ruby ruby = super.prepareRuntime(config, args);
  // If more arguments were provided, run them as normal before checkpointing
  if (args.length > 0) {
    InputStream in   = config.getScriptSource();
    String filename  = config.displayedFileName();
    try {
      if (in == null || config.getShouldCheckSyntax()) {
        // ignore if there's no code to run
      } else {
        // proceed to run the script
        ruby.runFromMain(in, filename);
      }
    } catch (RaiseException rj) {
      handleRaiseException(rj);
    }
  }
  return ruby;
}
```

* Request the checkpoint

  At this point CRaC steps in, performing its own graceful hand-holding for the JVM before requesting a CRIU checkpoint be captured.

```java
@Override
protected void endPreboot(RubyInstanceConfig config, Ruby ruby, String[] args) {
    super.endPreboot(config, ruby, args);
    try {
        Core.getGlobalContext().register(new JRubyContext());
        Core.checkpointRestore();
    } catch (CheckpointException | RestoreException e) {
        e.printStackTrace();
        System.exit(1);
    }
}
```

The `--restore` flag uses the normal JRuby [Main](https://github.com/jruby/jruby/blob/e3de925818ab356374b7c53a5154802e791da141/core/src/main/java/org/jruby/Main.java) class, but when it detects our pre-booted JRuby instance in memory, it uses that rather than start a new one:

```java
final Ruby runtime;
if (PrebootMain.getPrebootMain() != null) {
    // use prebooted runtime, reinitializing config
    runtime = PrebootMain.getPrebootMain().getPrebootRuntime();
    runtime.reinitialize(true);
} else {
    runtime = Ruby.newInstance(config);
}
```

Execution proceeds as normal from this point, except we've just skipped the most expensive part of JRuby's startup!

Digging Deeper
--------------

Hopefully this post has whet your appetite for the incredible potential of JRuby and CRaC! It's early days for the CRaC project, and we're just starting to add support in JRuby, but to me this represents the most promising startup-time improvement we've seen in years.

In Part 2, I'll show more advanced examples, running common command-line tools like `gem` and `bundle` and `rake`, and we'll explore how to pre-boot a Rails instance for faster local development. Future posts will talk through other aspects of using CRaC with JRuby, and I'll show you how to use Docker on Windows and MacOS to create a fast "virtual JRuby" that works like a normal local install.

I will also update this post as links change and new posts are releasesd.

For now, you have the tools you need to experiment with JRuby and CRaC on your own Linux machines. I'd love to hear your feedback... let's get CRaCking!

_JRuby Support and Sponsorship_
-----------------------------

_This is a call to action! JRuby development is funded entirely through your generous sponsorships and the sale of commercial support contracts for JRuby developers and enterprises around the world. If you find my work exciting or believe it is important your company or your projects, please consider partnering with me to keep JRuby strong and moving forward!_

[Sponsor Charles Oliver Nutter on GitHub!](https://github.com/sponsors/headius)

[Sign up for expert JRuby support from Headius Enterprises!](https://www.headius.com/jruby-support)
