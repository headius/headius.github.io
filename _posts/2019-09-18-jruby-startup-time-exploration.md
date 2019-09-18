---
layout: post
title: "Start It Up: Improving JRuby's Startup Time"
date: '2019-09-18'
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
line, be they installing libraries, generating models, or starting up interactive consoles and application
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
RubyGems subsystem. But JRuby's taking 100x longer to do it!

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

![JRUBY_OPTS --dev flag](/images/jruby_opts_dev_flag.png)

Let's move on to some examples that actually do some work. I promise things get better from here!

Less contrived examples
-----------------------

Of course most Rubyists won't stop at evaluating the number `1`. They're probably going to run some actual Ruby
commands, Let's look at a common one: installing a gem.

These numbers compare CRuby and JRuby installing a single gem with no dependencies from a local gem file. I'm
using Rake for this example.

![gem install rake from local file](/images/gem_install_rake.png)

Ok, now we can see that our ratio has improved from 100x slower to a mere 5x slower. We're now giving the JVM
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