---
layout: post
title:  "Actually Generating Art in Elixir"
subtitle: "(or) the Episode of Oprah Where Everyone Got a Yak"
date:   2021-12-20 10:30:00 -0400
categories: elixir rust generative xairo
partno: 2
---

As promised in my previous post, I'm going to do my best to keep the yaks
contained. So I'm going to set out a simple, concrete goal for this post:

**from an IEx session, create a new image, paint its background purple, and
save it as test.png**

So let's dive in!

{% include xairo_series.html %}

### Create a new image in Elixir

To accomplish this goal, our API will need three functions

```elixir
defmodule Image do
  defstruct [...]

  @spec new(number, number) :: Image.t
  def new(width, height)

  @spec paint(Image.t, number, number, number) :: Image.t
  def paint(image, red, green, blue)

  @spec save(Image.t, String.t) :: Image.t
  def save(image, filename)
end
```

In Elixir common practice, the image we are working with is the first argument to every function, and the returned value from each, allowing us to pipe it through the function chain.

At the end of this post, we should be able to run the following in IEx successfully

```elixir
Image.new(100, 100) |> Image.paint(0.5, 0.0, 1.0) |> Image.save("test.png")
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
    └── xairo
        ├── Cargo.toml
        ├── README.md
        └── src
            └── lib.rs
```

The Rust `xairo` library will compile and be loaded as NIFs into the Elixir `Xairo.Native` module. Then we'll use the `Xairo.Image` module to create our user-facing Elixir API.

