---
layout: post
title:  On Graphvix - Part 2
subtitle: A :digraph primer
partno: 2
date:   2018-06-22 19:35:00 -0400
categories: elixir graphviz
---

My goal for this post is not to provide a deep dive into the implementation of Erlang’s `digraph` module. Other blog posts do this, and do this well, so here I want to focus on the basics, and what is relevant for reimplementing `Graphvix`.

{% include graphvix_series.html %}

So let’s start with a brand new `digraph`

```elixir
iex> :digraph.new
{:digraph, #Reference<0.1293054518.1754923012.255883>,
 #Reference<0.1293054518.1754923012.255884>,
 #Reference<0.1293054518.1754923012.255885>, true}
```

Moving through this result tuple quickly, the first element is the Erlang record type; the three `#Reference<0.xxxxx.xxxx.xxxxx>` elements refer to `ets` (Erlang Term Storage, in-memory storage) tables Erlang uses internally to manage the graph data; and the `true` is a field that stores the boolean value of whether the graph is cyclic or not. For our purposes, this final field is irrelevant.

The three ETS tables hold, respectively
* the graph’s vertices (nodes)
* the graph’s edges - uniquely identified pairs of connected nodes
* the graph’s neighbors - a table of node-to-edge connections, include the direction of the edge to/from the node in question.

Let’s see this in practice in a small example

```elixir
## Create a new graph
iex> graph = :digraph.new

## Add vertices and edges
iex> boston = :digraph.add_vertex(graph, "Boston")
iex> baltimore = :digraph.add_vertex(graph, "Baltimore")
iex> paris = :digraph.add_vertex(graph, "Paris")
iex> :digraph.add_edge(graph, boston, paris)
iex> :digraph.add_edge(graph, baltimore, paris)

## Destructure the graph record to make accessing the tables easier
iex> {_, vertices, edges, neighbors, _} = graph

## Examine the tables
iex> :ets.tab2list(vertices)
[{"Paris", []}, {"Boston", []}, {"Baltimore", []}]

iex> :ets.tab2list(edges)
[
  {[:"$e" | 0], "Boston", "Paris", []},
  {[:"$e" | 1], "Baltimore", "Paris", []}
]

iex> :ets.tab2list(neighbors)
[
  {:"$eid", 2},
  {:"$vid", 0},
  { {:in, "Paris"}, [:"$e" | 1]},
  { {:out, "Baltimore"}, [:"$e" | 0]},
  { {:out, "Boston"}, [:"$e" | 1]},
  { {:in, "Paris"}, [:"$e" | 0]}
]
```

You can safely ignore the `:”$eid”` and `:$”vid”` keys in the neighbors tab for now, as well as the `[:”$e” | 0]` style unique identifiers in the edges table. We will revisit these in the next blog post.

We can immediately see that, like Elixir maps, input order is no indication of the order ETS will return the vertices and edges, but we’re still well on our way to having a working graph implementation that meets our needs:

### What Graphvix can use from this

* graph structure/record
* an API to add vertices and edges
* ability to reconstruct vertices and their edges for outputting in dot format

### What Graphvix still needs

* ability to store formatting and styles for vertices and edges
* ability to output to dot format
* ability to sort vertices in the order they were added to the graph[1]
* higher-level `dot` functionality: subgraphs/clusters, vertex ranking, etc.

We can get that first item, persistent formatting data, as part of the `digraph` and its ETS tables without needing to add any module or struct wrapping, so let’s start with that.

### Formatting

Let’s look again at the table contents for vertices and edges:

```elixir
iex> :ets.tab2list(vertices)
[{"Paris", []}, {"Boston", []}, {"Baltimore", []}]

iex> :ets.tab2list(edges)
[
  {[:"$e" | 0], "Boston", "Paris", []},
  {[:"$e" | 1], "Baltimore", "Paris", []}
]
```

Look at that empty list, `[ ]` , as the last element in each vertex or edge tuple. ETS is freeform, so you can store whatever data you want. `:digraph`  chooses to add that empty list to hold what it refers to in the [source code](https://github.com/erlang/otp/blob/master/lib/stdlib/src/digraph.erl) as a label. But the spec for the custom `label` type is the catchall `term()`, so we can put anything we want in there, such as a keyword list of Graphviz attributes:

```elixir
iex> :digraph.add_vertex(
...>   graph, "Boston", shape: "box", color: "blue", style: "filled"
...> )
iex> :ets.tab2list(vertices)
[
  {"Boston", [shape: "box", color: "blue", style: "filled"]}
  ...
]
```

This behavior holds the same for edges as well. This means that, provided a unique id for each vertex, we can, without needing to (yet) create a wrapping module, provide a storage mechanism for any formatting data (or, any data) we want to attach to a vertex or edge.

That’s it for now! Next time we’ll dig a little deeper into those odd identifiers in the ETS tables, and how we can use them to ensure the generated graph looks exactly the way you expect it to.[1]


[1] The order in which nodes are defined in the `.dot` source file can affect the final layout of the generated graph. While this is often offset by specifications with subgraphs or vertex ranking (which we’ll tackle in future posts), this is not a guarantee, and vertices appearing in the source file in a different order than you intended them can result in difficult to track formatting errors.


