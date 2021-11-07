---
layout: post
title:  "Actually Generating Art in Elixir"
subtitle: "(or) the Episode of Oprah Where Everyone Got a Yak"
date:   2021-11-06 10:30:00 -0400
categories: elixir generative
---

To do my best to keep the yaks contained, I'm going to set out a simple, concrete goal for this post:

**from an IEx session, create a new image, paint its background purple, and save it as test.png**

So let's dive in!

### Create a new image in Elixir

To accomplish this goal, our API will need three functions

```elixir
defmodule Image do
  defstruct [...]

  @spec new(number, number) :: Image.t
  def new(width, height)

  @spec paint(Image.t, color) :: Image.t
  def paint(image, color)

  @spec save(Image.t, String.t) :: Image.t
  def save(image, filename)
end
```

In Elixir common practice, the image we are working with is the first argument to every function, and the returned value from each, allowing us to pipe it through the function chain.

At the end of this post, we should be able to run the following in IEx successfully

```elixir
Image.new(100, 100) |> Image.paint(:purple) |> Image.save("test.png")
```

Ok, so we know what we want the Elixir side of this to look like. But almost all of the work do accomplish these things is going to happen in our Rust bindings, so let's move down the stack one layer and see what's up.

### Create a new image in Rust

Having followed the Rustler setup from the previous post, we have a mix application with a linked Rust library. For the sake of semi-clever naming, I've named this application `Xairo`, and, after adding a few Elixir files, our application file structure looks like this:

```tree
.
├── lib
│   ├── xairo
│   │   ├── image.ex
│   │   └── native.ex
│   └── xairo.ex
├── mix.exs
└── native
    └── xairo_native
        ├── Cargo.toml
        ├── README.md
        └── src
            └── lib.rs
```

The Rust `xairo_native` library will compile and be loaded as NIFs into the Elixir `Xairo.Native` module. Then we'll use the `Xairo.Image` module to create our user-facing Elixir API.

