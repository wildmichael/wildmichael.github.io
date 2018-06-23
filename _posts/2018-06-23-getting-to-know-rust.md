---
layout: post
title:  "Getting to Know Rust"
date:   2018-06-23 12:03:22 +0200
categories: programming
---

All aboard the hype train! Everybody's raving about this great new programming
language called [_Rust_][rust-hype]. Of all possible names, this is one that
did not inspire immediate trust in me. I guess that is me being a mechanical
engineer... Any ways, this being the first blog post here, I wanted to record
my experience, mostly for my own benefit.

## Installation

The official [Rust Book][rust-book-install] recommends the usage of
[rustup][rustup] for the installation and updating of the whole Rust tool chain.
Being a long-time Ubuntu user, I don't agree with the notion of third-party
tools messing with my `/usr`, so I passed on that option, and went with a
simple `apt install rustc cargo`. The first package contains the compiler,
the second provides the `cargo` build system and dependency manager. You'll
need to have the _Universe_ repositories enabled for this to work. Running

{% highlight bash %}
$ rustc --version
rustc 1.24.1
{% endhighlight %}

reveals that the compiler is there and that things look good. The fact that I
am running not the most recent version (at the time of writing that would be
[1.27.0][rust-1.27.0]) is for the purpose of these experiments of minor
importance.

## First Baby Steps

Every programming tutorial should start with the infamous _Hello, world!_
example. Rust makes this particularly easy when using the `cargo` tool.
Initialize a new Rust project with:

{% highlight bash %}
$ cargo new hello_world --bin
$ cd hello_world
{% endhighlight %}

Inspecting the directory tree I found the following files:

{% highlight bash %}
$ tree .
.
├── Cargo.toml
└── src
    └── main.rs

1 directory, 2 files
{% endhighlight %}

As you can see, the tool created the source file `src/main.rs` and added
the file `Cargo.toml` which contains the build instructions and dependency
definitions. Looking into the source file we see that the _Hello, world!_ code
is already there:

<figure>
<figcaption>File: <kbd>src/main.rs</kbd></figcaption>
{% highlight rust %}
fn main() {
    println!("Hello, world!");
}
{% endhighlight %}
</figure>

Not too many surprises there. What caught my eye is the exclamation mark in
`println!`. It indicates that the name does not refer to a function, but to
a macro instead. Peculiar, but each to his own... The only other noteworthy
thing here is that `fn` is used to declare a function, as you would with
`function` in JavaScript or `def` in Python. Also, as in C/C++, the entry
point is the `main` function. So far so good.

The next file to look at, is the build definition:

<figure>
<figcaption>File: <kbd>Cargo.toml</kbd></figcaption>
{% highlight toml %}
[package]
name = "hello_world"
version = "0.1.0"
authors = ["Michael Wild <themiwi@users.sourceforge.net>"]

[dependencies]
{% endhighlight %}
</figure>

As the name suffix indicates, this file is in the [TOML][toml] format. If you
didn't know it already, think of it as of what would come out if INI and YAML
had a baby.

The contents is simple enough. It declares a package named `hello_world`,
gives a version number and lists me as the author. Conveniently `cargo`
pulled this information from my `~/.gitconfig`. As we have not yet added
any dependencies, the corresponding group is empty.

Also, what was not show in the `tree` output above: `cargo new` directly
initializes a [Git][git] repository in the project directory, with an
approriate `.gitignore` file already in place. Nice and handy.

Compiling and running the code is as easy as invoking

