---
layout: post
title: "Updating Deprecations with Version Information"
date: "2025-10-10"
author: Charles Oliver Nutter
published: true

---

Java 9 added the ability to mark a `@Deprecated` annotation with a "since" version, so we figured it was worth updating JRuby.

Deprecation in Java
===================

Deprecation is the process of marking a feature as "no longer supported" or "on its way out", usually in a programmatic way so users can see warnings at build time. Nearly all languages and ecosystems have some way to mark features, APIs, or libraries as deprecated.

In Java 1.4 and earlier, this was done via the `@deprecated` directive (note lower-case "d") in JavaDoc, which was great for documentation users but required extra processing by source-level tools and was not visible to any tools at runtime.

Java 1.5 introduced a new annotation `java.lang.Deprecated` that is retained at runtime and requires only annotation-awareness to process. It has become a generally preferred way to mark classes, methods, and fields as deprecated. Unfortunately, it provided no additional information, like a date or version after which the feature should not be used, or whether it would eventually be removed.

Java 9 fixed that, by adding two attributes to the `@Deprecated` annotation: `since`, indicating when the deprecation first went into effect and `forRemoval`, indicating that the feature has been scheduled for removal. The `since` attribute is only informational, but a setting of `forRemoval=true` tells the compiler to complain about uses even more loudly (and perhaps fail the build).

Deprecation in JRuby
====================

The modern era of JRuby extends back almost to the release of Java 1.5, so we've steadily added `@Deprecated` annotations throughout our codebase as we move away from old features and peculiarly-named methods. These deprecations are generally not visible to the average JRuby user, but if you use JRuby as an API or embed it into a larger Java application, you'll get compile-time warnings for deprecated features.

With the release of JRuby 10, we now depend on a minimum of Java 21... which means we can also use the `since` tag. But what of all the existing deprecations?

Updating deprecations with "since"
----------------------------------

This week, we decided it would be worth rewriting all "bare" `@Deprecated` annotations to include `since` information. The process required a few steps:

* Get the source locations of all `@Deprecated` annotations that did not contain any attributes.
* For each location, determine the first JRuby release in which the deprecation was shipped.
* Update the deprecations to point at that version.

With a little Ruby and Git, here's the script I came up with:

```ruby
# scan all .java files in the dir specified by ARGV[0]
Dir[ARGV[0] + "/**/*.java"].each do |file|
  # read all lines from the file
  file_lines = File.readlines(file)
  
  file_lines.each_with_index do |line, i|
    line.gsub!(/@Deprecated\S*$/) do
      # use git blame to find the commit that introduced the annotation
      sha = `git blame -L #{i+1},#{i+1} #{file}`.split(" ")[0]
      
      # use git describe to get the first tag that contains that commit
      tag = `git describe --abbrev=0 --tags --contains #{sha}`.split("~")[0]
      
      # add that tag to a "since" attribute for the annotation
      "@Deprecated(since = \"#{tag}\")"
    end
  end
  
  File.write(file, file_lines.join(''))
  puts "processed #{file}"
end
```

Ups and downs
-------------

The PR I created is here: https://github.com/jruby/jruby/pull/9027

On the up side, this clearly indicates when each deprecation was added, so we can start to remove the oldest once (for some definition of "oldest"). It also gives better output to users so they know how long the feature they're using has been deprecated.

On the down side, I had to manually fix several tags that were bogus or inaccurate (which maybe is a plus, really), and a piece of code was ever moved then the history is likely broken and the "since" version might be inaccurate. A improved version might also try to track file moves, but that was more than I needed here.

There's also a small concern that we've basically just updated all of those `@Deprecated` lines, so any future blaming will have to dig deeper to find out when and why they were deprecated. But I kept this change as a single commit, so at worst you'll have to `git blame` the parent commit in such cases.

What do you think?
------------------

Before I merged this, I asked if anyone could think of a good reason not to do it... and I got no responses. What do you think of this rewriting of `@Deprecated` annotations to include the version?

If you decide to proceed with a similar transformation, I hope this script is helpful! Remember you can always just run it with JRuby, and the easiest way to run JRuby for a simple script would be to use [jbang](https://www.jbang.dev/):

```text
jruby $ jbang run@jruby deprecated_since.rb core/src/main/java
processed core/src/main/java/org/jruby/AbstractRubyMethod.java
processed core/src/main/java/org/jruby/Appendable.java
processed core/src/main/java/org/jruby/BasicObjectStub.java
processed core/src/main/java/org/jruby/DelegatedModule.java
processed core/src/main/java/org/jruby/EvalType.java
...
```

Enjoy!

## [Join the discussion on r/ruby!](https://www.reddit.com/r/ruby/comments/1o3eiid/updating_jrubys_deprecations_with_since_version/)

## [Join the discussion on r/java!](https://www.reddit.com/r/java/comments/1o3eku0/updating_historical_deprecations_with_since/)

_JRuby Support for Your Project_
================================

_This is a call to action!_
---------------------------

JRuby development is made possible by our primary sponsor, [Headius Enterprises](https://headius.com), offering a range of professional development and support resources for your team. You can choose JRuby for your next project knowing that you've got the world's best JRuby experts standing by to help.

If you are interested in **scaling Ruby** to new heights, deploying your applications in **large enterprises**, or taking advantage of JVM features like **world-class JIT compilers**, **battle-tested libraries**, and **leading-edge AI tools**, you need to be using JRuby. Let us help you.

[Sign up for expert JRuby support from Headius Enterprises!](https://www.headius.com/jruby-support)

If you find this article or my work on JRuby to be interesting and useful, you can also sponsor me directly on GitHub. Your contributions help ensure the JRuby project keeps moving forward!

[Sponsor Charles Oliver Nutter on GitHub!](https://github.com/sponsors/headius)
