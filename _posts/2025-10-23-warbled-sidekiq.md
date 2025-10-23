---
layout: post
title: "Warbled Sidekiq: Zero-install Executable for JVM"
date: "2025-10-23"
author: Charles Oliver Nutter
published: true

---

In my previous post, I showed [how to use Warbler](https://blog.headius.com/2025/10/packaging-ruby-apps-with-warbler-jar-files.html) to package a simple image-processing tool as an executable jar. This post will demonstrate how to "warble" a larger project: the [Sidekiq background job server](https://sidekiq.org/)!

Warbling Sidekiq
================

Sidekiq is one of the most successful packaged software projects in the Ruby world. It provides high-scale background job processing for Ruby applications atop Redis, and it has been commercially successful via enterprise features and support arrangements. It also happens to work great with JRuby and takes advantage of our excellent parallel threading capabilities.

Seems like a perfect use case for Warbler!

The Easy Way
------------

The easiest way to set up a new warbler configuration for an existing gem (that you may or may not control) is to create your own wrapper project.

I've started that for Sidekiq here: [sidekiq-warbler](https://github.com/headius/sidekiq-warbler).

You'll notice the project is quite slim, containing only a few files for Warbler to package a Sidekiq executable JAR:

* Gemfile and gemspec to specify dependencies. Note that this is not actually pushed as a gem, but Warbler currently auto-detects dependencies using these files.
* config/warble.rb to specify the `sidekiq` executable as the "main" and "sidekiq" as the filename for the JAR. I've removed unused configuration options from the generated warble.rb.
* Rakefile to load in rake tasks for warbler.
* README.md with basic usage information.

Given this, the process is simple:

* `git clone https://github.com/headius/sidekiq-warbler.git`
* `cd sidekiq-warbler`
* `bundle`
* `bundle exec rake jar`

Let's see it in action, using the "plain old Ruby" example from Sidekiq itself:

```text
sidekiq-warbler $ bundle
Fetching gem metadata from https://rubygems.org/............
Resolving dependencies...
Bundle complete! 2 Gemfile dependencies, 15 gems now installed.
Use `bundle info [gemname]` to see where a bundled gem is installed.
sidekiq-warbler $ bundle exec rake jar
rm -f sidekiq.jar
Creating sidekiq.jar
sidekiq-warbler $ java -jar sidekiq.jar -r ../sidekiq/examples/por.rb


               m,
               `$b
          .ss,  $$:         .,d$
          `$$P,d$P'    .,md$P"'
           ,$$$$$b/md$$$P^'
         .d$$$$$$/$$$P'
         $$^' `"/$$$'       ____  _     _      _    _
         $:    ',$$:       / ___|(_) __| | ___| | _(_) __ _
         `b     :$$        \___ \| |/ _` |/ _ \ |/ / |/ _` |
                $$:         ___) | | (_| |  __/   <| | (_| |
                $$         |____/|_|\__,_|\___|_|\_\_|\__, |
              .d$$                                       |_|
      

INFO  2025-10-23T15:37:27.684Z pid=85096 tid=1ryk: Running in jruby 10.0.2.0 (3.4.2) 2025-08-07 cba6031bd0 OpenJDK 64-Bit Server VM 21.0.8+9-LTS on 21.0.8+9-LTS +indy +jit [arm64-darwin]
INFO  2025-10-23T15:37:27.684Z pid=85096 tid=1ryk: See LICENSE and the LGPL-3.0 for licensing details.
INFO  2025-10-23T15:37:27.684Z pid=85096 tid=1ryk: Upgrade to Sidekiq Pro for more features and support: https://sidekiq.org
INFO  2025-10-23T15:37:27.686Z pid=85096 tid=1ryk: Sidekiq 8.0.8 connecting to Redis with options {size: 10, pool_name: "internal", url: nil}
INFO  2025-10-23T15:37:27.707Z pid=85096 tid=1ryk: Sidekiq 8.0.8 connecting to Redis with options {size: 5, pool_name: "default", url: nil}
INFO  2025-10-23T15:37:27.710Z pid=85096 tid=1ryk: Starting processing, hit Ctrl-C to stop
```

It's that easy!

Adding Official Warbler Support to Sidekiq
------------------------------------------

In cases where you own or control a given gem, you can also add support for Warbler directly to the project. I've done that on a branch of Sidekiq here: https://github.com/headius/sidekiq/tree/warbled

You'll notice a few differences in the "official support" `config/warble.rb`:

```ruby
# Hack to disable the "war" trait in Warbler
class Warbler::Traits::War
  def self.detect? = false
end
class Warbler::Traits::Rack
  def self.detect? = false
end

# Warbler web application assembly configuration file
Warbler::Config.new do |config|
  config.executable = "bin/sidekiq"
end
```

* The "hack" here disables Warbler's automatic detection of web applications, so that we can just produce a plain executable JAR file instead of a web application WAR file. I've filed an issue with Warbler to make this easier in the future: https://github.com/jruby/warbler/issues/587
* The JAR file name is detected from the gem name ("sidekiq") and we're using a "main" script from the same gem, so the config is a bit simpler.

I've also tweaked Sidekiq's `Gemfile` to pin Rails at 7.1 (we're in the process of shipping 7.2 and 8+ support for JRuby) and disable some native CRuby extensions we don't support (`debug`, `vernier`, `ruby-prof`).

The result is basically the same:

```text
sidekiq $ bundle exec rake jar
rm -f sidekiq.jar
Creating sidekiq.jar
sidekiq $ java -jar sidekiq.jar 
INFO  2025-10-23T16:01:50.836Z pid=86190 tid=1xca: ==================================================================
INFO  2025-10-23T16:01:50.837Z pid=86190 tid=1xca:   Please point Sidekiq to a Rails application or a Ruby file  
INFO  2025-10-23T16:01:50.837Z pid=86190 tid=1xca:   to load your job classes with -r [DIR|FILE].
INFO  2025-10-23T16:01:50.837Z pid=86190 tid=1xca: ==================================================================
INFO  2025-10-23T16:01:50.837Z pid=86190 tid=1xca: sidekiq [options]
    -c, --concurrency INT            processor threads to use
    -e, --environment ENV            Application environment
    -g, --tag TAG                    Process tag for procline
    -q, --queue QUEUE[,WEIGHT]       Queues to process with optional weights
    -r, --require [PATH|DIR]         Location of Rails application with jobs or file to require
    -t, --timeout NUM                Shutdown timeout
    -v, --verbose                    Print more verbose output
    -C, --config PATH                path to YAML config file
    -V, --version                    Print version and exit
    -h, --help                       Show help
```

IPv4 vs IPv6
------------

If you're on a platform that supports IPv6 you may notice that JRuby (the JDK, really) will try to use IPv6 addresses and connections by default. On my system, the Redis server only bound itself to IPv4, preventing Sidekiq from being able to make that connection.

The magic flag to force the JDK to use IPv4 is `-Djava.net.preferIPv4Stack=true`. You can pass that directly to the `java` command, or use environment variable `JDK_JAVA_OPTIONS` as I do here:

```text
sidekiq $ export JDK_JAVA_OPTIONS="-Djava.net.preferIPv4Stack=true"
sidekiq $ java -jar sidekiq.jar -r ./examples/por.rb
NOTE: Picked up JDK_JAVA_OPTIONS: -Djava.net.preferIPv4Stack=true


               m,
               `$b
          .ss,  $$:         .,d$
          `$$P,d$P'    .,md$P"'
           ,$$$$$b/md$$$P^'
         .d$$$$$$/$$$P'
         $$^' `"/$$$'       ____  _     _      _    _
         $:    ',$$:       / ___|(_) __| | ___| | _(_) __ _
         `b     :$$        \___ \| |/ _` |/ _ \ |/ / |/ _` |
                $$:         ___) | | (_| |  __/   <| | (_| |
                $$         |____/|_|\__,_|\___|_|\_\_|\__, |
              .d$$                                       |_|
      
...
```

Why Warble?
===========

We've seen that it's pretty easy to turn a larger app like Sidekiq into an executable JAR file, but what's the real advantage here?

Some of that comes from running on JRuby, which you can get with or without Warbler (Sidekiq works well and is regularly tested on JRuby):

* Better parallel scaling across cores, for both Sidekiq itself and any garbage produced along the way (JVM's GCs are highly concurrent and super scalable).
* Better tooling based on JVM profiling and monitoring features.

And the other features are specific to a "warbled" executable JAR file:

* "Zero-install" other than needing a JDK on the host system.
* No native libraries, no build tools, no additional dependencies.
* Enterprise-friendly: they don't have to know or care about Ruby to deploy your product.
* IP-safe: Warbler can precompile Ruby code into JRuby's bytecode format, making it very difficult to steal.

Basically, if you want to ship Ruby tools and applications for modern organizations, JRuby and Warbler are a great way to get there!

Your Turn
=========

Obviously there's enormous potential for Ruby tools and applications to be packaged and shipped using Warbler. If you're interested in making this happen for you project, here's the steps to take:

* Make sure it runs on JRuby! Pure-Ruby libraries should "just work", but more complicated apps may require alternative gems.
* Decide what your command-line "main" should look like when run as an executable jar. This may simply be your existing bin script.
* Follow docs on the [Warbler project](https://github.com/jruby/warbler) for configuring and warbling your app.
* Profit! Enterprises love simple executable tools!

JRuby is clearly the future of Ruby in the enterprise, and Warbler is a huge part of that. I am always here to help you build and package Ruby applications with JRuby and Warbler. Follow the links below and let's set up a chat or call to help you bring your apps to a wider world!

## [Join the discussion on r/ruby!](https://www.reddit.com/r/ruby/comments/1ocje8h/packaging_ruby_apps_with_warbler_executable_jar/)

_JRuby Support for Your Project_
================================

_This is a call to action!_
---------------------------

JRuby development is made possible by our primary sponsor, [Headius Enterprises](https://headius.com), offering a range of professional development and support resources for your team. You can choose JRuby for your next project knowing that you've got the world's best JRuby experts standing by to help.

If you are interested in **scaling Ruby** to new heights, deploying your applications in **large enterprises**, or taking advantage of JVM features like **world-class JIT compilers**, **battle-tested libraries**, and **leading-edge AI tools**, you need to be using JRuby. Let us help you.

[Sign up for expert JRuby support from Headius Enterprises!](https://www.headius.com/jruby-support)

If you find this article or my work on JRuby to be interesting and useful, you can also sponsor me directly on GitHub. Your contributions help ensure the JRuby project keeps moving forward!

[Sponsor Charles Oliver Nutter on GitHub!](https://github.com/sponsors/headius)
