---
title: Why Clikt?
sidebar: home_sidebar
permalink: whyclikt.html
---

There are existing Kotlin libraries for creating command line interfaces,
and many Java libraries work in Kotlin as well. However, none of them
had all of the following features:

* Unrestricted composability of commands
* Fully static type safety for parameters
* Composable parameter customization that doesn't require registering converter objects.
* Full support for Unix command line conventions
* Capable of reading parameter values from environment variables out of the box
* Included support for common functionality (keyboard interactivity, line ending normalization, launching editors, etc.)
* Built-in support for multi-token command aliases

Clikt is focused on making writing robust, posix-compliant command line
interfaces as easy as possible. A good CLI does more than just parse
`argv`. It allows users to specify values in environment variables, and
in some cases prompts for additional input, or opens an editor. Clikt
supports all of this out of the box.

Some times you need to make a CLI that doesn't follow Unix conventions.
You might be writing for windows, or you want to use the Java style of
long options with a single dash. Maybe you need to use a bunch of
required options instead of arguments, or you want the help page
formatted differently. "Best practices" might not be the best for you,
so Clikt tries to make implementing uncommon use-cases as easy as
possible.

## Why not a Kotlin library like kotlin-argparse or kotlinx.cli?

Clikt didn't invent the idea of a property delegate-based cli parser.
JetBrains made [kotlinx.cli](https://github.com/Kotlin/kotlinx.cli),
which is functional, but is more of a proof-of-concept than a production
ready library.

[kotlin-argparser](https://github.com/xenomachina/kotlin-argparser)
builds off of [kotlinx.cli](https://github.com/Kotlin/kotlinx.cli), and
works well for simple cases. It's missing a lot of features that Clikt
has, but features can be added. Its real drawback is that it
fundamentally does not support composition of commands or parameter
values. The lack of subcommand support was already a non-starter, but
there are other design descisions that make it unsuitable.

In the simple cases, the two libraries are similar. Here's an example
from its README:

```kotlin
class MyArgs(parser: ArgParser) {
    val v: Boolean by parser.flagging(help="enable verbose mode")
    val name: String? by parser.storing(help="name of the user")
    val count: Int? by parser.storing(help="number of the widgets") { toInt() }
    val source: String by parser.positional(help="source filename")
    val destination: List<String> by parser.positional(help="destination filename")
}

fun main(args: Array<String>) = mainBody {
    ArgParser(args).parseInto(::MyArgs).run {
        println("Hello, $name!")
        println("Moving $count widgets from $source to $destination.")
    }
}
```

Here's the same thing with Clikt:

```kotlin
class Cli : CliktCommand() {
    val v: Boolean by option(help = "enable verbose mode").flag()
    val username: String? by option(help = "name of the user")
    val count: Int? by option(help = "number of the widgets").int()
    val source: String by argument(help = "source filename")
    val destination: List<String> by argument(help = "destination filename").multiple()
    override fun run() {
        println("Hello, $name!")
        println("Moving $count widgets from $source to $destination.")
    }
}

fun main(args: Array<String>) = Cli().main(args)
```

Both work fine, although you may find Clikt more consistent and a bit
less verbose. The differences become more pronounced once you try to do
anything that isn't built in to [kotlin-argparser](https://github.com/xenomachina/kotlin-argparser).

Maybe you need an option to take two values. Here's another example from
the `kotlin-argparser` README showing how to do that:

```kotlin
fun ArgParser.putting(vararg names: String, help: String) =
          option<MutableMap<String, String>>(*names,
                  argNames = listOf("KEY", "VALUE"),
                  help = help) {
              value.orElse { mutableMapOf<String, String>() }.apply {
                  put(arguments.first(), arguments.last()) }
          }

 fun ArgParser.putting(help: String) =
          ArgParser.DelegateProvider { identifier ->
              putting(identifierToOptionName(identifier), help = help) }

class MyArgs(parser: ArgParser) {
    val v by parser.putting(help="this takes two values")
}
```

Clikt has that functionality built in as `paired()`,  <!-- TODO: link to paired -->
but you could implement it yourself like this:

```kotlin
class Cli : CliktCommand() {
    val v by option().transformNargs(2) { it[0] to it[1] }
}
```

The Clikt version is of course much simpler, but there are more
fundamental issues with the `kotlin-argparser` version that drove the
creation of Clikt:

* Its inheritance-based design means that if you wanted to change the type of each value, you would have to copy all of the code for each type. With Clikt, you could just do `option().int().transformNargs(2) { it[0] to it[1] }`
* Its inheritance-based design means that supporting types, multiple values, and multiple option occurances would require a combinatorial copies of the above code. With Clikt, these are all orthoganal.
* You have to do all error checking yourself. The `argparser` example silently discards extra values, or copies the single value, rather than inform the user of the mistake. You could write more code to do so, but Clikt takes care of it for you.
* Option name inferrence is not automatic, requiring you to wrap the delegate with yet another function.
* Each delegate function has a different name, with no indication of whether its creating an option or positional argument. With Clikt, all options are created with `option()`, and all arguments with `argument()`.

Some of these problems can be solved by writing more code, and some
can't. Either way, Clikt attempts to consistent, intuitive, composable
interface that tries to do the right thing without forcing you to think
about edge cases.

{% include links.html %}