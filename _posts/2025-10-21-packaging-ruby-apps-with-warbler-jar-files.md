---
layout: post
title: "Packaging Ruby Apps with Warbler: Executable JAR Files"
date: "2025-10-21"
author: Charles Oliver Nutter
published: true

---

[Warbler](https://github.com/jruby/warbler) is the JRuby ecosystem's tool for packaging up Ruby apps with all dependencies in a single deployable file. We've just released an update, so let's explore how to use Warbler to create all-in-one packaged Ruby apps!

Application Packaging for the Java World
========================================

The Java world has been creating distributable, "run anywhere" packages since Java was first released in 1996. Java source code is compiled to bytecode, stored in .class files and then archived together with metadata in JAR files (Java ARchive) that can be run as command-line executable files or as deployable web applications. A JAR file is just a zip file, laid out in a specific way to contain the code and resources your application or library needs, and essentially all libraries for the JVM get distributed as JAR files.

The JRuby distribution includes `lib/jruby.jar`, for example, which is where all of the internal JRuby Java and Ruby code is located:

```text
jruby $ jruby -v
jruby 10.0.3.0-SNAPSHOT (3.4.5) 2025-10-10 0ea455e57a OpenJDK 64-Bit Server VM 21.0.8+9-LTS on 21.0.8+9-LTS +indy +jit [arm64-darwin]
jruby $ ls -l lib/jruby.jar
-rw-r--r--  1 headius  staff  16928929 Oct 14 12:59 lib/jruby.jar
jruby $ java -jar lib/jruby.jar -v
jruby 10.0.3.0-SNAPSHOT (3.4.5) 2025-10-10 0ea455e57a OpenJDK 64-Bit Server VM 21.0.8+9-LTS on 21.0.8+9-LTS +indy +jit [arm64-darwin]
```

When distributed as a deployable web application, the filename typically ends with ".war" (Web ARchive) and contains a combination of code, configuration, and other JAR files (dependencies of the app). Enterprise applications are packaged in ".ear" files (Enterprise ARchive) which will in turn contain jars and wars and additional configuration for enterprise Java servers. At the end of the day, though, they're all just zip files laid out in a particular way.

Write Once, Run Anywhere
------------------------

This is all possible because of the Write Once, Run Anywhere (WORA) promise of JVM bytecode: if your application runs on the JVM, you can write, compile, and package it once and deploy it anywhere without recompiling.

Compare this to the way external dependencies are managed in Ruby:

* Gem files provide installable packaging for Ruby libraries, but they must be unpacked into a specific path on the filesystem to be usable.
* Libraries with native code require separate binary gems for each platform or must be built at install time and installed on the filesystem.
* Ruby itself must be built for each platform you plan to run it on, or downloaded as a platform-specific binary.

Wouldn't it be nice if we could package Ruby applications for distribution without these hassles, in a way that enterprises and security-conscious companies can handle? That's where Warbler comes in!

Warbler: packaging tool for JRuby applications
----------------------------------------------

Warbler is the JRuby tool of choice for packaging Ruby applications for distribution. Given a Ruby utility or application, Warbler can:

* Package your Ruby utility **along with gem and jar dependencies** (including JRuby itself) as a single executable JAR file.
* Package your Rails, Hanami, Sinatra or other web application as a **deployable WAR file** (again with all dependencies included).
* Add a **mini-server** to your WAR file so it can be **run directly at the command line**.
* Precompile all Ruby code to JRuby's bytecode format, to **obfuscate and protect your intellectual property**.

Let's try a simple example to get started.

Warbler in Practice
===================

We'll be packaging a demo utility called [image_voodoo_demo](https://github.com/headius/image_voodoo_demo) for these examples, based on the [image_voodoo](https://github.com/jruby/image_voodoo) image-manipulation gem.

ImageVoodoo is a Ruby wrapper around the JDK's built-in image-processing APIs (inspired by the ImageMagick gem), allowing JRuby users to scale, crop, thumbnail, greyscale, and apply other transformations to images without installing external libraries. This demo takes the example script from ImageVoodoo and turns it into a command-line utility called `voodoo`.

Here's basic use of the `voodoo` command:

```
$ gem install image_voodoo_demo
Successfully installed image_voodoo_demo-0.1.0
Parsing documentation for image_voodoo_demo-0.1.0
Done installing documentation for image_voodoo_demo after 0 seconds
1 gem installed
$ voodoo PXL_20250802_162259596.jpg 
wrote demo files to /Users/headius/work/jruby/PXL_20250802_162259596/
$ ls -1 PXL_20250802_162259596 
border.jpg
cropped_thumb.jpg
cropped.jpg
reduced.jpg
resized.jpg
resized.png
thumb.jpg
```

If you've already got Ruby or JRuby installed, this workflow is fine. But what if you want to distribute this utility to someone with no knowledge of the Ruby ecosystem? Let's package image_voodoo_demo as an executable JAR file!

Executable JAR Files
--------------------

Java's JAR files can be marked as "executable" by including metadata that configures a "main" entry point. In the case of JRuby, the "main" points to `org.jruby.main.Main`, which bootstraps JRuby and runs your code. Warbler can generate a JAR file for your Ruby tool that includes this configuration and launches your Ruby tool directly.

The simplest way to do this is to run Warbler from within the source repo for your utility.

```text
image_voodoo_demo $ ls -1                         
exe
Gemfile
Gemfile.lock
image_voodoo_demo.gemspec
lib
Rakefile
README.md
```

image_voodoo_demo has a pretty standard gem layout, with a `gemspec` and `Gemfile` to specify dependencies and gem configuration.

```text
image_voodoo_demo $ cat exe/voodoo 
#!/usr/bin/env ruby

require 'image_voodoo/demo'

filename = ARGV[0]

ImageVoodoo::Demo.output_demo_files(filename)

image_voodoo_demo $ tail image_voodoo_demo.gemspec 
  spec.bindir = "exe"
  spec.executables = spec.files.grep(%r{\Aexe/}) { |f| File.basename(f) }
  spec.require_paths = ["lib"]

  # Uncomment to register a new dependency of your gem
  spec.add_dependency "image_voodoo", "0.9.3"

  # For more information and examples about making a new gem, check out our
  # guide at: https://bundler.io/guides/creating_gem.html
end
```

The `voodoo` command just loads the demo code and executes it against `ARGV[0]`, and the `gemspec` includes `exe/voodoo` as its sole executable.

With this layout and configuration, using Warbler is simple!

```text
image_voodoo_demo $ gem install warbler
Fetching warbler-2.1.0.gem
Successfully installed warbler-2.1.0
Parsing documentation for warbler-2.1.0
Installing ri documentation for warbler-2.1.0
Done installing documentation for warbler after 0 seconds
1 gem installed
image_voodoo_demo $ warble
rm -f image_voodoo_demo.jar
Creating image_voodoo_demo.jar
```

We've just created an executable JAR containing everything needed for the `voodoo` command:

* The code and executable for image_voodoo_demo
* The image_voodoo gem
* JRuby's own JAR file and dependencies
* The Ruby standard library

We run the jar with the `java -jar` command line:

```text
image_voodoo_demo $ java -jar image_voodoo_demo.jar ~/Downloads/PXL_20250802_162259596.jpg 
wrote demo files to /Users/headius/work/image_voodoo_demo/PXL_20250802_162259596/
```

It works!

Going Deeper
------------

Here's an abbreviated listing of the contents of the `image_voodoo_demo.jar` file:

```text
image_voodoo_demo $ jar tf image_voodoo_demo.jar 
JarMain.class
META-INF/
META-INF/MANIFEST.MF
META-INF/init.rb
META-INF/lib/
META-INF/lib/jruby-core-10.0.2.0-complete.jar
META-INF/lib/jruby-stdlib-10.0.2.0.jar
META-INF/main.rb
gems/
gems/bundler-2.6.9/
gems/bundler-2.6.9/exe/
gems/bundler-2.6.9/exe/bundle
gems/bundler-2.6.9/exe/bundler
gems/image_voodoo-0.9.3/
gems/image_voodoo-0.9.3/Gemfile
gems/image_voodoo-0.9.3/History.txt
gems/image_voodoo-0.9.3/Jars.lock
...
image_voodoo_demo/
image_voodoo_demo/Gemfile
image_voodoo_demo/Gemfile.lock
image_voodoo_demo/README.md
image_voodoo_demo/Rakefile
image_voodoo_demo/exe/
image_voodoo_demo/exe/voodoo
image_voodoo_demo/lib/
image_voodoo_demo/lib/image_voodoo/
image_voodoo_demo/lib/image_voodoo/demo/
image_voodoo_demo/lib/image_voodoo/demo.rb
image_voodoo_demo/lib/image_voodoo/demo/version.rb
specifications/
specifications/bundler-2.6.9.gemspec
specifications/image_voodoo-0.9.3.gemspec
```

* `JarMain.class` is the "main" file for the JVM. It sets up the JRuby runtime and in-archive gem paths and then launches our executable `voodoo` command.
* `META-INF` contains the JAR metadata along with any dependency libraries. In this case it just needs the `jruby-core` and `jruby-stdlib` jars to have a complete JRuby runtime available.
* `gems` contains all dependency gems for this tool.
* `image_voodoo_demo` naturally contains our demo tool's source files.

Let's customize the building of this jar a little bit.

Customizing Warbler
-------------------

Warbler supports a number of configuration settings and can dump a dummy config file to `config/warble.rb`:

*(Note: the config directory must exist to generate the config file; this will be fixed in warbler 2.1.1)*

```text
image_voodoo_demo $ warble config 
cp /Users/headius/work/jruby/lib/ruby/gems/shared/gems/warbler-2.1.0/warble.rb config/warble.rb
```

Let's say my `image_voodoo_demo` utility is super-proprietary IP but I need to deliver it to a customer for use on-premises. The configuration setting we're looking for is `features`:

```text
image_voodoo_demo $ head -12 config/warble.rb
# Disable Rake-environment-task framework detection by uncommenting/setting to false
# Warbler.framework_detection = false

# Warbler web application assembly configuration file
Warbler::Config.new do |config|
  # Features: additional options controlling how the jar is built.
  # Currently the following features are supported:
  # - *gemjar*: package the gem repository in a jar file in WEB-INF/lib
  # - *executable*: embed a web server and make the war executable
  # - *runnable*: allows to run bin scripts e.g. `java -jar my.war -S rake -T`
  # - *compiled*: compile .rb files to .class files
  # config.features = %w(gemjar)
```

We modify this line to `config.features = %w[compiled]` and re-run the `warble` command:

```text
image_voodoo_demo $ warble             
java -classpath "/Users/headius/work/jruby/lib/ruby/gems/shared/gems/jruby-jars-10.0.2.0/lib/jruby-core-10.0.2.0-complete.jar":"/Users/headius/work/jruby/lib/ruby/gems/shared/gems/jruby-jars-10.0.2.0/lib/jruby-stdlib-10.0.2.0.jar"  \
        org.jruby.Main -S jrubyc  "lib/image_voodoo/demo.rb" "lib/image_voodoo/demo/version.rb"
Ignoring prism-1.5.2 because its extensions are not built. Try: gem pristine prism --version 1.5.2
Ignoring resolv-0.6.2 because its extensions are not built. Try: gem pristine resolv --version 0.6.2
rm -f image_voodoo_demo.jar
Creating image_voodoo_demo.jar
rm -f lib/image_voodoo/demo.class lib/image_voodoo/demo/version.class
```

Our build has changed a bit, using JRuby's `jrubyc` command-line compiler to precompile the `image_voodoo_demo` sources into JVM class files containing JRuby bytecode.

```text
image_voodoo_demo $ jar tf image_voodoo_demo.jar | grep image_voodoo_demo/lib
image_voodoo_demo/lib/
image_voodoo_demo/lib/image_voodoo/
image_voodoo_demo/lib/image_voodoo/demo/
image_voodoo_demo/lib/image_voodoo/demo.class
image_voodoo_demo/lib/image_voodoo/demo.rb
image_voodoo_demo/lib/image_voodoo/demo/version.class
image_voodoo_demo/lib/image_voodoo/demo/version.rb
```

The .rb files here are just stubs to load the .class files:

```text
image_voodoo_demo $ unzip image_voodoo_demo.jar image_voodoo_demo/lib/image_voodoo/demo.rb
Archive:  image_voodoo_demo.jar
replace image_voodoo_demo/lib/image_voodoo/demo.rb? [y]es, [n]o, [A]ll, [N]one, [r]ename: y
  inflating: image_voodoo_demo/lib/image_voodoo/demo.rb  
image_voodoo_demo $ cat image_voodoo_demo/lib/image_voodoo/demo.rb 
load __FILE__.sub(/.rb$/, '.class')%  
```

We can use the `javap` disassembler to see the contents of one of these precompiled class files. Good luck turning this back into source code!

```text
image_voodoo_demo $ javap -c -classpath image_voodoo_demo.jar image_voodoo_demo/lib/image_voodoo/demo
Warning: File image_voodoo_demo.jar(/image_voodoo_demo/lib/image_voodoo/demo.class) does not contain class image_voodoo_demo/lib/image_voodoo/demo
Compiled from "lib/image_voodoo/demo.rb"
public class lib.image_voodoo.demo {
  public static {};
    Code:
       0: new           #11                 // class java/lang/StringBuilder
       3: dup
       4: invokespecial #14                 // Method java/lang/StringBuilder."<init>":()V
       7: ldc           #16                 // String \u0000\u0000\u0000\u0002\u0000\u0000\n®\b\tH\u0002\"\u0001\u0000S\u0001z\fimage_voodooÿÿÿÿþ\u0010\u0018lib/image_voodoo/demo.rb\u0002\u0000t\u0000\u0000H\u0003\"\u0001\u0000S\u0001z\u0019image_voodoo/demo/versionÿÿÿÿþ\u0010\u0018lib/image_voodoo/demo.rb\u0003\u0000t\u0000\u0001H\u00053t\u0000\u0002\u0001_\u0000u-t\u0000\u0002\u0001\u0007requireÿÿÿÿÿ\u0007Xt\u0006\u0000_\u0000I\u0001t\u0006\u0000\u0000\u0018lib/image_voodoo/demo.rb\u0006H\u00062t\u0000\u0001_\u0000\u0002H\u0018I\u0002t\u0006\u0000\u0000\u0018lib/image_voodoo/demo.rb\u0019-t\u0000\u0001\u0001ÿÿÿÿÿ\u0007\tXt\u0006\u0000_\u0000I\u0001t\u0006\u0000\u0000\u0018lib/image_voodoo/demo.rb\u00076S\u0003H\u0017I\u0002t\u0006\u0000\u0000\u0018lib/image_voodoo/demo.rb\u0018-:\u0001\u0002ÿÿÿÿÿ\u0011output_demo_filesÿÿÿÿÿ\b\t\ft\u0000\u0000ffU\u0001\u0000fÿÿÿÿÿt\u0000\u0000\nl\u0000\u0000t\u0000\u0000\u0000H\b?t\u0000\u0001s\u0001f$\u0000\u0002t\u0000\u0001ÿÿÿÿþl\u0000\u0000wS\u0004\u0000t\u0000\u0002-t\u0000\u0002\u0003\bfilenameÿÿÿÿÿ\u000bImageVoodooÿÿÿÿÿ\nwith_imageÿÿÿÿÿ0;L\u0015_GLOBAL_ENSURE_BLOCK_\u0000\t\ft\u0005\u0000\u0001ff\nl\u0000\u0000t\u0005\u0000\u0001\u0000H\t?t\u0005\u0001\u0001s\u0001f#\u0000\u0002t\u0005\u0001\u0001\u0002l\u0003\u0001z\u0002.*ÿÿÿÿþ\u0010\u0018lib/image_voodoo/demo.rb\t\u0000t\u0005\u0002\u0001Xl\u0002\u0000t\u0005\u0002\u0001H\n?t\u0005\u0003\u0001s\u0004f\"\u0000\u0005t\u0005\u0003\u0001\u0001l\u0002\u0000\u0000t\u0005\u0004\u0001\u0006L\u0007CL1_LBL\u0000t\u0005\u0004\u0001Xt\u0005\u0005\u0001N\u0001L\u0007CL1_LBL\u0001:L\u0007CL1_LBL\u0000?t\u0005\u0006\u0001s\u0004f\"\u0000\u0006t\u0005\u0006\u0001\u0001l\u0002\u0000\u0000t\u0005\u0007\u0001Xt\u0005\u0005\u0001t\u0005\u0007\u0001:L\u0007CL1_LBL\u0001H\u000b$\u0000\u0007l\u0000\u0000ÿÿÿÿþfdwS\u0005\u0000t\u0005\b\u0001H\f\u0017\u0000\bl\u0000\u0000ÿÿÿÿûfdfÿ\u0000\u0000\u0000\u0000\u0000\u0000\u0000Èfÿ\u0000\u0000\u0000\u0000\u0000\u0000\u0001\u0090fÿ\u0000\u0000\u0000\u0000\u0000\u0000\u0002XwS\u0006\u0000t\u0005\t\u0001H\r$\u0000\tl\u0000\u0000ÿÿÿÿþf2wS\u0007\u0000t\u0005\n\u0001H\u000e\u0017\u0000\nl\u0000\u0000ÿÿÿÿýfdfÿ\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0096wS\b\u0000t\u0005\u000b\u0001H\u0012Xt\u0005\r\u0001{\u0002:\u000bf\u0014:\fz\u0006FF0000ÿÿÿÿþ\u0010\u0018lib/image_voodoo/demo.rb\u0012\"\u0000\rl\u0000\u0000\u0001t\u0005\r\u0001\u0002t\u0005\f\u0001Pt\u0005\u000f\u0001\u0002l\u0002\u0000z\u000b/border.jpgÿÿÿÿþ\u0010\u0018lib/image_voodoo/demo.rb\u0012ÿÿÿÿþ\u000fft\u0018lib/image_voodoo/demo.rb\u0012\"\u0000\u000et\u0005\f\u0001\u0001t\u0005\u000f\u0001\u0000t\u0005\u000e\u0001H\u0013!\u0000\u000fl\u0000\u0000\u0001F?è\u0000\u0000\u0000\u0000\u0000\u0000\u0000t\u0005\u0010\u0001Pt\u0005\u0012\u0001\u0002l\u0002\u0000z\f/reduced.jpgÿÿÿÿþ\u0010\u0018lib/image_voodoo/demo.rb\u0013ÿÿÿÿþ\u0010ft\u0018lib/image_voodoo/demo.rb\u0013\"\u0000\u000et\u0005\u0010\u0001\u0001t\u0005\u0012\u0001\u0000t\u0005\u0011\u0001H\u0014?t\u0005\u0015\u0001s\u0004f%\u0000\u0010t\u0005\u0015\u0001\u0000\u0000t\u0005\u0016\u0001Pt\u0005\u0014\u0001\u0005z\u0014wrote demo files to ÿÿÿÿþ\u0010\u0018lib/image_voodoo/demo.rb\u0014t\u0005\u0016\u0001z\u0001/ÿÿÿÿþ\u0010\u0018lib/image_voodoo/demo.rb\u0014l\u0002\u0000z\u0001/ÿÿÿÿþ\u0010\u0018lib/image_voodoo/demo.rb\u0014ÿÿÿÿþ\u001eft\u0018lib/image_voodoo/demo.rb\u0014\"\u0001\u0011S\u0001t\u0005\u0014\u0001\u0000t\u0005\u0013\u0001-t\u0005\u0013\u0001<:L\u0015_GLOBAL_ENSURE_BLOCK_\u0000\u0012t\u0005\u0017\u0001ct\u0005\u0018\u0001\u0002\u0001t\u0005\u0017\u00010t\u0005\u0018\u0001:L\u0007CL1_LBL\u0002\u0012\u0003imgÿÿÿÿÿ\u0004Fileÿÿÿÿÿ\bbasenameÿÿÿÿÿ\bfilenameÿÿÿÿÿ\u0003Dirÿÿÿÿÿ\u0006exist?ÿÿÿÿÿ\u0005mkdirÿÿÿÿÿ\u0011cropped_thumbnailÿÿÿÿÿ\twith_cropÿÿÿÿÿ\tthumbnailÿÿÿÿÿ\u0006resizeÿÿÿÿÿ\u0005widthÿÿÿÿÿ\u0005colorÿÿÿÿÿ\nadd_borderÿÿÿÿÿ\u0004saveÿÿÿÿÿ\u0007qualityÿÿÿÿÿ\u0003pwdÿÿÿÿÿ\u0004putsÿÿÿÿÿ\r;L\u0015_GLOBAL_ENSURE_BLOCK_\u0000\ft\u0005\u0000\u0002ff\nl\u0000\u0000t\u0005\u0000\u0002\u0000H\u000bPt\u0005\u0002\u0002\u0002l\u0001\u0001z\u0012/cropped_thumb.jpgÿÿÿÿþ\u0010\u0018lib/image_voodoo/demo.rb\u000bÿÿÿÿþ\u0016ft\u0018lib/image_voodoo/demo.rb\u000b\"\u0000\u0002l\u0000\u0000\u0001t\u0005\u0002\u0002\u0000t\u0005\u0001\u0002-t\u0005\u0001\u0002<:L\u0015_GLOBAL_ENSURE_BLOCK_\u0000\u0012t\u0005\u0003\u0002ct\u0005\u0004\u0002\u0002\u0001t\u0005\u0003\u00020t\u0005\u0004\u0002:L\u0007CL2_LBL\u0000\u0003\u0004img2ÿÿÿÿÿ\bbasenameÿÿÿÿÿ\u0004saveÿÿÿÿÿ\r;L\u0015_GLOBAL_ENSURE_BLOCK_\u0000\ft\u0005\u0000\u0003ff\nl\u0000\u0000t\u0005\u0000\u0003\u0000H\fPt\u0005\u0002\u0003\u0002l\u0001\u0001z\f/cropped.jpgÿÿÿÿþ\u0010\u0018lib/image_voodoo/demo.rb\fÿÿÿÿþ\u0010ft\u0018lib/image_voodoo/demo.rb\f\"\u0000\u0002l\u0000\u0000\u0001t\u0005\u0002\u0003\u0000t\u0005\u0001\u0003-t\u0005\u0001\u0003<:L\u0015_GLOBAL_ENSURE_BLOCK_\u0000\u0012t\u0005\u0003\u0003ct\u0005\u0004\u0003\u0002\u0001t\u0005\u0003\u00030t\u0005\u0004\u0003:L\u0007CL3_LBL\u0000\u0003\u0004img2ÿÿÿÿÿ\bbasenameÿÿÿÿÿ\u0004saveÿÿÿÿÿ\r;L\u0015_GLOBAL_ENSURE_BLOCK_\u0000\ft\u0005\u0000\u0004ff\nl\u0000\u0000t\u0005\u0000\u0004\u0000H\rPt\u0005\u0002\u0004\u0002l\u0001\u0001z\n/thumb.jpgÿÿÿÿþ\u0010\u0018lib/image_voodoo/demo.rb\rÿÿÿÿþ\u000eft\u0018lib/image_voodoo/demo.rb\r\"\u0000\u0002l\u0000\u0000\u0001t\u0005\u0002\u0004\u0000t\u0005\u0001\u0004-t\u0005\u0001\u0004<:L\u0015_GLOBAL_ENSURE_BLOCK_\u0000\u0012t\u0005\u0003\u0004ct\u0005\u0004\u0004\u0002\u0001t\u0005\u0003\u00040t\u0005\u0004\u0004:L\u0007CL4_LBL\u0000\u0003\u0004img2ÿÿÿÿÿ\bbasenameÿÿÿÿÿ\u0004saveÿÿÿÿÿ\u0010;L\u0015_GLOBAL_ENSURE_BLOCK_\u0000\ft\u0005\u0000\u0005ff\nl\u0000\u0000t\u0005\u0000\u0005\u0000H\u000fPt\u0005\u0002\u0005\u0002l\u0001\u0001z\f/resized.jpgÿÿÿÿþ\u0010\u0018lib/image_voodoo/demo.rb\u000fÿÿÿÿþ\u0010ft\u0018lib/image_voodoo/demo.rb\u000f\"\u0000\u0002l\u0000\u0000\u0001t\u0005\u0002\u0005\u0000t\u0005\u0001\u0005H\u0010Pt\u0005\u0004\u0005\u0002l\u0001\u0001z\f/resized.pngÿÿÿÿþ\u0010\u0018lib/image_voodoo/demo.rb\u0010ÿÿÿÿþ\u0010ft\u0018lib/image_voodoo/demo.rb\u0010\"\u0000\u0002l\u0000\u0000\u0001t\u0005\u0004\u0005\u0000t\u0005\u0003\u0005-t\u0005\u0003\u0005<:L\u0015_GLOBAL_ENSURE_BLOCK_\u0000\u0012t\u0005\u0005\u0005ct\u0005\u0006\u0005\u0002\u0001t\u0005\u0005\u00050t\u0005\u0006\u0005:L\u0007CL5_LBL\u0000\u0003\u0004img2ÿÿÿÿÿ\bbasenameÿÿÿÿÿ\u0004saveÿÿÿÿÿ\t\u0007\u0000\u0003\u0000\u0000\u0000\u0000ÿ\u0000\u0000\u0000\u0000\u0000\u0000\u0000ÿÿ\u0000\u0000 \u0005ffffffffffff\u0000\bÿ\u0000\u0000\u0000\u0097\u0005\u0005\u0002\u0000\u000bImageVoodooÿÿÿÿÿ\u0000\u0000\u0000\u0000ÿ\u0000\u0000\u0000\u0000\u0000\u0000\u0000ÿÿ\u0000\u0000?øffffffffffff\u0000ÿ\u0000\u0000\u0000¥ÿ\u0000\u0000\u0000û\u0004\u0006\u0001\u0000\u0004Demoÿÿÿÿÿ\u0001\u0000\u0000\u0000ÿ\u0000\u0000\u0000\u0000\u0000\u0000\u0000ÿÿ\u0000\u0000?øffffffffffff\u0000ÿ\u0000\u0000\u0001\u0001ÿ\u0000\u0000\u0001Q\u0003\u0007\u0003\u0000\u0011output_demo_filesÿÿÿÿÿ\u0002\u0000\u0001\bfilename\u0000ÿ\u0000\u0001\u0000\u0000\u0000\u0000\u0000ÿÿ\u0000\u0000?ýffffffffffff\u0000ÿ\u0000\u0000\u0001nÿ\u0000\u0000\u0001¬\u0000\b\u0019\u0003fÿ\u0000\u0001\u0000\u0000\u0000\u0000\u0000ÿ\u0012with_image &|img|1ÿÿÿÿÿ\u0003\u0001\u0002\u0003img\bbasename\u0000ÿ\u0000\u0001\u0000\u0000\u0000\u0000\u0000ÿÿ\u0000\u0000?ýffffftffffff\u0000ÿ\u0000\u0000\u0001Üÿ\u0000\u0000\u0005\u0082\u0000\u000b\u0005\u0001fÿ\u0000\u0001\u0000\u0000\u0000\u0000\u0000ÿ\u001acropped_thumbnail &|img2|2ÿÿÿÿÿ\u0004\u0001\u0001\u0004img2\u0000ÿ\u0000\u0001\u0000\u0000\u0000\u0000\u0000ÿÿ\u0000\u0000 \u0000ffffftffffff\u0000ÿ\u0000\u0000\u0006cÿ\u0000\u0000\u0007=\u0000\f\u0005\u0001fÿ\u0000\u0001\u0000\u0000\u0000\u0000\u0000ÿ\u0012with_crop &|img2|3ÿÿÿÿÿ\u0004\u0001\u0001\u0004img2\u0000ÿ\u0000\u0001\u0000\u0000\u0000\u0000\u0000ÿÿ\u0000\u0000 \u0000ffffftffffff\u0000ÿ\u0000\u0000\u0007`ÿ\u0000\u0000\b4\u0000\r\u0005\u0001fÿ\u0000\u0001\u0000\u0000\u0000\u0000\u0000ÿ\u0012thumbnail &|img2|4ÿÿÿÿÿ\u0004\u0001\u0001\u0004img2\u0000ÿ\u0000\u0001\u0000\u0000\u0000\u0000\u0000ÿÿ\u0000\u0000 \u0000ffffftffffff\u0000ÿ\u0000\u0000\bWÿ\u0000\u0000\t)\u0000\u000e\u0007\u0001fÿ\u0000\u0001\u0000\u0000\u0000\u0000\u0000ÿ\u000fresize &|img2|5ÿÿÿÿÿ\u0004\u0001\u0001\u0004img2\u0000ÿ\u0000\u0001\u0000\u0000\u0000\u0000\u0000ÿÿ\u0000\u0000 \u0000ffffftffffff\u0000ÿ\u0000\u0000\tLÿ\u0000\u0000\n\u008b
       9: invokevirtual #20                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      12: invokevirtual #24                 // Method java/lang/Object.toString:()Ljava/lang/String;
      15: putstatic     #26                 // Field script_ir:Ljava/lang/String;
      18: return

  public static void main(java.lang.String[]);
    Code:
       0: invokestatic  #34                 // Method org/jruby/Ruby.newInstance:()Lorg/jruby/Ruby;
       3: astore_1
       4: aload_1
       5: aload_1
       6: getstatic     #26                 // Field script_ir:Ljava/lang/String;
       9: ldc           #36                 // String ISO-8859-1
      11: invokevirtual #42                 // Method java/lang/String.getBytes:(Ljava/lang/String;)[B
      14: ldc           #43                 // String lib/image_voodoo/demo.rb
      16: invokestatic  #49                 // Method org/jruby/ir/runtime/IRRuntimeHelpers.decodeScopeFromBytes:(Lorg/jruby/Ruby;[BLjava/lang/String;)Lorg/jruby/ir/IRScope;
      19: invokevirtual #53                 // Method org/jruby/Ruby.runInterpreter:(Lorg/jruby/ParseResult;)Lorg/jruby/runtime/builtin/IRubyObject;
      22: return

  public static org.jruby.ir.IRScope loadIR(org.jruby.Ruby, java.lang.String);
    Code:
       0: aload_0
       1: getstatic     #26                 // Field script_ir:Ljava/lang/String;
       4: ldc           #36                 // String ISO-8859-1
       6: invokevirtual #42                 // Method java/lang/String.getBytes:(Ljava/lang/String;)[B
       9: aload_1
      10: invokestatic  #49                 // Method org/jruby/ir/runtime/IRRuntimeHelpers.decodeScopeFromBytes:(Lorg/jruby/Ruby;[BLjava/lang/String;)Lorg/jruby/ir/IRScope;
      13: areturn
```

Future Topics
=============

I'm trying to do more frequent, compact blog posts, so I'm not going to get into the more elaborate generation of deployable web archives today. Here's a few topics I'll try to cover in future posts:

* Generating deployable WAR files from any rack-compatible web application.
* Including jar dependencies from the JVM ecosystem as part of your tool.
* Getting around the JVM requirement: using JVM packaging tools to combine your jar with a complete runnable JVM environment.

A good friend of the JRuby project – Mohit Sindhwani – also [blogged about Warbler back in 2021](https://notepad.onghu.com/2021/jruby-win-day2-creating-jar-files/). His company and many others use Warbler for commercial packaging of Ruby applications every day, ranging from simple command-line utilities all the way up to enterprise scale applications.

The Warbler project just had its first major release in almost a decade, so we're currently updating it for modern JRuby, Ruby, and JVM features and requirements. There's a lot of room for cleanup and new features!

If you're interested in packaging your Ruby tools for easy, secure distribution to friends, family, or customers... please give Warbler a try and let us know how we can help you!

https://github.com/jruby/warbler

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
