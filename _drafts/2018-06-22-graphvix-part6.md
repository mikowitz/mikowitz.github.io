---
layout: post
title:  On Graphvix - Part 6
subtitle: Subgraphs
partno: 6
date:   2018-06-22 19:35:00 -0400
categories: elixir graphviz
---

### Subgraphs vs Clusters

In DOT, a subgraph is any subset of the defined graph enclosed within `subgraph ID { … }`, or even just `{ … }`. This can be used to specify a common style for a set of nodes or edges (subgraphs can have their own global properties, like the graph as a whole, as we saw in the previous post), or to help group the graph for directional ranking (as we’ll see in a future post).

From the DOT documentation: “A cluster is a subgraph placed in its own distinct rectangle of the layout. A subgraph is recognized as a cluster when its name has the prefix `cluster`”, that is, any subset of the defined graph enclosed within `subgraph cluster<ID> { … }`.

In addition to the global properties for nodes and edges, a cluster can also define a set of layout options for itself, such as a label and border/fill colors.

So in Graphvix, we can model this either as two discrete struct types, or as a single struct type with a flag to determine whether the defined subgraph should be considered a cluster or not. There is enough overlap between subgraphs and clusters that sharing a struct type feels like the correct choice.

### The `Subgraph` struct

```elixir
defmodule Subgraph do
  defstruct [
    id: nil,
    is_cluster: false,
    vertex_ids: [],
    global_properties: [
      node: [],
      edge: []
    ],
    cluster_properties: []
  end
end
```

Nothing should be too surprising here. Like the `Graph` module, we have a nested keyword map for global properties shared between all vertices and edges in the subgraph. There is also a list of `cluster_properties`, but this will only be used if `is_cluster` is set to `true.` We have an `id` field to ensure a unique reference and `DOT` identifier for each subgraph. Finally, there is a list of vertex ids: we will continue to store vertices in the `digraph` ETS table, and reference them in any subgraph.

In addition to the subgraph module, we need to make two changes to the `Graph` module: add a way to store subgraphs created inside the main graph, and keep track of the next subgraph id to ensure unique references.

The first of these changes is straightforward: we can add a `subgraphs: []` entry to our `Graph` struct definition.

The second task is more involved, but the answer already exists in our code. When we looked into the `:digraph` module, we saw that the neighbors table contained two fields tracking the next id to be used for the vertices and the edges:

```elixir
iex> :ets.tab2list(graph.ntab)
[
  {:"$eid", 0},
  {:"$vid", 0},
  ...
]
```

and, in part 3 of this series, [link here], we saw how we can read those values to ensure new vertices have the correct id. Based on that, we can modify our `Graph.new/0` function to insert another entry into that table, an id field that can track our subgraphs:

```elixir
def new do
  digraph = :digraph.new()
  {_, _, _, ntab, _} = digraph
  :ets.insert(ntab, {:"$sid", 0})
  ...
end
```

Then, when we create a new subgraph or cluster, we can pull the current value from that row, and increment the next value in the table, the same as we do when creating a new vertex.

{% include graphvix_series.html %}

### Creating a new subgraph

Creating a new subgraph is fairly straightforward. The function needs to take a fair number of arguments, but all fairly simple:

1. the graph the subgraph belongs to
2. the ids of the vertices included in the subgraph
3. whether the subgraph should be considered a cluster
4. global properties for contained nodes and edges
5. styles for the cluster (if applicable)

It would then return a tuple of two elements, the graph (which has had the subgraph added to its list of subgraphs), and the id of the new subgraph.

### DOT format for subgraphs

This is where things get interesting. Up until this point, our `Graph.to_dot` function has printed all vertices and edges at the top level of the graph. Now that we have introduced subgraphs, this needs to change.

For each subgraph, it needs to be printed containing the definitions of any vertices it contains, as well as any edges which have both their endpoints inside the subgraph. In addition to this, those vertices and edges need to **not** be printed in the top level of the main graph.

#### Vertices

Handling the vertices is the simpler of these two tasks. Each subgraph stores the ids of the vertices, so when printing the subgraph, we can pass in the vertex table from the main graph, find the contained vertices, and print them. The inverse can be done when determining which vertices *should* be printed in the top-level graph: for each vertex, we check whether it is included in any of the graph’s subgraphs, and only print it if it is not.

In the subgraph, we can return a subset of a graph’s vertices whose ids appear in that subgraph:

```elixir
defp vertices_in_this_subgraph(subgraph_vertex_ids, vertices_from_graph) do
  vertices_from_graph
  |> Enum.filter(fn {vid, _attributes} -> vid in subgraph_vertex_ids end)
end
```

And in the graph, we can select all vertices that appear in its subgraphs:

```elixir
defp vertex_ids_in_subgraphs(%__MODULE__{subgraphs: subgraphs}) do
  Enum.reduce(subgraphs, [], fn c, acc ->
    acc ++ c.vertex_ids
  end)
end
```

With these lists constructed, we can use the existing `vertices_to_dot` function and helper functions to print each vertex in `DOT` format in its correct place in the graph.

#### Edges

Determining whether an edge is entirely contained by a single subgraph (that is, the vertices on both ends of the edge are in the same subgraph) is a bit more complex, but follows the same pattern:

For a subgraph, we determine, for each edge in a graph, whether both its ends are in the subgraph in question:

```elixir
def edges_with_both_vertices_in_subgraph(%{vertex_ids: vertex_ids}, graph) do
  [_, etab, _] = Graphvix.Graph.digraph_tables(graph)
  edges = :ets.tab2list(etab)
  Enum.filter(edges, fn {_, vid1, vid2, _} ->
    both_vertices_in_subgraph?(vertex_ids, vid1, vid2)
  end)
end

def both_vertices_in_subgraph?(vertex_ids, vid1, vid2) do
  vid1 in vertex_ids && vid2 in vertex_ids
end
```

And in the main graph, we find all edges which have both their ends in the same one of its subgraphs, using the helper function `Subgraph.both_vertices_in_subgraph?/3` we defined above:

```elixir
def edges_contained_in_subgraphs(graph = {subgraphs: subgraphs}) do
  [_, etab, _] = digraph_tables(graph)
  edges = :ets.tab2list(etab)
  Enum.filter(edges, fn {_, vid1, vid2, _} ->
    Enum.any?(subgraphs, fn %{vertex_ids: vertex_ids} ->
      Subgraph.both_vertices_in_subgraph?(vertex_ids, vid1, vid2)
    end)
  end)
end
```


And, again, we can use the existing `*_to_dot` functions to print out each edge at the correct level in the graph.

And that wraps up looking at subgraphs! Thanks for reading, and stay tuned for the next topic: Record Nodes.