Reading through the [cairo](https://www.cairographics.org/manual/) and [cairo-rs](https://gtk-rs.org/gtk-rs-core/stable/latest/docs/cairo/index.html) docs, as well as some helpful blog posts that cover similar ground[^1], we discover that the two cairo objects we need to concern ourselves with primarily are the `ImageSurface`, which is, well, the surface on which an image is drawn, and the `Context`, which wraps the `ImageSurface` and provides the API for drawing. Pulling from the previously linked blog post[^1], and splitting the code out into individual functions, we get something that almost matches our desired Elixir API

```rust
use cairo::{ImageSurface, Context, Format};
use std::fs::File;

#[rustler::nif]
fn new(width: i32, height: i32) -> Context {
  let surface = ImageSurface::create(Format::ARgb32, width, height);
  Context::new(&surface)
}

#[rustler::nif]
fn paint(context: Context, red: f64, green: f64, blue: f64) -> Context {
  context.set_source_rgb(red, green, blue);
  context.paint();
  context
}

#[rustler::nif]
fn save(context: Context, filename: String) -> Context {
  let mut file = File::create(filename);
  // This won't actually work! because `surface` is not
  // an accessible attribute on a `Context`, but we'll get to that...
  context.surface.write_to_png(&mut file);
  context
}

rustler::init!("Elixir.Xairo.Native", [new, paint, save]);
```

Now, for many reasons, this won't actually compile. But, just for fun, let's build it and see what errors we get. One of the great things about the Rust compiler is that its error messages will very often help point us in the right direction for how to resolve them.

Now, there are a lot of them, but a similar error pops up a number of times, so let's start there:

```
error[E0277]: the trait bound `cairo::Context: Encoder` is not satisfied
   --> src/lib.rs:17:1
    |
17  | #[rustler::nif]
    | ^^^^^^^^^^^^^^^ the trait `Encoder` is not implemented for `cairo::Context`
```

Remember in the last post we talked about this Encoder trait as the trick `rustler` uses to let us pass data from Rust to Elixir and back. Unsurprisingly, the data types that ship with the cairo-rs library don't implement this trait. But can we do it here?

No, we can't. One of Rust's safety tricks is that, in order to implement a trait for a data type, your library must be the origin of either the trait or the data type. We don't own either, so we can't do this. Is this the end? Have we shaved these yaks in vain?

No, we haven't. `rustler` provides us with the `ResourceArc`, which the docs desribe as

> thread-safe, reference-counted storage for Rust data that can be shared across threads...Rust code and Erlang code can both have references to the same resource at the same time. Rust code uses ResourceArc; in Erlang, a reference to a resource is a kind of term. You can convert back and forth between the two using Encoder and Decoder.

Which, in practical terms, boils down to allowing us to wrap a Rust struct in `ResourceArc` and return that from a NIF. This will be returned to Elixir as an Elixir `Reference` pointing to the Rust data in memory. Then, we can pass that reference back into a NIF and it will be dereferenced to the Rust resource.

So with a few changes, we can get a few steps closer. The full code for this step is available in a Gist [here](https://gist.github.com/mikowitz/8ca55d78ff2929ef4a7bb0cb233edd71), but the relevant excerpts are shown below:

```rust
pub struct CairoWrapper {
  pub context: Context,
  pub surface: ImageSurface
}

type WrapperRA = ResourceArc<CairoWrapper>;

...

#[rustler::nif]
fn save(context: WrapperRA, filename: String) -> WrapperRA {
  ...
}

rustler::init!("Elixir.Xairo.Native", [...], load=on_load);

fn on_load(env: Env, _info: Term) -> bool {
  rustler::resource!(CairoWrapper, env);
  true
}
```

We define a custom struct that wraps our context and image surface (we'll need a direct reference to the surface to save the image, as we saw in the failing example above). Then we create a type alias with our new struct wrapped in a `ResourceArc`, which allows us to pass a resource to the object in memory back and forth between Rust and Elixir. We also change all our function signatures to accept and return our new wrapper type. And finally, we create a function that runs when the NIFs are loaded, and in it call the `rustler::resource!` macro on our struct, which creates the functions necessary to allow `ResourceArc` to wrap and pass our struct.


Over in Elixir, we need to wire this up to the module we've defined as the target, `Xairo.Native`, like so. We just need the function signatures present so we can call them in Elixir, but the actual function bodies should never be called, as we can see by the fact that they all return an error.

```elixir
defmodule Xairo.Native do
  use Rustler, otp_app: :xairo, crate: "xairo"

  def new(_, _), do: :erlang.nif_error(:nif_not_loaded)
  def paint(_, _, _, _), do: :erlang.nif_error(:nif_not_loaded)
  def save(_, _), do: :erlang.nif_error(:nif_not_loaded)
end
```

We run `mix compile`, and it finishes cleanly!

Let's fire up `iex -S mix` and see what we can do.

```iex
iex(1)> i = Xairo.Native.new(100, 100)
#Reference<0.3888755325.2009989122.47088>
iex(4)> Xairo.Native.paint(i, 0.5, 0.0, 1.0)
#Reference<0.3888755325.2009989122.47088>
iex(5)> Xairo.Native.save(i, "test.png")
#Reference<0.3888755325.2009989122.47088>
```

Well, we didn't get any errors, and, as expected because of how rustler's `ResourceArc` works, we see that all these functions return an Elixir reference. In fact, we see they all return the _same_ Elixir reference, because under the hood in Rust (and under that hood, in C), we're modifying a mutable object in memory. This means that, unlike if we were writing pure Elixir code, we don't need to reassign `i` to the result of every operation.[^2]


Because these functions *do* return a reference to the object in memory, we can still pipe them, turning the above IEx commands into

```iex
iex(1)> Xairo.Native.new(100, 100) \
...(1)> |> Xairo.Native.paint(0.5, 0.0, 1.0) \
...(1)> |> Xairo.Native.save("test.png")
#Reference<0.3888755325.2009989122.47088>
```

Again, it all seems to work. But what, if anything, is saved in "test.png?" Let's open it up and find out

![test.png]({{ site.url }}/assets/xairo/test.png)

Just as we'd hoped! A 100x100 pixel purple square, created in an IEx session, and saved to a file!

Seems like we accomplished the goal we set out at the beginning of the post, and with only a quite manageable amount of yak fur to show for it.

Next time, we'll look into ways to make handling resource `Reference` a bit more Elixir-y, and wrap our `Xairo.Native` NIFs in a `Xairo.Image` public API.

Until next time! Thanks for reading!

<hr style="border:1px solid #888888;margin-bottom:15px;"/>
#### Footnotes

[^1]: [Intro to Cairo Graphics in Rust](https://medium.com/@bit101/intro-to-cairo-graphics-in-rust-35470a6aed86), for example, which dives a bit deeper into the specifics of the Rust syntax than I intend to. In fact, this blog post gives us pretty much all the Rust code we need to accomplish our goals for this blog post, so we will be drawing on it heavily for the Rust side of the library.

[^2]: This also means that, technically, we can do some very un-Elixir things with `map` and `for` to modify the image. We can attempt to discourage this sort of behaviour via our final API design, but it will still be possible. Father Valim, full of grace, lead us not into mutable temptation...
