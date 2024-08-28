---
layout: post
title: "JRuby Update: August 2024"
date: "2024-08-24"
author: Charles Oliver Nutter
---

Hello friends! I'm reentering the blogspace with an update on JRuby status and some of the exciting projects we have coming up in the latter half of 2024. Let's get started!

A New Path
----------

As I posted on the JRuby blog in July, my work on JRuby at Red Hat came to an end after twelve exciting years. It's hard to feel too bad about so many years of unconditional project sponsorship, for which we are very thankful, but it has also meant new challenges for my work on JRuby and JVM languages.

In the month-and-a-half since then, my wife and business partner Sondra and I have been ramping up our new business entity, Headius Enterprises, and have been signing up JRuby users for our basic "Pro" commercial JRuby support offering. We have a middle-tier "Expert" level coming soon with features like guaranteed bug response times, access to the private Headius Slack, and a quarterly pool of "feature and fix" development hours. This has been and continues to be a learning process; after nearly 20 years of someone else paying for JRuby development, I now need to fund it myself. Luckily we have a sales pipeline in progress and interest from many of the JRuby users out there. Perhaps you're one of them?

I have also been accepting sponsorships from individuals and organizations via my GitHub Sponsors page. Funding OSS entirely through sponsorship donations is challenging, given the real cost of development and self-maintenance, but we are thankful the donations we have received so far. A few individuals making larger regular donations deserve a shout-out here: Shopify, Gautam Rege, Mark Triggs and BEESCM. If you are interested in keeping JRuby going, you should sponsor my work today!

