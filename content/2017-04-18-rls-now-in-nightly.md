+++
title = "RLS now available on nightly"
[taxonomies]
tags = [ "rust" ]
+++

We've got some good news for Rust IDE support.

We recently hit a milestone for the Rust Language Server, a tool that combines the power of the compiler with a fast autocompletion engine.  Up to this point, if you wanted to use it you had to build it from scratch.  The process was tedious, time-consuming, and error-prone.

We're happy to report you can now install RLS from [rustup](https://rustup.rs/).  Landing as a component in rustup means a few important things.

First, this will make it **easier to get up and running** with the RLS. You no longer have to build the RLS to use it. Getting it up and running now just takes the time to run a few rustup commands:

```
rustup default nightly
rustup update nightly
rustup component add rls
rustup component add rust-analysis
rustup component add rust-src
```

You can find more helpful tips in the [readme](https://github.com/rust-lang-nursery/rls/blob/master/README.md).

Second, this also makes it **easier for IDE/editor plugins to automatically install the RLS** and use its capabilities.  This opens up the possibility for an even smoother user experience that is driven directly by the IDE plugin.  We're looking forward to exploring this further.

Lastly, it is now **easier to stay up-to-date** with the RLS as it improves.  Once you install the component, it will update alongside your compiler.  All you'll need to do is run the `rustup update nightly` command, and the RLS and its dependencies will update with it.

We've also been hard at work fixing bugs and improving the experience of using the RLS so that it's more stable and accurate than before.  For larger projects, you may also notice that the RLS is a bit faster as well.

We're excited to get this out to more people and hear your [feedback](https://github.com/rust-lang-nursery/rls/issues).  There's still plenty of work to do, and if you'd like to jump in and [help](https://github.com/rust-lang-nursery/rls/blob/master/contributing.md), we'd love to have you.
