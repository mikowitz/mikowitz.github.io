---
layout: post
title:  On Graphvix - Part 4
subtitle: A first API
partno: 4
date:   2018-06-22 19:35:00 -0400
categories: elixir graphviz
---

At its simplest, what do we need `Graphvix` to accomplish	for us?

1. Create a new graph
2. Add a vertex
3. Add an edge
4. Convert our graph to `.dot` format
5. Write that `.dot` format to a file
6. Compile that file and display the graph

Fortunately, these instructions translate 1-to-1 into discrete Elixir functions:

1. `Graph.new/0`
2. `Graph.add_vertex/3`
3. `Graph.add_edge/4`
4. `Graph.to_dot/1`
5. `Graph.write/2`
6. `Graph.show/2`

Plugging these into an imaginary IEx session , we can see how these should behave:

```elixir
iex> graph = Graph.new
:digraph, #Reference<..>, #Reference<...>, #Reference<..>, true}

iex> {graph, vid1} = Graph.add_vertex(graph, "hello", color: "green")
{ {:digraph, ...}, "v0"}
iex> {graph, vid2} = Graph.add_vertex(graph, "goodbye", color: "red")
{ {:digraph, ...}, "v1"}

iex> {graph, _eid} = Graph.add_edge(graph, vid1, vid2, color: "blue")
{ {:digraph, ...}, "e0"}

iex> Graph.to_dot(graph)
"""
digraph G {

  v0 [label="hello",color="green"];
  v1 [label="goodbye",color="red"];

  v0 -> v1 [color="blue"];
}
"""

iex> Graph.write(graph, "test_graph")
:ok #=> creates "test_graph.dot" in the current directory

iex> Graph.show(graph, "test_graph")
:ok #=> creates "test_graph.dot" in the current directory, and, assuming the `dot` binary is installed, compiles "test_graph.dot" into a `.png` and opens it using the system's default image viewer
```

Note that instead of creating a struct to wrap our digraph record, these functions accept and return the digraph record itself. There is nothing in this API that requires storing data that cannot be stored in the three ETS tables provided by the record, so there’s no need to over-engineer a struct-based solution.

As we continue to expand the functionality of this library, we will eventually need to store data that has no place inside the existing digraph record and its attached ETS tables. In order to make that a smoother transition, I’ve designed the API here to return the digraph record from functions (even though, as we saw in earlier posts, the record itself doesn’t change) so that when we replace that record with a stateful struct, the API is already prepared to handle it.

And, if we run this code, we end up with a nice, little “test_graph.png” that looks like this:

[image:99C58299-36D2-406C-9DA3-B499992D26A8-449-0000A8F498C8D098/23A4B0C6-637F-468E-99E6-C50F9DB7CC43.png]


Except for the `add_vertex` code we looked at in the previous post [add link here], there’s very little in the code implementing this API that is particularly noteworthy. One piece I do want to explore briefly is a bit of fancy footwork to reduce duplication between the code that generates `DOT` formatted vertices and edges:

```elixir
defp nodes_to_dot(graph) do
  [vtab, _, _] = digraph_tables(graph)
  elements_to_dot(vtab, fn {[_ | id], attributes} ->
    "  v#{id} #{attributes_to_dot(attributes)}"
  end)
end

defp edges_to_dot(graph) do
  [_, etab, _] = digraph_tables(graph)
  elements_to_dot(etab, fn {_, [:"$v" | v1], [:"$v" | v2], attributes} ->
    "  v#{v1} -> v#{v2} #{attributes_to_dot(attributes)}"
  end)
end
```

### Caveat

Before we dive into the meat of this post, there’s one piece of unfortunate terminology trouble we need to get out of the way. The erlang `digraph` module, as we’ve seen, uses the term “vertex”, but the `.dot` syntax uses the word “node”. Both terms represent the same concept, and will always be interchangeable. However, it’s not ideal to have to switch back and forth often between the terms. For two reasons, I’m going to do my best to use “vertex” in all places that are **not** specifically referring to `.dot` syntax:

1. The existing `digraph` code uses “vertex” and “v”, and so I would prefer to maintain terminology with existing code
2. `node` is itself a concept in distributed Erlang and Elixir, so I’d like to keep my own code from overlapping[^1]

### Back to it

In both `nodes_to_dot` and `edges_to_dot` we destructure our digraph tables to find the relevant ETS table. Then, in each case, we pass that table, along with a formatting function, to another function named `elements_to_dot`. We can do this because, in Elixir, functions are first-class citizens, meaning they can be passed as parameters to other functions the same as more basic types like numbers and strings. This allows us to use the same code to do some of the duplicated work for us. Otherwise, each of our functions above would need to

* use ETS to get a list of rows from the table
* sort the elements by their ID
* map our formatting function over them
* join the results

But because the only difference would be the exact work done in that formatting function, we can separate that out, and use a single, shared function to handle the identical work:

```elixir
defp elements_to_dot(table, formatting_func) do
  case :ets.tab2list(table) do
    [] -> nil
    elements ->
      elements
      |> Enum.sort_by(fn entry ->
        [[_ | id] | _] = Tuple.to_list(entry)
        id
      end)
      |> Enum.map(&formatting_func.(&1))
      |> Enum.join("\n")
  end
end
```

In this example, we see two ways of passing a function into another, higher-order function. For `Enum.sort_by`, we pass in an anonymous function, of the form `fn [args] -> function_body end`, to ensure our vertices or edges are printed out in the same order we added them to the graph. For `Enum.map` we pass in our `formatting_func` using the Elixir shorthand of prepending `&` to a named function to invoke it.

With named functions like this, calling it is done by `func_name.(…)` syntax (note the period between the function name and the parentheses containing the arguments.) When using the `&` shorthand, `&1`, `&2`, etc. are available inside the function body as shorthand for the passed-in parameters (first, second, etc, respectively). That entire line is equivalent to

```elixir
Enum.map(elements, fn element -> formatting_func.(element) end)
```

As you can see, the shorthand is, as you would hope, more concise. And, with practice, becomes as easy to read and understand as the longhand version.

### Wrapping up

The Graphvix code containing the implementation of this API can be found here: [Graphvix v1.0.0.pre](https://github.com/mikowitz/graphvix/tree/v1.0.0.pre)

I’m going to leave off here. Explore the code (what there is of it) at your leisure, and do let me know if you have any questions. Now that we have a *very* basic graph implementation working, the next steps are to move on to more advanced `.dot` options:

1. global default settings for the main graph and subgraphs
2.  subgraphs (also called clusters)
3.  vertices with a shape of “record”, allowing nested rows and columns in a single vertex, as well as multiple edge ports
4. the ability to set ranks for sets of vertices, enforcing a vertical or horizontal hierarchy on the graph as a whole


[^1]: Also, it makes the syntax highlighting in my code weird, since it interprets `node` as an Elixir keyword.