The future of JRuby is not in question (I'll work on it regardless of what happens), but I figure I've got about two years to make this enterprise work. The sooner you can help out, the more likely that I can guarantee funded, full-time JRuby work will continue.

Accelerating JRuby Development
------------------------------

Along with the big funding changes this year comes some major technical moves for JRuby itself.

We're currently working on JRuby 10, the next big leap after our release of JRuby 9000 (9.0.0.0) back in 2015. And boy what a leap it will be:

* Ruby compatibility will be updated from Ruby 3.1 (in the JRuby 9.4 series) all the way up to Ruby 3.4, which will be released at the end of 2024. We're actually pretty solid on compatibility with this version already, but the next few months we'll stabilize that support and prepare to release JRuby 10 soon after the new year.
* The minimum Java version required to run JRuby will be updated to either Java 17 or Java 21. The JRuby 9000 series continues to support Java 8, which was first released a decade ago and has been slowly EOL'ed by Oracle and other companies without paid support. Maintaining support for such an old Java version has meant we miss out on many enhancements to Java, JDK libraries, and JVM features from newer releases. It's time to make a big leap.
* With compatibility caught up and the Java version leap under way, we're also exploring optimizations we have kept on the back burner for years. JRuby has always focused on compatibility first, prioritizing language updates and bug fixes for our users. It's going to be very exciting to see what we can do for JRuby performance with modern JVMs, and I've already started some of that work.

It's also my hope that funding changes for JRuby will give us more of a free hand to hire community members and work directly with users, something that was difficult to do as employees of a large corporate sponsor. It's a very exciting time!

Next-Generation JVM Features
----------------------------

We are also very excited about the array of amazing features being built for the modern JVM. Upcoming blog posts will cover these in detail, but here's a sample of the projects I'm looking at:

Project CRaC - Checkpoint and Restore for the JVM
-------------------------------------------------

JRuby has always been challenged by long startup times, and despite our best efforts it remains a hard problem to solve on existing JVMs. But that may be changing.

Project CRaC (Coordinated Restore at Checkpoint) utilizes a Linux feature called CRIU (Checkpoint and Restore in Userspace) to pre-boot a JVM and create an image of the process for future runs. Once you capture one of these "checkpoints" you can "restore" back to the same point for future commands, leaping directly to the execution of your code.

We'll fully support CRaC in JRuby, and I'll have a series of posts soon showing how JRuby users will benefit.

Project Leyden - Ahead-of-time booting and optimization of JVM applications
---------------------------------------------------------------------------

A similar effort is OpenJDK's Project Leyden, which also seeks to improve startup time. Instead of using a Linux-specific feature like CRaC, however, Leyden uses aggressive ahead-of-time (AOT) booting, profiling, and JIT to help future processses get to optimized code more quickly. Where CRaC will be limited to Linux-like environments, Leyden will be available on all platforms.

We've already started playing with Leyden, and will post more as the project develops.

Project Loom - Virtual threading for the JVM
--------------------------------------------

Project Loom was officially included as part of Java 21 about a year ago, and JRuby already leverages Loom's "virtual threads" to let Ruby fibers scale to thousands of concurrent instances. We're going to continue working with Loom engineers to improve performance of cross-fiber communication, structured concurrency for asynchronous IO frameworks, and reduced memory footprints for small fibers. After startup time, scalable fiber support was the biggest challenge for JRuby. We finally have an excellent solution, and I'll be blogging about our work over the coming weeks.

Project Panama - Foreign function and memory interface for the JVM
------------------------------------------------------------------

JRuby has always pushed the boundaries of the JVM, and to support Ruby better that has meant integrating with native libraries. Because the Ruby C API is so invasive and difficult to support, we needed an alternative way to bind C libraries for use from Ruby code, and rather than building JNI (Java Native Interface) wrappers for every case, we helped popularize the use of FFI (Foreign Function Interface) to programmatically load and bind native functions and manage native memory.

With Project Panama, the JDK is now directly supporting those efforts. Panama brings three features that will improve the performance, scalability, and ease-of-use of JRuby's FFI interfaces:

* Fast, JIT-supported invocation of foreign functions will speed up the many ways we utilize native libraries.
* Native memory-management APIs will make it cheap and efficient to allocate, modify, share, and manage native memory. Along with being able to better interact with native functions, this will provide official support for off-heap data, improving performance in cases that don't need full GC support.
* Code-generation tools like jextract will make it easier to build both Java-based and Ruby-based FFI bindings to native libraries. Eventually, you'll just point your JRuby install at header files, and the binding code will magically appear.

We've already accepted help from OpenJDK developers at Oracle to upgrade our existing FFI backend, JNR (Java Native Runtime) to full Panama support. Expect to see several posts about this work very soon.

Project Babylon - A code and translation model for JVM and beyond
-----------------------------------------------------------------

Project Babylon seeks to finally answer challenges beinging JVM bytecode-based applications and libraries to exotic platforms like GPUs. Babylone introduces the concept of a "code model", which provides a more structured representation of code that can be translated to JVM bytecode or other languages like CUDA and OpenCL.

The potential for Java is obvious: write simple Java code, translated it to your preferred GPU or TPU backend, and make the most of massively-parallel computation. The potential for JRuby is even more exciting: what if you could write numeric algorithms in Ruby and run them directly on the GPU thousands of times faster? That's a real possibility with JRuby and Babylon, and I'm looking forward to making it happen.

Project Valhalla - Value types and reified generics
---------------------------------------------------

When it comes to hard problems on the JVM, Project Valhalla may take the prize. Started way back in 2014, Valhalla seeks to finally bring to the JVM support for stack allocation (via value types) and reified generics (efficient representation of primitive values in an object world). For JRuby, this will mean several huge improvements:

* Reducing the cost of numeric box objects: in Ruby, even numbers are objects, and JRuby allocates an object for most "primitive" numeric values. In the future, we'll be able to use value types to represent those primitives, avoiding the object overhead without affecting Ruby features.
* Cheaper runtime data structures needed for Ruby features: Ruby's blocks can capture variables from their surrounding method, and without JVM support for directly modifying variables on the stack we are forced to allocate a heap object in these cases. Value types will make it easier for us to pass captured state through to block execution, and may eliminate a large source of memory pressure for JRuby appliations.
* More efficient collections of "primitive" values: along with boxed primitives comes the cost of collections of boxed primitives. By being able to reify primitive values into object-based collections, we'll have more efficient representation of primitive values in arrays and hashes.

We are hopeful that Project Valhalla will finally answer one of the most difficult challenges for JRuby: how do you make everything look like an object without the cost of managing all those objects?

The Future of JRuby is Exciting!
--------------------------------

This is just a taste of plans for JRuby in 2024, 2025 and beyond! Development of OpenJDK has never been as rapid or as exciting as it is today, and there's even more projects on the horizon that will make it easier to support non-Java languages on the JVM. Watch this space for my upcoming posts on CRaC, Leyden, Loom, Panama, Babylon, Valhalla and more!