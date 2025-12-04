---
layout: post
title: "Non-null variable declaration in Java using instanceof patterns"
date: "2025-12-04"
author: Charles Oliver Nutter
published: true

---

Ever since [JRuby 10 upgraded to Java 21](https://blog.jruby.org/2025/04/jruby-10-part-1-whats-new), I've been re-learning Java with all the excellent language enhancements of the past decade. One of my favorites has to be the `instanceof` [pattern matching features](https://openjdk.org/jeps/394) added in Java 16. Today I also realized I can use an `instanceof` pattern to null-check and assign a variable at the same time.

Using instanceof to null-check
==============================

When checking if a value in Java is `instanceof` some type, we can get a false result in two cases:

* The value is an instance of a type not equal to or descended from the specified type.
* The value is `null`.

This second property turns instanceof into a weird sort of null-check when applied to a variable or method that matches the `instanceof` type, since we know the only false result must be when it's `null`.

```java
String foo = null;
IO.println(foo instanceof String) // prints "false";
```

Of course this is longer than just saying `foo == null` but _so what_? **It works!**

Using instanceof patterns for inline null checking
==================================================

The above example doesn't really have any practical use, which is probably why you never see folks talking about it. But when combined with an `instanceof` pattern, we can do something more fun.

```java
// getString() is a method that might return null
if (firstCondition) {
    // something
} else if (getString() instanceof String string) {
    IO.println("length: " + string.length());
} else {
    IO.println("string is null");
}
```

We've just done an inline null check and variable declaration in the middle of an `if ... else` chain! The `string` value **must** be non-null in the `length` branch, so we can safely call methods on it. And we know it **must** be null if we end up in the final `else`.

But why?
========

Because it's there?

Without `instanceof` patterns, you'd need to move everything after the `firstCondition` branch into the following `else`, declare a temporary variable for the value of `getString()`, and then do a nested null-checking branch. This pattern (pun intended) allows us to skip all that.

### Is it better?

I can't say. It's frequently shorter, and requires less indented code.

### Does it convey the intent to null-check?

Not if you don't know about this "hidden" behavior of `instanceof`. That's your fault, though!

### Won't it be slower than a null equality check?

Actually, I had this question too, so I fired up the HotSpot disassembler.

```java
class Blah {
    String foo = System.getProperty("foo");
    void main() {
        for (int i = 0; i < 100000000; i++) {
            bar();
        }
    }
    void bar() {
        if (foo instanceof String string) {
            IO.println(string);
        }
    }
}
```

```text
$ java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly Blah.java
OpenJDK 64-Bit Server VM warning: PrintAssembly is enabled; turning on DebugNonSafepoints to gain additional output
...
```

It turns out, HotSpot's just as smart as I am!

Here's what the field access and `instanceof` turn into for this case:

```text
  0x0000000117153544:   ldr		w11, [x1, #0xc]
  0x0000000117153548:   lsl		x10, x11, #3
                          ;*getfield foo {reexecute=0 rethrow=0 return_oop=0}
                          ; - Blah::bar@1 (line 9)
  0x000000011715354c:   cbnz	x10, #0x117153568
  0x0000000117153550:   ldp		x29, x30, [sp, #0x20]
  0x0000000117153554:   add		sp, sp, #0x30
  0x0000000117153558:   ldr		x8, [x28, #0x28]
                          ;   {poll_return}
  0x000000011715355c:   cmp		sp, x8
  0x0000000117153560:   b.hi	#0x117153580
  0x0000000117153564:   ret		
```

Instead of doing a more complex inheritance check, the `foo` field (loaded by `lsl` into the `x10` register from the `Blah` class in `x11`) is compared with zero (if that's not the case, `cbnz` branches to the `println` branch at `0x117153568`), and then the rest of the instructions just tidy up and return from the `bar()` method.

Basically, because the only possible branch in the code is a null check, the JVM's JIT will compile it as such.

So with the performance question out of the way, only one remains:

What do you think?

## Join the discussion on r/java!

---

_Keep the Hits Coming_
======================

_This is a call to action!_
---------------------------

My blog posts, conference talks, and open-source work are all sponsored by [Headius Enterprises](https://headius.com), offering software support and development services for JVM and JRuby users. Our experts have decades of experience building and optimizing complex JVM applications and managing open-source projects.

If you are interested in **scaling** to new heights, deploying your applications in **large enterprises**, or taking advantage of the JVM's **world-class JIT compilers**, **battle-tested libraries**, and **leading-edge AI tools**, let us help you.

[Sign up for expert support from Headius Enterprises!](https://www.headius.com/jruby-support)

If you find my content or my work on JRuby to be interesting and useful, you can also sponsor me directly on GitHub. Your contributions help ensure the posts keep coming and the JRuby project keeps moving forward!

[Sponsor Charles Oliver Nutter on GitHub!](https://github.com/sponsors/headius)
