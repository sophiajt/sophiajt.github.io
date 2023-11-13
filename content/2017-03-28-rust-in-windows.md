+++
title = "Using Rust in Windows"
[taxonomies]
tags = [ "rust" ]
+++

As a Rust user, I was discouraged by not having all the good tips for using Rust in Windows in one place.  This is an attempt to just that.  If you have more tips tweet them to @jntrnr, and I'll try to add them here.

# Getting started

## Step 1: Install Visual Studio Community edition

My recommendation for working with Rust in Windows is to first get [Visual Studio 2017 Community](https://www.visualstudio.com/downloads/) installed.  *Edit:* Make sure you check the boxes to install **C++ support** when you instal Visual Studio, this provides the linker. The Rust compiler will use the linker that VS provides to translate our Rust code all the way to being an application.  Having VS installed pays off in more ways than one, as we'll see later.

## Step 2: Install rustup

The Rust toolchain (including the compiler and the `cargo` build tool) are managed by a tool called [rustup](https://rustup.rs/).  This is the easiest way get going with Rust.  It also has a few advantages, like allowing you to more easily switch the style of compiler (stable/beta/nightly) as well as building for other targets, like embedded systems.

Once Visual Studio is installed, rustup will detect it and let us install the compiler that works with the VS toolchain.  Rustup on the initial install will let you pick the compiler, and you can select working with nightly or release.  What you choose is up to you.  Nightly gives you access to new features but hasn't been tested as thoroughly as the stable release compiler.

Whether you choose Nightly or Stable, you'll have commands available to you at the terminal.  You won't need a separate shell to get to them.  The commands 'rustc' and 'cargo' are the Rust compiler and Rust package manager respectively. 

## Step 3: Make sure everything is working

In a source directory, you can test if Rust has been installed correctly by creating a simple project, compiling it, and running it.

```
cargo new hello_world --bin
cd hello_world
cargo run (or "cargo build" followed by "cargo run")
```

You can find out lots of helpful general [getting started tips](https://doc.rust-lang.org/book/getting-started.html) in the Rust documentation.

# Writing Rust Code in Windows

Currently, IDE support for Rust is still in its infancy.  If you don't need fully intellisense-style IDE experience, pretty much all editors now have basic Rust support.  Pick your favorite one, install their Rust plugin or syntax highlighter, and you're good to go.  Many of these plugins come with support for a project called 'racer', which provides an autocompletion.  Visual Studio users may be interested in [Visual Rust](https://marketplace.visualstudio.com/items?itemName=vosen.VisualRust).

For the more daring, there's a project called the [Rust Language Server](https://github.com/rust-lang-nursery/rls), which is based on the [Language Server Protocol](https://github.com/Microsoft/language-server-protocol/blob/master/protocol.md) developed for VS Code and now part of a growing number of editors and IDEs.

You can find steps for how to set up the RLS and use it with VS Code on its [readme](https://github.com/rust-lang-nursery/rls/blob/master/README.md).  That said, the RLS is still pre-beta quality as of the writing of this post, so please exercise caution if you go this route.

# Debugging with Visual Studio

Earlier we installed Visual Studio.  It seems a waste to have this nice VS install sitting around and not getting more benefit from it it.  Let's use it for debugging.

Before we do some debugging, there are two steps we'll want to do beforehand:

## Step 1: Install rust-src

When we're debugging, VS will sometimes complain because it can't find the source code to the standard libs.  We can remove this announce by installing the rust-src, and helping VS find the source it's looking for (this is also a helpful step for `racer` support)

```
rustup component add rust-src
```

This will let you find the source to the standard libs when you're debugging.  After the command is run, it will be put in your .multirust directory.  For me that directory is:

```
C:\Users\Jonathan Turner\.multirust\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\src
```

When Visual Studio asks you for a file from the standard library source, this should give you a good place to start looking for it.

## Step 2: Add natvis for Rust

Rust has its own common structs, like String and Vec, that Visual Studio won't understand by default currently.  To be able to view these data structures, we use native visualisation or .natvis files.  Rustup will helpfully make them available for us, we just have to copy them into our personal directory.  For me, the .natvis files are in:

```
C:\Users\Jonathan Turner\.multirust\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\src\etc\natvis
```

NOTE: you may need to install the `nightly` toolchain to have this directory.  You don't need to use the nightly compiler as your main compiler, though, to take advantage of them.

I copy these files into:

```
C:\Users\Jonathan Turner\Documents\Visual Studio 2017\Visualizers
```

If you don't have a `Visualizers` directory, go ahead and create it and then copy the files into it.  Once you start up VS, it should pick them up and give you a little better view of your Rust data.

With that, debugging a Rust app works very similar to debugging other apps.  There are a few steps you need to do:

* Compile your projects with debug info.  The easiest way is to build using `cargo build` and NOT `cargo build --release`.
* Run your application
* Attach the VS debugger using the 'Attach...' feature.  You should be able to find your application in the list.  Once you attach, you can hit pause, set breakpoints, and step as normal.

# Debugging with VSCode and MSVC

You can also setup vscode as your development and debugging platform.  There are some good tips on how to do this in [this blog post](https://www.brycevandyk.com/debug-rust-on-windows-with-visual-studio-code-and-the-msvc-debugger/).

# Conclusion

There is still plenty of room for improvement, but already there's enough here that we can write, build, and debug Rust applications without too much fuss.  The Rust teams are working to continue to improve the experience of developing in Windows on Rust, but it's encouraging to see the progress and where it could go in the future.