{% highlight bash %}
$ cargo run
   Compiling hello_world v0.1.0 (file:///home/mwild/Projects/rust/hello_world)
    Finished dev [unoptimized + debuginfo] target(s) in 6.3 secs
     Running `target/debug/hello_world`
Hello, world!
{% endhighlight %}

This compiles and runs in one step. Note, that there are also the sub-commands
`cargo build` and `cargo check` for building and syntax-checking, respectively.

Now that the code has been compiled, the resulting executable can also be
directly run:

{% highlight bash %}
$ ./target/debug/hello_world
Hello, world!
{% endhighlight %}

You will have noticed that there is now a new file `Cargo.lock` and a directory
called `target/`. The former is an automatically generated file that fixes all
dependency versions, enabling reproducible builds. It should be definitely
committed to version control. You can bring it up-to-date with `cargo update`.

The directory `target/` contains the build output. Let's take a look:

{% highlight bash %}
$ tree -a target
target
└── debug
    ├── build
    ├── .cargo-lock
    ├── deps
    │   ├── hello_world-23e1044645ca665a
    │   └── hello_world-23e1044645ca665a.d
    ├── examples
    ├── .fingerprint
    │   └── hello_world-23e1044645ca665a
    │       ├── bin-hello_world-23e1044645ca665a
    │       ├── bin-hello_world-23e1044645ca665a.json
    │       └── dep-bin-hello_world-23e1044645ca665a
    ├── hello_world
    ├── hello_world.d
    ├── incremental
    │   └── hello_world-2n621tcba0fv4
    │       ├── s-f293r0yvlo-1fos1jy-2ipc26o60gus8
    │       │   ├── 1y16o1qfye96o7m0.o
    │       │   ├── 3ff08c0u0kk7cb3j.o
    │       │   ├── 3rngp6bm2u2q5z0y.o
    │       │   ├── 4xq48u46a1pwiqn7.o
    │       │   ├── dep-graph.bin
    │       │   ├── query-cache.bin
    │       │   └── work-products.bin
    │       └── s-f293r0yvlo-1fos1jy.lock
    └── native

10 directories, 16 files
{% endhighlight %}

As you can see, it mostly contains object files and information for incremental
builds. Also, there seems to be something prepared for dependencies. I assume
the `.lock` files are there to avoid multiple `cargo` process stepping on each
others toes.

## Getting Fancier

Most command-line utilities need to perform argument parsing. Here I want to
expand the trivial _Hello, World!_ a bit and get to know some more Rust
features. Coming from a Python background I got to love the excellent
[argparse][py-argparse] module. Let's try our luck and see whether Rust
has something similar in store for us:

{% highlight bash %}
$ cargo search argparse
    Updating registry `https://github.com/rust-lang/crates.io-index`
argparse = "0.2.1"       # Powerful command-line argument parsing library
argparse-rs = "0.1.0"    # A simple argument parser, meant to parse command line input. It is inspired by the Python ArgumentPars…
argonaut = "0.12.0"      # A simple argument parser
autojump = "0.3.1"       # A Rust port and drop-in replacement of autojump
{% endhighlight %}

I'll try my luck with `argparse` (no other reason than it's name...) and
modify the `Cargo.toml` file:

<figure>
<figcaption>File: <kbd>Cargo.toml</kbd></figcaption>
{% highlight toml %}
[package]
name = "hello_world"
version = "0.1.0"
authors = ["Michael Wild <themiwi@users.sourceforge.net>"]

[dependencies]
argparse = "0"
{% endhighlight %}
</figure>

The part after the equal sign indicates that all versions compatible with
version 0 are acceptable. Rust and the packages use [semantic
versioning][semver], so requiring a fixed major version should prevent API
breaking changes to occur on `cargo update`.

Now, I know nothing about how to use this `argparse` package. But no worries,
`cargo` got our back:

