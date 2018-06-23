---
layout: post
title:  On Graphvix - Part 1
subtitle: The over engineering-ing
partno: 1
date:   2018-06-22 19:35:00 -0400
categories: elixir graphviz
---

### The past and present
Around a year and a half ago, when I was first really digging into Elixir, a yak’s worth of nested side projects got me to wanting to implement Graphviz, specifically the `.dot` notation, in Elixir.

I was new to Elixir, steeped in Ruby and object-oriented programming, and excited to test out this new OTP stuff I’d been reading so much about.

Given that, it is perhaps unsurprising that the first version of this code ended up being a GenServer and struct backed, imperative library, with an API that works like this:

```elixir
Graph.new(:new_graph)

{hello_id, _hello} = Node.new("hello")
{goodbye_id, _goodbye} = Node.new("goodbye")

{_edge_id, _edge} = Edge.new(hello_id, goodbye_id, color: "blue")

:ok = Node.update(hello_id, color: "red")
```

A few things worth noting about this implementation:

* The graph state is stored in a GenServer, thus
* The graph structure and contents are opaque to the creator
* Functions that create nodes and edges implicitly attach them to the graph
* There is no feedback to confirm a node has been updated succesfully

Looking at these qualities, it becomes clear that the only reason Elixir is the “right” language to use here is if you really want to use Elixir, not because it is best suited to this API. This runs counter to Elixir’s fundamental notion of functions that accept, modify, and return data. An API that hews closer to this model would

* Store the graph state in active memory, thus
* The creator would have constant access to the contents of the graph without having to retrieve it from an GenServer
* Functions that create nodes and edges would be separate from the functions that connect them to a graph
* Errors returned from those functions would be immediately returned to the user

A re-implementation of the above code in this new API might look something like this:

```elixir
graph = Graph.new(:new_graph)

hello = Node.new("hello")
goodbye = Node.new("goodbye")

{graph, hello} = Graph.add_node(graph, hello)
{graph, goodbye} = Graph.add_node(graph, goodbye)

{graph, edge} = Graph.add_edge(graph, hello, goodbye, color: "blue")

{graph, hello} = Graph.update_node(graph, hello, color: "red")
```

{% include graphvix_series.html %}

### Next steps
As you may have guessed, I have recently returned to this project, with an additional year of Elixir under my belt, and the desire to finally make this project something to use as I slowly unspool the yak wool I’ve been gathering back towards the original side project.

A few months ago, I watched Dave Thomas’ excellent video course on Elixir: https://codestool.coding-gnome.com/courses/take/elixir-for-programmers
I highly recommend it even for intermediate Elixir programmers, as it breaks things down in a way I’ve not seen elsewhere. Over the course of the videos you build a hangman game from a single module that holds a dictionary of valid words all the way up to a single page web app using Phoenix and channels.

One of the most valuable things I took from the course is Dave’s approach to breaking up apps: asking the question “what is the bare minimum this part of the application has to do”, and then extracting that part into its own library. In the hangman game, the dictionary is an application all to itself, which is then a dependency in a library that holds the game rules for hangman, which is then a dependency in a text client implementation, as well as the Phoenix implementation.

Thinking along these lines for Graphvix, it became clear I had done a bit of over engineering. All I need the application to do is provide a way to model a directed graph consisting of nodes, edges, subgraphs (also called clusters), and style attributes for all elements. I don’t need to worry about what happens to that model if Elixir crashes, and indeed perhaps it is presumptuous to force one solution to that on potential users. And, if that’s no longer a requirement, then neither is GenServer.

So where does that leave us for Graphvix v2? Well, we can start with a simple set of structs:

```elixir
defmodule Node do
  defstruct [
    label: "",
    attributes: []
  ]
end

defmodule Edge do
  defstruct [
    from: %Node{},
    to: %Node{},
    attributes: []
  ]
end

defmodule Cluster do
  defstruct [
    label: "",
    nodes: []
  ]
end

defmodule Digraph do
  defstruct [
    nodes: [],
    edges: [],
    subgraphs: [],
    attributes: []
  ]
end
```

and then we can begin to narrow this down, For example,  nodes, edges, and clusters don’t really exist outside of the context of a graph, so we don’t need full structs for them, as they can exist as simple maps inside of the graph data structure.

The final basic piece we need to handle is how to make sure nodes and edges can be reliably connected (and identified as being connected) inside the graph. One simple solution would be to add a unique id to each node or edge as it is added, which would remain constant even if the attributes of the node change throughout the life of the graph. This could be done via random string id generation (UUID, or something similar), or simply by keeping track of a count of nodes/edges, and assigning incrementing numerical ids.

Another, simpler option, is to use Erlang’s `:digraph` library, which, I recently found out, handles modeling the structure of a directed graph. It is not equipped to handle displaying a graph using Graphviz and dot’s syntax, but it provides a functional base several steps beyond reinventing the entire wheel. In a surprising turn of events, `:digraph` uses, for part of its implementation, an imperative style not totally dissimilar to the current version of Graphvix.

So that’s the story of an over-engineered first version, some lessons, and the opening salvo of a more appropriate solution. Next time, I’ll dig into `:digraph` a bit more in depth, and start building on top of it to flesh out the additional functionality we need in Graphvix.

 You can see this code here: [Release v0.5.0 · mikowitz/graphvix · GitHub](https://github.com/mikowitz/graphvix/releases/tag/v0.5.0).

 A simple example of the API as it exists at this point can be found [here](https://github.com/mikowitz/graphvix/blob/v0.5.0/examples/basic.ex), along with the resulting [dot](https://github.com/mikowitz/graphvix/blob/v0.5.0/examples/basic.dot) and [pdf](https://github.com/mikowitz/graphvix/blob/v0.5.0/examples/basic.pdf) files.
