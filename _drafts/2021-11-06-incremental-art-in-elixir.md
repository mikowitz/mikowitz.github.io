---
layout: post
title:  "Generating Art in Elixir"
subtitle: "(or) My Time in the Yak Mines"
date:   2021-11-06 10:30:00 -0400
categories: elixir generative
---

I have been interested in generative art for a long time. Over the years I have experimented primarily in [Processing](https://processing.org), variously its Java, Python, and p5.js modes, and, for a time, the now-deprecated ruby-processing library. As I've become more bullish on [Elixir](https://elixir-lang.org) for my personal projects, I've wanted a similar Elixir library to work with, but none has been forthcoming.

Which, me being me, means I set out to write one.

In the course of my searching, I came across the Haskell [cairo package](https://hackage.haskell.org/package/cairo), Haskell bindings for the [C cairo 2d grahpics library](https://www.cairographics.org), as well as an excellent introduction to it on [Ben Kovach's blog](https://www.kovach.me/Generating_artwork_with_Haskell.html).

Now that I knew about the C cairo library, and knowing that it is possible to create C bindings to Erlang/Elixir using [erl_nif](https://www.erlang.org/doc/man/erl_nif.html), after some thought I determined that I preferred NIFs to Monads. No disrespect to Haskell, but while generating art was certainly one of my focuses embarking on this project, doing so with Elixir was the primary goal. Also, Monads make my brain hurt, and while I want to learn more about type theory than I currently know, I decided this project was not the time.

### First steps

I spent some time experimenting with writing cairo bindings directly in C, with...success? In some ways creating the interface was quite simple. A C method with a specific argument signature and a declaration of functions to bind.

The example below shows a needlessly basic of implementing adding two integers together:

```c
#include "erl_nif.h"

static ERL_NIF_TERM
nif_add(ErlNifEnv *env, int argc, const ERL_NIF_TERM argv[]) {
  int a, b;
  enif_get_int(env, argv[0], &a);
  enif_get_int(env, argv[1], &b);

  int result = a + b;

  return enif_make_int(env, result);
}

static ErlNifFunc nif_funcs[] = {
  {"nif_add", 2, nif_add}
};

ERL_NIF_INIT(Elixir.MyNifModule, nif_funcs, NULL, NULL, NULL, NULL)
```

and in Elixir, the matching module to load the NIFs and function definitions that match the bound functions and raise an error if the NIFs cannot be loaded

```elixir
defmodule MyNifModule do
  @on_load :load_nifs

  def load_nifs do
    :erlang.load_nif('./addition', 0)
  end

  def nif_add(_a, _b), do: raise "NIF nif_add/1 not implemented"
end
```

In broad strokes, that's really the root of it. Which seems simple enough, on the surface. But, much like Monads, manual memory management makes my head hurt, and once you include the `cairo` structs and start trying to manage their state between C and Elixir, there's, well, a lot of that.

So now what?

### C + memory-safety == Rust

I've been interested in [Rust](https://www.rust-lang.org/) for a while, but I've never had a project that made sense to use it for. Until now.

With a bit more searching, I found the next piece of the puzzle: [Rustler](https://github.com/rusterlium/rustler), which, according to its README, is

> a library for writing Erlang NIFs in safe Rust code. That means there should be no ways to crash the BEAM (Erlang VM). The library provides facilities for generating the boilerplate for interacting with the BEAM, handles encoding and decoding of Erlang terms, and catches rust panics before they unwind into C.

Which, hey, sounds perfect! Plus, it comes with all the hooks into Elixir's `mix` command so you don't have to write your own Makefiles.

So I followed the instructions in the README, and it generates a nice little template to build off of, which includes the same `add` function as my example above. Let's see what it looks like in Rust:

```rust
#[rustler::nif]
fn add(a: i64, b: i64) -> i64 {
    a + b
}

rustler::init!("Elixir.Math", [add]);
```

and the Elixir part

```elixir
defmodule Math do
  use Rustler, otp_app: :math, crate: "math"

  def add(_a, _b), do: :erlang.nif_error(:nif_not_loaded)
end
```

Even at a quick glance, and even without digging into the particulars of Rust syntax[^1], we can see the parallels


* the C `ERL_NIF_TERM` return type becomes the Rust attribute `#[rustler::nif]`, marking the function as a NIF
* `ERL_NIF_INIT` becomes `rustler::init!`
* the `use` statement in Elixir instead of needing to manually call `:erlang.load_nif/2`

and one important difference: instead of needing to pass all our arguments in as an `ERL_NIF_TERM` array and parse them out, we can use native Rust types in our function signatures. This is going to make my life a lot easier.

In extreme cases, we can drop down to a more explicit function signature that more closely matches the C code,

```rust
fn add<'a>(env: Env<'a>, args: &[Term<'a>]) -> Result<Term<'a>, Error>
```

but, as we'll see later, this is rarely necessary, as Rustler provides the [Encoder trait](https://docs.rs/rustler/0.22.2/rustler/types/trait.Encoder.html) that allows sending native Rust types (and a few defined in Rustler), as well as any structs built from them, directly between Rust and Elixir safely. As an added bonus, Rustler also provides macro attributes that allow us to define a struct in both Rust and Elixir and pass it back and forth directly.

### Ok, but what about the art generating?

Right, so, as I do tend do, I've gone down a bit of a yak shave here. Let's shake all this loose fur off and get back to where I meant to be. With our Rust to/from Elixir layer sorted out, we just need a way to get cairo functionality from C to Rust. And here the Rust community will save us again, with [cairo-rs](https://crates.io/crates/cairo-rs), "Rust bindings for the Cairo library".

So, yak shaved.

Which, I think, is an excellent number of yaks for a single blog post. Next time, we'll begin the work of putting these pieces together.

<hr/>
<br/>

[^1]: which I'm not going to do in these posts to any great extent. There are far better resources for this than I can provide. [The Rust Programming Language](https://doc.rust-lang.org/book/title-page.html) by Steve Klabnik and Carol Nichols, for example, is a great starting point.

