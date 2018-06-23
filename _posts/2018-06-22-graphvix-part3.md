---
layout: post
title:  On Graphvix - Part 3
subtitle: IDs
partno: 3
date:   2018-06-22 19:35:00 -0400
categories: elixir graphviz
---

As mentioned at the end of the last post, the order of entry for vertices in a `.dot` source file can affect the generated output. In this post we’re going to dig a bit deeper into how `:digraph` keeps track of its contents, and how we can use that system to ensure that the graph we end up with looks the way we expect it to.

Let’s start with a new digraph:

```elixir
iex> graph = {_, vtab, etab, ntab, _} = :digraph.new()
iex> :ets.tab2list(vtab)
[]
iex> :ets.tab2list(etab)
[]
iex> :ets.tab2list(ntab)
["$eid": 0, "$vid": 0]

```

As expected, the vertex and edge tables are empty, but the neighbors table has these two “$eid” and “$vid” keys, both with a value of zero. Perhaps unsurprisingly, these represent “edge id” and “vertex id”. Let’s see them in action:

```elixir
iex> v0 = :digraph.add_vertex(graph, "Boston")
iex> v1 = :digraph.add_vertex(graph, "Baltimore")
iex> :digraph.add_edge(graph, v0, v1)
iex> :ets.tab2list(ntab)
[
  { {:out, "Boston"}, [:"$e" | 0]},
  {:"$eid", 1},
  {:"$vid", 0},
  { {:in, "Baltimore"}, [:"$e" | 0]}
]
```

So, we see that the edge connecting Boston to Baltimore has the id `[:”$e” | 0]`, and the value for “$eid” has been incremented to 1. But despite adding two vertices, the value of “$vid” is still 0.

To find out why, let’s take a look at the spec for `:digraph.add_vertex`. It has 3 possible arities: 1, 2, and 3. We’ve seen arities `2` and `3` already, but what about `:digraph.add_vertex/1`?

```elixir
iex> v2 = :digraph.add_vertex(graph)
[:"$v" | 0]
iex> :ets.tab2list(vtab)
[{"Baltimore", []}, {"Boston", []}, {[:"$v" | 0], []}]
```

If we don’t pass any information about the vertex we want to add, `:digraph` will pull the current “$vid” value out of the neighbor’s table and assign it to the vertex, much as it does with edges. We can subsequently see that the 	`$vid` value in the neighbors table is updated accordingly:

```elixir
iex> :ets.tab2list(ntab)
[
  { {:out, "Boston"}, [:"$e" | 0]},
  {:"$vid", 1},
  {:"$eid", 1},
  { {:in, "Baltimore"}, [:"$e" | 0]}
]
```

But we want to be able to pass information in when we create a vertex! We’ve reached a point where we need to look beyond the existing `:digraph` API and have to start extending our own. Let’s take a look at what this function will need to do:

1. take in the `graph`, the vertex name, and its attributes
2. pull the current `$vid` value from the graph’s neighbor’s table
3. insert an ETS row using that id value and the name and attributes passed in
4. increment the `$vid` in the neighbor’s table
5. return the id of the vertex so we can find it later (similarly to how `:graphvix.insert_edge` returns the id value for the edge)

```elixir
def insert_vertex(graph = {:digraph, _, _, ntab, _}, name, attributes \\ []) do
  [{:"$vid", next_id}] = :ets.lookup(ntab, :"$vid")
  true = :ets.delete(ntab, :"$vid")
  true = :ets.insert(ntab, {:"$vid", next_id + 1})
  attributes = Keyword.put(attributes, :name, name)
  vertex_id = [:"$v" | next_id]
  :digraph.add_vertex(graph, vertex_id, attributes)
end
```

The code around getting and setting the `$vid` value is directly from the Erlang `:digraph` source code. The Erlang function that handles this is not exported with the module, so we cannot call it directly. The code can be found [here](https://github.com/erlang/otp/blob/master/lib/stdlib/src/digraph.erl#L357) if you are curious.

Assuming we have included this in a module named `Graphvix`, let’s see what this code does:

```elixir
iex> Graphvix.insert_vertex(graph, "first tab", color: "blue")
[:"$v" | 1]
iex> :ets.tab2list(vtab)
[
  {"Baltimore", []},
  {"Boston", []},
  {[:"$v" | 0], []},
  {[:"$v" | 1], [color: "blue", name: "first tab"]}
]
iex> :ets.tab2list(ntab)
[
  { {:out, "Boston"}, [:"$e" | 0]},
  {:"$vid", 2},
  {:"$eid", 1},
  { {:in, "Baltimore"}, [:"$e" | 0]}
]
```

Look at that! Now, when we go back to generate a `.dot` source file from the digraph record, we can sort the vertices by that initial id to ensure that the order we have entered them into the graph is the same order they appear in the source file.

For the sake of completeness, let’s confirm that we can use the existing `:digraph` API to similarly add attribute fields to edges and get the properly ordered ids:

```elixir
iex> :digraph.add_edge(graph, v1, v2, color: :blue, style: :dotted)
[:"$e" | 1]
iex> :ets.tab2list(etab)
[
  {[:"$e" | 1], [:"$v" | 1], [:"$v" | 0], [color: :blue, style: :dotted]},
  {[:"$e" | 0], "Boston", "Baltimore", []}
]
```

The more recently added edge may appear first in the table, but, like vertices, we will be able to sort these results by the id when it comes time to add them to the `.dot` source file.

In the code above, I’ve glossed over some of the details regarding the `:digraph` and `:ets`  APIs. My hope is that my use of them here is self explanatory, but if you would like to dig deeper into what these modules can do — and ETS in particular is very powerful — you can find their source code and documentation by following these links.

* ETS
	* [source code](https://github.com/erlang/otp/blob/master/lib/stdlib/src/ets.erl)
	* [documentation](http://erlang.org/doc/man/ets.html)
* digraph
	* [source code](https://github.com/erlang/otp/blob/master/lib/stdlib/src/digraph.erl)
	* [documentation](http://erlang.org/doc/man/digraph.html)


Thanks for reading! Come back next time when we start plugging what we’ve built here into an Elixir module and design an API to allow building *and* generating full graphs.
