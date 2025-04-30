---
layout: post
title: "Creating Beautiful Charts with JRuby and JFreeChart"
date: "2025-04-30"
author: Charles Oliver Nutter
published: true
---

I recently returned from [RubyKaigi](https://rubykaigi.org/2025/) where I had the opportunity to sit down with members of the Japanese Ruby community and show them a little bit of JRuby. One of the items that came up a few times was the difficulty of utilizing external libraries from Ruby: if it's a C library, typically you have to either write a C extension or do the extra work of writing up an FFI binding.

If the library is not implemented in C or Ruby, things get even weirder.

One example is the [Charty](https://github.com/red-data-tools/charty) library, one of the more popular options for generating beautiful chart graphics on Ruby. But Charty actually wraps a _Python_ library called [matplotlib](https://matplotlib.org/), and the bindings for CRuby literally load Python into the current process and call it via Ruby C API calls which then make Python C API calls. The horror!

Also recently, a [Reddit Ruby post](https://www.reddit.com/r/ruby/comments/1kalb7s/rails_action_mailer_rendering_charts_or_graphs_in/) demonstrated the use of the QuickChart library, which is implemented in JavaScript... yuck! In this case, the call-out is via process launching to the QuickChart command line, but either way it's not at really integrated into the Ruby app using it.

Is there a better way? You bet there is: use a JVM-based library from JRuby! This post will show you how to get started with JRuby's Java integration and a few quick examples of using the JFreeChart library from Ruby code. I promise you won't have to write a single line of Java, C, Python, or JavaScript.

JFreeChart
==========

The [JFreeChart library](https://www.jfree.org/jfreechart/) provides an extensive set of chart formats, with a huge array of rendering options and easy integration with either JVM-based GUI toolkits or simple image-file output. Unlike many of the other charting libraries, it also supports live updating of a given chart GUI, making it an excellent choice for desktop monitoring tools.

Building a GUI with JRuby is fun and easy, but for this post we'll focus on just the chart and image generation parts of the API and how you can use them easily from Ruby.

JRuby's Java Integration
========================

The magic starts with JRuby's [Java integration layer](https://github.com/jruby/jruby/wiki/CallingJavaFromJRuby). The simplest way to use a Java library is to basically pretend it's a Ruby library, with a few tweaks along the way:

* Download the library's jar file and dependencies, or use jar-dependencies (part of JRuby's standard library) to fetch them for you.
* Require the jar file and dependencies manually (a simple `require "myfile.jar"` works in JRuby!) or by using `require_jar` from `jar-dependencies`.
* Call the Java classes you're interested from Ruby as if they were plain old Ruby classes. You can even import them, Java-style, but it's often not necessary.

Let's start with a simple example.

Calling into the java.lang.Runtime class
----------------------------------------

Assuming you've installed [JRuby 10](https://www.jruby.org/2025/04/14/jruby-10-0-0-0.html), you can start playing with JVM libraries directly from IRB. Let's fire it up and make sure we're running on JRuby.

```text
$ irb
irb(main):001> RUBY_ENGINE
=> "jruby"
irb(main):002>
```

The JDK comes with a large number of built-in features, of course, and I like to use the [java.lang.Runtime](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Runtime.html) class for quick demonstrations. It provides methods to query a bit of information about the running JVM and the host operating system. Let's "import" it into our session and call a few methods.

```text
irb(main):002> rt = java.lang.Runtime.runtime
=> #<Java::JavaLang::Runtime:0x39a87e72>
irb(main):003> rt.available_processors
=> 8
irb(main):004> rt.free_memory
=> 222145360
```

There's a few bits of magic here that make it really feel like Ruby:

* JRuby defines a few top-level methods for common Java package roots like `java`, `javax`, `org`, and `com`. We use this to reference the Java package `java.lang` here and access the `Runtime` class.
* `Runtime` defines a static method (similar to a Ruby class or singleton method) called `getRuntime` which gets the singleton instance of `Runtime` for the current JVM process. JRuby turns Java "getter" and "setter" methods into Ruby "attribute accessors", so you can just call `Runtime.runtime` here.
* The Ruby calls to `available_processors` and `free_memory` translate to the Java methods `getAvailableProcessors` and `getFreeMemory` in the same way. The original methods are still callable, but we add aliases for the Ruby-friendly names.

We can accomplish a tremendous amount with the libraries shipped in the JDK, ranging from simple database access to professional desktop GUI development.

If the libraries we want to use are not shipped with the JDK, however, we need to fetch them. It's always possible to download the library's "jar" file directly, but then you may also have to hunt down its dependency jars. A better way is to use JRuby's [jar-dependencies](https://github.com/jruby/jar-dependencies) library.

jar-dependencies
----------------

Like Ruby gems get pushed to [rubygems.org](https://rubygems.org/), Java libraries get published as jar files to [Maven](https://maven.apache.org/) repositories, a global, federated repository of every version of every Java library known to man. In order to make it as easy as possible to use these libraries from JRuby, we've built extensive Maven tooling for both Ruby and Java applications.

The simplest way to load a JVM library from Maven into your JRuby app is to use `jar-dependencies`, a built-in tool for fetching and managing Java dependencies in either standalone applications or in gems.

For our examples, we want to fetch JFreeChart and its dependency libraries. If we search for it at [search.maven.org](https://search.maven.org), we can acquire its "Maven coordinates".

![Searching Maven Central for JFreeChart](/images/maven_search_jfreechart.png)

The most recently-published artifact is jfreechart 1.5.5. From here we can see the coordinates we're looking for:

* The group ID, used for namespacing libraries, is `org.jfree`.
* The artifact ID, which identifies the library, is `jfreechart`.
* And the version we want is `1.5.5` (Maven strongly encourages always using specific versions).

Given that, we can set up our JFreeChart project.

JRuby and JFreeChart: Happy Together!
=====================================

Without knowing anything about JFreeChart, I was able to load it up and get some simple examples working from IRB just last night. I was so impressed, I decided to write this blog post!

I've pushed this example as [headius/jruby-charts](https://github.com/headius/jruby-charts) on GitHub so you can follow along.

Using jar-dependencies to fetch JFreeChart
------------------------------------------

We'll start by creating a `jar-dependencies` "Jarfile", which is roughly equivalent to Bundler's "Gemfile":

```ruby
jar 'org.jfree:jfreechart:1.5.5'
```

Similar to a Gemfile's gem name and version, we specify the Maven coordinates of the library we want (separated by colons). Once we have this file, we can fetch and "lock" this dependency with the `lock_jars` command.

```text
jruby-charts $ lock_jars

-- jar root dependencies --

      org.jfree:jfreechart:1.5.5:compile

Jars.lock created
```

This command fetches JFreeChart and any other dependencies to your local Maven repository, which by default is in `~/.m2/repository`. In general, we recommend using jar-dependencies this way, so that your jars don't conflict with other gems' jars, and all applications on a given system use the same set of downloaded files. It's also possible to fetch the jars and ship them inside your application or gem, but we'll leave that example for another day.

Let's get to the fun stuff: using JFreeChart from Ruby!

Generating a simple bar chart
-------------------------

Our first example will create a simple [bar chart](https://github.com/headius/jruby-charts/blob/master/examples/barchart.rb).

First, we need to load in the jars we just downloaded and locked.

```ruby
# Use jar-dependencies, included with JRuby, to load JFreeChart
require 'jar-dependencies'
require_jar 'org.jfree', 'jfreechart', '1.5.5'
```

At this point [all of the classes of JFreeChart](https://www.jfree.org/jfreechart/javadoc/index.html) are available to Ruby. We start by creating a [DefaultCategoryDataset](https://www.jfree.org/jfreechart/javadoc/org/jfree/data/category/DefaultCategoryDataset.html) which will hold our bar chart data.

```ruby
# Create an empty CategoryDataSet
bar_data = org.jfree.data.category.DefaultCategoryDataset.new
```

JFreeChart provides a wide array of dataset types that can source data from a database, a JVM-based collection object (which includes Ruby collections), or other forms of structured data. The "default" version works nicely for a simple example. Let's fill it with some data.

```ruby
# Add values to the dataset
bar_data.add_value 44, "Ben and Jerry's", "Flavors"
bar_data.add_value 31, "Baskin Robbins", "Flavors"
bar_data.add_value 11, "Cold Stone", "Flavors"
```

This bar chart will display a count of ice cream flavors from three well-known purveyors of the creamery arts. The `add_value` method here is `addValue` in Java, and takes a number, a column key, and a row key.

Given our dataset, we can now request that JFreeChart create a basic bar chart for us using the [ChartFactory](https://www.jfree.org/jfreechart/javadoc/org/jfree/chart/ChartFactory.html) class.

```ruby
# Create a bar chart with default settings
java_import org.jfree.chart.ChartFactory
bar_chart = ChartFactory.create_bar_chart "How Many Ice Cream Flavors?",
                                          "Flavors", "Creamery", bar_data
```

We import the `ChartFactory` class for convenience (which is basically equivalent to doing `ChartFactory = org.jfree.chart.ChartFactory`) and then call `create_bar_chart` to generate a bar chart with default settings. The arguments we pass are the name of the chart, the label for the X axis, and the label for the Y axis, and our dataset.

Now that we have a chart, we can use Java's graphics APIs (provided with the JDK by default) to output a PNG file.

```ruby
# Create a buffered image in memory at 500x500
bar_image = bar_chart.create_buffered_image 500, 500

# Write the image as a PNG to a file
bar_file = File.open("barchart.png", "w")
javax.imageio.ImageIO.write(bar_image, "PNG", bar_file.to_outputstream)
```

JFreeChart's bar chart type knows how to create a Java2D [BufferedImage](https://docs.oracle.com/en/java/javase/21/docs/api/java.desktop/java/awt/image/BufferedImage.html), which we can then pass to [ImageIO](https://docs.oracle.com/en/java/javase/21/docs/api/java.desktop/javax/imageio/ImageIO.html) to write it out. In this case, we even use a Ruby `File` as the target, using a bit of JRuby magic that turns it into a Java [OutputStream](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/io/OutputStream.html)!

With our jar dependencies locked and our script written, we can simply run it with JRuby from a command line.

```text
jruby-charts $ jruby examples/barchart.rb                      
2025-04-30 14:58:21.744 java[84148:2288876] +[IMKClient subclass]: chose IMKClient_Modern
2025-04-30 14:58:21.744 java[84148:2288876] +[IMKInputSession subclass]: chose IMKInputSession_Modern
```

Depending on your platform, you may see a little bit of "helpful" JVM output indicating that the graphics subsystem has been loaded.

Let's see what we just made!

![A bar chart generated by JRuby and JFreeChart](/images/barchart.png)

Just a few lines of code and we're done! Such fun!

Everyone loves pie!
-------------------

The Java integration walkthrough and bar chart example above should whet your appetite for doing cool things with JRuby, but I include a [pie chart example](https://github.com/headius/jruby-charts/blob/master/examples/piechart.rb) here to show a few differences:

```ruby
# Use jar-dependencies, included with JRuby, to load JFreeChart
require 'jar-dependencies'
require_jar 'org.jfree', 'jfreechart', '1.5.5'

# Create an empty PieDataset
pie_data = org.jfree.data.general.DefaultPieDataset.new

# Add values to the dataset
pie_data.insert_value 0, "Fun", 0.45
pie_data.insert_value 1, "Useful", 0.25
pie_data.insert_value 2, "Cool", 0.15
pie_data.insert_value 3, "Enterprisey", 0.10
pie_data.insert_value 4, "Exciting", 0.5

# Create a pie chart with default settings
pie_chart = org.jfree.chart.ChartFactory.create_pie_chart "Why JRuby?", pie_data

# Anti-alias the chart to look a bit cleaner
pie_chart.anti_alias = true

# Access the actual PiePlot to tweak additional settings
pie_plot = pie_chart.plot
pie_plot.set_explode_percent "Fun", 0.20

# Create a buffered image in memory at 500x500
pie_image = pie_chart.create_buffered_image 500, 500

# Write the image as a GIF to a file

pie_file = File.open("piechart.gif", "w")
javax.imageio.ImageIO.write(pie_image, "gif", pie_file.to_outputstream)
```

This example takes advantage of a few customizations provided by JFreeChart:

* The edges of the pie are set to anti-alias for a cleaner look (`pie_chart.anti_alias = true`, which calls [setAntiAlias](https://www.jfree.org/jfreechart/javadoc/org/jfree/chart/JFreeChart.html#setAntiAlias(boolean))).
* We access the actual [PiePlot](https://www.jfree.org/jfreechart/javadoc/org/jfree/chart/plot/PiePlot.html) object to ["explode"](https://www.jfree.org/jfreechart/javadoc/org/jfree/chart/plot/PiePlot.html#setExplodePercent(K,double)) one of the elements out of the pie.
* Instead of a PNG, we output a GIF, just because. The standard [Java ImageIO support](https://docs.oracle.com/en/java/javase/21/docs/api/java.desktop/javax/imageio/package-summary.html) can handle BMP, GIF, JPEG, PNG, TIFF, and WBMP, and there's third-party support for everything else.

And here's the resulting pie chart:

![A pie chart generated by JRuby and JFreeChart](/images/piechart.gif)

It's as easy as pie!

Your Turn
=========

In this post, you learned the following:

* Basics of Java integration in JRuby
* How to fetch and load Java libraries from Maven
* A few simple ways to create charts with JFreeChart

We didn't have to write a single line of Java code, and we didn't have to call out to any nasty C, Python, or JavaScript libraries. The code you see and the libraries we loaded all run in the same JVM process alongside your Ruby code, and can be easily deployed to any system with a JDK. It's really that simple!

JFreeChart is just one charting library out of many in the Java ecosystem, and there's thousands of other useful libraries you can start using with JRuby today. Need to generate PDFs or Office documents? Try [OpenPDF](https://github.com/LibrePDF/OpenPDF) or [Apache Poi](https://poi.apache.org/). Need to integrate with unusual databases? [JDBC](https://docs.oracle.com/en/java/javase/21/docs/api/java.sql/java/sql/package-summary.html) has you covered with a standard API. Want to deploy a single binary for your entire application? JRuby's [Warbler](https://github.com/jruby/warbler) project allows you to bundle everything up as a single jar file.

Ruby faces many challenges these days, and we're solving them one at a time with JRuby and the JVM. I hope you will experiment with JRuby yourself and create something beautiful!

## [Join the discussion on Reddit!](https://www.reddit.com/r/ruby/comments/1kbqh5l/creating_beautiful_charts_with_jruby_and/)

_JRuby Support and Sponsorship_
===============================

_This is a call to action!_
---------------------------

_JRuby development is funded entirely through your generous sponsorships and the sale of commercial support contracts for JRuby developers and enterprises around the world. If you find my work exciting or believe it is important your company or your projects, please consider partnering with me to keep JRuby strong and moving forward!_

[Sponsor Charles Oliver Nutter on GitHub!](https://github.com/sponsors/headius)

[Sign up for expert JRuby support from Headius Enterprises!](https://www.headius.com/jruby-support)