{% highlight bash %}
$ cargo doc --open
 Updating registry `https://github.com/rust-lang/crates.io-index`
 Downloading argparse v0.2.1
   Compiling argparse v0.2.1
 Documenting argparse v0.2.1
 Documenting hello_world v0.1.0 (file:///home/mwild/Projects/rust/hello_world)
    Finished dev [unoptimized + debuginfo] target(s) in 6.24 secs
     Opening /home/mwild/Projects/rust/hello_world/target/doc/hello_world/index.html
   Launching xdg-open
{% endhighlight %}

Upon completion, a browser window should pop up. Clicking `argparse` in the
left-hand menu brings up the package's documentation. **Very slick!**

OK, let's put this to some use:

<figure>
<figcaption>File: <kbd>src/main.rs</kbd></figcaption>
{% highlight rust %}
extern crate argparse;

fn main() {
    let parser = argparse::ArgumentParser::new();
    parser.parse_args_or_exist();
    println!("Hello, world!");
}
{% endhighlight %}
</figure>

That wasn't too difficult. The first line is similar to `import` in Python. Its
purpose is to instruct the compiler that we want to use the `argparse` library
and use its symbols. The variable assignment and type inference going on with
the `let` statement should also be obvious. All in all, nothing surprising.

Now, let's see whether it works:

{% highlight bash %}
$ cargo build -q
$ ./target/debug/hello_world --help
Usage:
    ./target/debug/hello_world

A friendly program to say hello

optional arguments:
  -h,--help             show this help message and exit
{% endhighlight %}

Great, that works as expected. Now, let's add an optional `--name` argument:

<figure>
<figcaption>File: <kbd>src/main.rs</kbd></figcaption>
{% highlight rust %}
extern crate argparse;

fn main() {
    let mut name = String::from("world");
    {
        let mut parser = argparse::ArgumentParser::new();
        parser.set_description("A friendly program to say hello");
        parser.refer(&mut name)
            .add_option(&["-n", "--name"], argparse::Store,
                        "Name to greet");
        parser.parse_args_or_exit();
    }
    println!("Hello, {}!", name);
}
{% endhighlight %}
</figure>

*Woahh, hold the horses!* What's going on. That doesn't look nowhere near as
close to my familiar Python and C++ as the stuff before. Let's break it down,
line by line:

{% highlight rust %}
    let mut name = String::from("world");
{% endhighlight %}

Reading this from right-to-left: The string literal `"world"` is converted to
a heap-allocated `String` object and assigned to the variable `name`. What does
the `mut` do, you ask? Variables in Rust are by default *immutable*! Since we
want to be able to assign a user-provided value at runtime, we must mark the
variable as *mutable* by using the `mut` keyword. You will also have noticed
that the `parser` variable declaration gained the `mut` keyword. The reason
for this will become clear further down. Also, there is now a new set of
curly braces wrapping the majority of the code whose meaning will be explained
in a minute.

{% highlight rust %}
        parser.refer(&mut name)
            .add_option(&["-n", "--name"], argparse::Store,
                        "Name to greet");
{% endhighlight %}

The statement `parser.refer(&mut name)` warrants some explanation. It's purpose
is to *"borrow"* a variable for modification by an argument. *But what does it
mean?* I hear you asking. The heap memory management in Rust is different than
that of the other well known programming languages. It neither uses explicit,
manual `alloc` and `free`, nor does it use a *garbage collector*. It is much
closer to the *RAII* and move-semantics of modern C++. Only one variable can
*own* a value at any given time. Assigning a variable containing a
heap-allocated value to another value *moves* the value to the new variable,
invalidating the original variable. Same happens when passing a variable as
a function argument, or returning a value from a function. Also, whenever a
variable goes out of scope, the memory gets deallocated, just as if it was
stack-allocated. Think `std::unique_ptr` if you're familiar with C++.

Now, if passing a value into a function transfers ownership with
move-semantics, how could we continue using the `name` variable after argument
parsing? Here is where the next concept comes in: *"borrowing"*. That is what
happens in `parser.refer(&mut name)`. The variable `name` is mutably *borrowed*
to the `.refer()` method call, and with that eventually to the `parser` object.
Ownership is only returned once `parser` goes out of scope. And hence we
have the explanation for the additional set of curly braces. Otherwise we would
not be able to use the `name` variable after the argument parsing.

Next up, still in the same statement, we have the `add_option()` method call.
With `&["-n", "--name"]` an array containing the string literals is created
and passed by reference to the function. The second argument instructs the
parser that it should store the option argument, and the last argument provides
a description for the option to be displayed in the help message.

{% highlight rust %}
        parser.parse_args_or_exit();
{% endhighlight %}

This method call retrieves the command line arguments, parses them, and returns
only after successful parsing or exits with an appropriate code otherwise.
Also, if the user passed `-h` or `--help`, the help message is displayed and
the program exits.

{% highlight rust %}
    println!("Hello, {}!", name);
{% endhighlight %}

With this line the greeting message is displayed. The `{}` represents a
placeholder that is used when formatting the `name` argument into the
template string. This should feel familiar to Pythonistas (`str.format()`) or
C# programmers (`String.Format()`).

## Summary

In this blog post I have made first contact with Rust. I set up the programming
tools, most importantly the `rustc` compiler and the `cargo` build system and
package manager. In terms of programming concepts, I introduced the most basic
syntax and made already contact with the very powerful ownership model of Rust.
However, many important topics have not been touched upon at all. Foremost
is error handling (which Rust is also quite opinionated and pedantic about)
and then we haven't seen anything about flow control, structures, enums and
matching, etc. pp. All topics for follow-up posts -- stay tuned.

## Resources

* Rust book: https://doc.rust-lang.org/book/
* Rust community: https://www.rust-lang.org/en-US/community.html
* Rust-by-example: https://doc.rust-lang.org/stable/rust-by-example/

[rust-hype]: https://insights.stackoverflow.com/survey/2018/#most-loved-dreaded-and-wanted
[rust-book-install]: https://doc.rust-lang.org/book/second-edition/ch01-01-installation.html
[rustup]: https://rustup.rs
[rust-1.27.0]: https://github.com/rust-lang/rust/blob/master/RELEASES.md#version-1270-2018-06-21
[toml]: https://github.com/toml-lang/toml
[git]: https://git-scm.com
[py-argparse]: https://docs.python.org/3/library/argparse.html
[semver]: https://semver.org/
