---
layout: post
title:  On Graphvix - Part 5
subtitle: Global settings
partno: 5
date:   2018-06-22 19:35:00 -0400
categories: elixir graphviz
---

By the  end of the previous post, we had a basic Graphvix API for creating and displaying simple digraphs. Because of the low complexity of the task at hand, we were able to provide these features simply by wrapping the Erlang `:digraph` API in an Elixir shell, and adding some functions to handle the `.dot` output.

But the `:digraph` module was designed to handle more purely mathematical functions on graphs: detecting cycles, finding paths, etc. In order to implement the more advanced, display-centric features of Graphviz, we’re going to need to start building a struct that can store additional data.

{% include graphvix_series.html %}

Let’s take a look first at one of the simpler features: global default display settings. These allow you to apply formatting rules to *all* vertices or edges in a graph. These can then be overwritten on a per-vertex-or-edge basis, but if, say, you want every edge to be colored green, you can define that in one place and it will apply to the entire graph.

It is fairly simple to imagine this as an Elixir struct.

```elixir
defmodule Graph do
  defstruct [
    digraph: :digraph.new(),
    global_defaults: [
      node: [],
      edge: []
    ]
  ]
end
```

I’ve chosen to have a single keyword list named `global_defaults`, with keys for `node` and `edge` nested inside, rather two lists named `node_defaults` and `edge_defaults`. For me, this is to keep the  top-level API of the struct cleaner, and to group like-purposed values more closely together. I also believe this might lead to cleaner, DRY-er code when it comes time to retrieve any values stored there. But this is purely a personal style preference. with no Elixir-based reason to choose it over going with two discrete lists, so don’t take this as the *right* way to do this.[^1]

Fortunately, this results in very few changes to the `Graphvix` API that already exists. Even though, due to the internal workings of `:digraph`, the state of the `digraph` record does not change when elements are added, I chose to write the Elixir API as if it did, meaning that any function that caused a change to the graph returned the new (really the same) graph. Now that we’re moving to using a struct, this matters, but it’s already been taken care of!

The only other place the code had to change was in destructuring in order to retrieve the ETS table references from the digraph. I wrote a helper function `digraph_tables` which took a `digraph` record and returned a list of the table references:

```elixir
def digraph_tables({:digraph, vtab, etab, ntab, _}) do
  [vtab, etab, ntab]
end
```

In the code, I would use it like this:

```elixir
defp vertices_to_dot(graph) do
  [vtab, _, _] = digraph_tables(graph)
  ...
end
```

Now that the `graph` passed to the function is our struct, this code needs to change, but only slightly:

```elixir
defp vertices_to_dot(graph) do
  [vtab, _, _] = digraph_tables(graph.digraph)
  ...
end
```

Now that the existing code has been updated to use the new struct (and all the tests are passing!), we can turn our attention to the functions we need to store and write these global attributes:

```elixir
def set_global_property(graph, attr_for, [{key, value}]) do
  properties = Keyword.get(graph.global_properties, attr_for)
  new_props = Keyword.put(properties, key, value)
  new_properties = Keyword.put(graph.global_properties, attr_for, new_props)
  %{ graph | global_properties: new_properties }
  end
end

def set_global_properties(graph, attr_for, attributes \\ []) do
  Enum.reduce(attributes, graph, fn {key, value}, graph_acc ->
    set_property(graph_acc, attr_for, {key, value})
  end)
end

iex> graph = Graph.set_global_property(graph, :node, color: :blue)
%Graph{...}
iex> graph = Graph.set_global_properties(graph, :edge, color: :red, style: "dotted")
```

The final function left to write is the `global_properties_to_dot/1` function, which can make use of the same helper functions used to print out attributes for vertices and edges:

```elixir
defp global_properties_to_dot(graph) do
  global_props = [
    _global_properties_to_dot(graph, :node),
    _global_properties_to_dot(graph, :edge)
  ] |> Enum.reject(&is_nil/1)

  case length(global_props) do
    0 -> nil
    _ -> Enum.join(global_props, "\n")
  end
end

defp _global_properties_to_dot(%{global_properties: global_props}, key) do
  with props <- Keyword.get(global_props, key) do
    case length(props) do
      0 -> nil
      _ -> "  #{key} #{attributes_to_dot(props)}"
    end
  end
end
```

This function is then called in our top-level `Graph.to_dot/1` function, adding these properties to the graph before formatting our nodes and edges.

Graphvix code adding support for global properties: [Add global properties · mikowitz/graphvix@7700e37 · GitHub](https://github.com/mikowitz/graphvix/commit/7700e37dfb00d17c86b20e883fc030ed1bb40e90)

And that’s pretty much all there is to global settings for a Graphviz graph. Thanks for reading, and come back next time when we look at grouping vertices into subgraphs.

[^1]: This could apply to most content on this blog. I’m not, currently, writing tools in Elixir that are likely to see serious production usage, so I allow code layout aesthetics to guide my coding as much as, if not more than, performance concerns. Here ends the disclaimer.

