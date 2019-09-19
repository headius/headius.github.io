---
layout: post
title: "Start It Up: Improving JRuby's Startup Time"
date: "2019-09-18 16:09"
author: Charles Oliver Nutter
---

Hello friends! How long has it been? Too long! This post marks the beginning of a return to blogging about JRuby, the JVM, and everything in between. Today we'll cover a topic that never gets old: JRuby's startup performance.

Startup time is important!
--------------------------

Before we get started running numbers and tweaking flags, you may be wondering why startup time is such a big
deal. Most applications are deployed pretty rarely. right? Even the most aggressive organizations usually won't
redeploy more than once a day, right?

While this is usually true (ignoring [continuous deployment](https://www.agilealliance.org/glossary/continuous-deployment) for now),
the actual development experience of different languages varies greatly. Java, for example, managed to dodge the
startup time issue for years because most developers built their apps using an always-on IDE that transparently
compiled code in the background and often restarted or redeployed to local development servers automatically. Only
recently with the rise of small services and cloud "functions" has the issue of Java startup time really become
a concern again.

In the Ruby world, things are very different. Most non-coding development tasks involve running Ruby at a command
line, be they installing libraries, generating code, or starting up interactive consoles and application
servers. Rubyists do the great majority of their work using two tools: an editor, and a terminal. And because of
this, JRuby needs to have acceptable startup time.

Why is this hard?
-----------------

JRuby is an implementation of Ruby that runs on the JVM, and as you'd expect a large part of our codebase starts
running as JVM bytecode. Most JVMs initially execute that bytecode using an interpreter, similar to how CRuby
executes its "instruction sequences". As that code gets "hot", the JVM will usually compile it to native
instructions that the system CPU can execute directly. The idea is that since the interpreter can start executing
immediately, we save some startup time by not running an expensive JIT compiler before execution.

This works...sort of. Modern JVMs do start executing very quickly, but if your application needs to run lots of
code at boot, before it can do useful work, startup time will reflect the fact that the bytecode interpreter
is much slower than native code.

Anyone familiar with deploying JVM-based applications will know this as the dreaded "warmup curve". It hits
JRuby particularly hard.

Compare JRuby with CRuby at startup. Ruby apps are distributed as source code, which means they must first
first pass through a parser and compiler before being executed with an interpreter -- all of which start out as
"cold" JVM bytecode in JRuby. Contrast this to CRuby, where all of these stages
are written in C and precompiled to native code long before the runtime is launched. Both JRuby and CRuby
run lots of Ruby code on every startup, mostly to boot up the RubyGems subsystem

Making matters worse, all of the logic to set up the core classes (defining String or Array and their
methods, for example) as well as the actual core method implementations *also* start out "cold". Everything
running during the first few seconds after launching JRuby is running in the JVM interpreter or the JRuby
interpreter...which itself is also running in the JVM interpreter!

How bad is it?
--------------

Let's take a look at some worst-case numbers.

This first example compares `ruby -e 1` times on CRuby 2.6.4 versus JRuby 9.2.9 (from master) on OpenJDK 8. 

![ruby -e 1 startup times](/images/ruby_dash_e.png)

Yikes! Both implementations are doing roughly the same work here: setting up the core classes and booting the
RubyGems subsystem. But JRuby's taking 16x longer to do it!

I say this is a worst case because it really gives us no time at all to optimize any code. JRuby doesn't try to
JIT anything until *after* baseline startup. The JVM does optimize some of our code, but it's too little, too
late.

Optimizing for development
--------------------------

In fact, the JVM is actually a little too aggressive, spending many CPU cycles during this 1.6 seconds optimizing
and emitting code that will only be used briefly. We pay a large cost at startup in trade for reducing
longer-term warmup times.

We can actually tweak OpenJDK to be less aggressive by forcing it to only use the simplest part of its JIT,
rather than working hard to create optimized native code we won't use.

We do this by forcing the Hotspot "tiered" compiler to only use its first tier by passing
`-XX:TieredStopAtLevel=1` to the JVM.

In addition, we know JRuby's compiler won't help us much during these first few seconds, so we can turn that
off too using the `-X-C` JRuby flag.

Finally, we also turn off the JVM's *bytecode verification* since all the bytecode we'll run has been verified
to death in JRuby's continuous integration server. We do this by passing `-Xverify:none` to the JVM. 

![jruby --dev -e 1](/images/jruby_dev_flag.png)

As you can see here, this combination of flags trims a good bit of time off startup. As a service to JRuby users
everywhere, we include these flags (and a few others) as the `--dev` flag. This is your first,
best startup time tip!

![jruby --dev -e 1 comparison](/images/jruby_dev_flag_chart.png)

We recommend JRuby users add this flag to the `JRUBY_OPTS` environment variable, so all JRuby processes and
subprocesses see it.

(Edit: Keep in mind that this flag turns off a number of optimizations, so don't try to benchmark any code with
it enabled. Note below that I have dedicated part of my bash prompt to showing JRUBY_OPTS so I don't forget.)

![JRUBY_OPTS --dev flag](/images/jruby_opts_dev_flag.png)

Let's move on to some examples that actually do some work. I promise things get better from here!

Less contrived examples
-----------------------

Of course most Rubyists won't stop at evaluating the number `1`. They're probably going to run some actual Ruby
commands, Let's look at a common one: installing a gem.

These numbers compare CRuby and JRuby installing a single gem with no dependencies from a local gem file. I'm
using Rake for this example.

![gem install rake from local file](/images/gem_install_rake.png)

Ok, now we can see that our ratio has improved from 16x slower to a mere 5x slower. We're now giving the JVM
a chance to "warm up" and optimize JRuby itself (but not too much!) so the numbers improve.

Here's a comparison of the `gem list` command, listing all the gems I have installed locally (about 640 of them).

![gem list with 641 gems installed](/images/gem_list.png)

Even better: our slowdown is only 3x compared to CRuby!

Finally, let's measure the cost of starting up a Rails interactive console. This example is actually aggravated
by a Rails design choice: in order to guarantee only the required libraries get loaded, most Rails commands
you run will spawn a subprocess to actually execute. So we pay twice the cost and startup time becomes an even
bigger challenge.

For this example, I'm piping the function call `exit` to the console so it terminates immediately after it starts
up.

![rails console startup and exit](/images/rails_console.png)

Here, finally, we have an example where JRuby's "best" startup time is less than 2x slower than CRuby. We are
closing the gap!

We have some baseline numbers now, based on the `-e 1`, `gem list`, `gem install`, and `rails console` command
lines. Let's explore a few other scenarios for speeding up these commands.

Ahead-of-time compilation
-------------------------

The new hotness in the JVM world is the re-emergence of *ahead-of-time* compilation (AOT), which precompiles all
your application's JVM bytecode to native code before you ever run it. The most recent and arguably most
successful example of this comes from GraalVM, an enhanced OpenJDK-based runtime that examines and optimizes
your whole application at once (so-called "closed-world" optimization) to produce small, fast, efficient native
binaries.

Sounds like just the magic we need, right? For some cases, it may be! An alternative Ruby implementation called
TruffleRuby -- part of the same GraalVM project -- uses AOT and a prebooted image of the heap to improve their
baseline startup substantially.

![rails -e 1 truffleruby comparison](/images/ruby_dash_e_truffleruby.png)

Wow! Not only has this eliminated the startup gap, it's actually *better* startup time than CRuby.

There's a number of techniques at play here in addition to AOT, which you can read about in Benoit Daloze's blog post
[How TruffleRuby's Startup Became Faster than MRI's](https://eregon.me/blog/2019/04/24/how-truffleruby-startup-became-faster-than-mri.html).
There's a lot of clever, exciting work going on there.

So how does TruffleRuby fare on running our three common Ruby commands above? Things get a little murky here.

![common ruby commands truffleruby comparison](/images/ruby_commands_truffleruby.png)

Unfortunately, since TruffleRuby still parses, compiles, and executes Ruby code from source, they still see poor
startup time in comparison to CRuby. I'm sure they're continuing to work on this, so watch this space.

Ahead-of-time compilation may still be an option for JRuby, however. Our interpreter is much simpler than the one
found in TruffleRuby, so precompiling JRuby to native code should run pretty well. And since JRuby already can
precompile Ruby code to JVM bytecode, it's possible we could skip right past the parse, compile, and JIT phases. We
hope to explore this option in the next few months.

Alternative JVMs: Eclipse OpenJ9
--------------------------------

One of the most exciting developments of the past year was the official open source release of IBM's J9 JVM as OpenJ9.
J9 is one of the few world-class, fully-compliant JVM implementations out there, with a completely different array
of optimizations, garbage collectors, and supported platforms. Having OpenJ9 available gives JRubyists another way
to run, scale, and deploy Ruby applications.

One of the cooler features of OpenJ9 is its ability to share pre-processed class data across runs. When you pass the
`-Xshareclasses` flag, OpenJ9 will create a shared archive containing pre-parsed, pre-verified JVM bytecode and class
data. In addition, it will dynamically save native code output from the JIT, allowing those methods to start up a bit
"hotter" and skipping the interpreter and some optimization stages.

An additional flag `-Xquickstart` reduces how much optimization OpenJ9 does (similar to the Hotspot `TieredStopAtLevel`
flag shown above) to allow short-running commands to get up and going more quickly.

And as of JRuby 9.2.9, we include these flags in our `--dev` mode!

![common ruby commands openj9 comparison](/images/ruby_commands_openj9.png)

Now we're talking! By combining the "quickstart" and "shareclasses" flags, we've improved on the Hotspot startup
times in two out of three cases. The third, `rails console` is oddly slower...we look forward to working with the
OpenJ9 team to get that one optimized as well.

If you're on an unusual platform (AIX? PowerPC?) or just want to try something new, you should definitely start
playing with JRuby on OpenJ9. Expect to see JRuby make better use of OpenJ9's unique features very soon.

Bleeding edge: OpenJDK 13
------------------------

The last example I want to show is from the just-released OpenJDK 13, which brings to the table improved support for
what they call "class data sharing" (CDS).

The CDS feature started out as a paid option from Sun Microsystems and later Oracle, but as of OpenJDK 10 it is both
free and Free for all uses. Since that time, it has been improved to cache more data, more efficiently, and most recently it is
now possible to generate the CDS "archive" dynamically based on a given run of the JVM.

We can use the new `-XX:ArchiveClassesAtExit=filename.jsa` flag to produce one of these archives, and
`-XX:SharedArchiveFile=filename.jsa` flag to use it at runtime.

![jdk13 cds command line](/images/cds_command_line.png)

You can specify different files for different commands, of course, to have fine grained control over what's getting
cached for you. With our simple example, we see some very nice improvements:

![common ruby commands on JDK13 CDS](/images/ruby_commands_cds13.png)

Another win over plain `--dev` on OpenJDK 8's version of Hotspot! Both of the gem commands are the fastest ever, and
the `-e 1` time has dropped to a merge 10x CRuby's time. Frustratingly, the `rails console` is again slower than on
Hotspot 8...what is it about Rails that continues to confound optimizing VMs?

We will be exploring how best to take advantage of these improvements. Currently, JRuby will automatically use the
CDS archive in `lib/jruby.jsa` if it exists, and if you're not using JDK 13 we provide the [jruby-startup](https://github.com/jruby/jruby-startup)
gem with its `generate-appcds` command to regenerate this archive for you. Give it a try and let us know how it works
for you!

The bottom line
---------------

Startup time is crucial to the development process for most Rubyists, and we continue to improve how JRuby boots and
executes to reduce startup time. Meanwhile, there's an army of VM engineers bringing startup optimizations to OpenJDK,
GraalVM, and OpenJ9...that you as a JRubyist will benefit from. Work continues, but the future looks bright!

Here's a short summary of what we've learned today:

* JRuby provides a `--dev` flag to reduce startup time by reducing how much optimization happens.
* GraalVM may provide a future path toward pre-compiling JRuby and key Ruby libraries to improve startup.
* OpenJ9 includes [`-Xquickstart` and `-Xshareclasses`](https://developer.ibm.com/articles/optimize-jvm-startup-with-eclipse-openjj9/)
  flags today to help commands start up more quickly.
* Hotspot's Class Data Sharing [continues to improve](https://bugs.openjdk.java.net/browse/JDK-8221706) and already
  provides the best JRuby startup for many commands.

You can try out JRuby 9.2.9 from our [nightly builds](https://oss.sonatype.org/content/repositories/snapshots/org/jruby/jruby-dist/9.2.9.0-SNAPSHOT/)
if you'd want the fastest-starting version of JRuby yet. We'll be putting out a formal release within the next
couple weeks.

Have fun!

[Discuss this post on Reddit](https://www.reddit.com/r/ruby/comments/d6444b/start_it_up_improving_jrubys_startup_time/)

(Edit: An earlier version of this pots stated that Class Data Sharing was made free/Free in OpenJDK 8. It has been amended
to indicate this happened in OpenJDK 10.)