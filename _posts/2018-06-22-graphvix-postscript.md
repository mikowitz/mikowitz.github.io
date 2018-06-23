---
layout: post
title:  On Graphvix - Postscript
subtitle: HTML Records
partno: 100
date:   2018-06-22 19:35:00 -0400
categories: elixir graphviz
---

I had intended to end this blog series with the previous post, but my own personal yak-shaving led to wanting to add one more feature (for now) to `Graphvix`, and so I wanted to write it up for any readers I might have, and also for myself. This feature is HTML Records, which allow the user to create table-style records (as we saw in parts 7 and 8) using HTML `<table>` markup.

To start off, let’s take a look at what a record like this might look like in `DOT` notation:

```elixir
html_record [label=<<table>
  <tr>
    <td>a</td>
    <td>b</td>
  </tr>
  <tr>
    <td>c</td>
    <td>d</td>
  </tr>
</table>>,style="plaintext"]
```

And here’s what that looks like

[image:D9CB6525-C87F-478B-9E05-549163E1C872-46251-000092B2CD67279D/9004C40A-2991-40EE-8EC6-A16945DDD191.png]

And something a little more complicated:

```elixir
html_record [label=<<table border="0" cellborder="1" cellspacing="0">
  <tr>
    <td rowspan="3" color="blue"><font point-size="24" color="#ff0000">hello</font><br/>world</td>
    <td colspan="3">b</td>
    <td rowspan="3" border="3">g</td>
    <td rowspan="3">h</td>
  </tr>
  <tr>
    <td>c</td>
    <td port="here">d</td>
    <td bgcolor="green">e</td>
  </tr>
  <tr>
    <td colspan="3" cellpadding="10">f</td>
  </tr>
</table>>,shape="plaintext"]
```

which produces the resulting output.

[image:69AF3F14-1118-4E76-9AD6-E33B5D7513FA-46251-000093341B8BE01A/AAEDF023-4732-413D-8D03-2332A4A76F65.png]

You can go even further here. It is possible to nest tables inside one another, to change font, to explicitly set text alignment, to have background colors form a gradient, and more. However, let us begin by writing an API that will let us generate the first, simple table shown above. Once we have done that, much like more generic nodes and edges in earlier posts in this series, we will see that creating more complex examples requires little more than passing additional parameters to set as attributes, not a large expansion of the API.

## Graphvix API

Since this is a different type of record than the row and column based ones we’ve looked at before, it makes sense to begin by writing a new module to contain the logic we need:

```elixir
defmodule Graphvix.HTMLRecord do
  defstruct [
    rows: [],
    attributes: []
  ]

  def new(rows, attributes \\ []) when is_list(rows) do
    %__MODULE__{rows: rows, attributes: attributes}
  end
end
```

Simple enough. And no harder to start building the API for adding rows and cells:

```elixir
defmodule Graphvix.HTMLRecord do
  # in the Graphviz HTML record notation `tr`s cannot have attributes,
  # hence the lack of a keyword list here
  def tr(cells) when is_list(cells) do
    %{cells: cells}
  end

  def td(label, attributes \\ []) do
    %{label: label, attributes: attributes}
  end
end
```

Great! Now we can build out a full `HTMLRecord`. Let’s start by creating our first, simple example from the beginning of the post:

```elixir
alias Graphvix.HTMLRecord
import HTMLRecord, only: [tr: 1, td: 1, td: 2]

record = HTMLRecord.new([
  tr([
    td("a"),
    td("b")
  ]),
  tr([
    td("c"),
    td("d")
  ])
])
```

And it turns out, in order to implement our more complex example using this API, we only need to add two more functions, `font/2` and `br/0`

```elixir
defmodule Graphvix.HTMLRecord do
  ...
  # We'll add the `tag: "font"` element here to differentiate
  # from a `td` tag
  def font(label, attributes \\ []) do
    %{label: label, attributes: attributes, tag: "font"}
  end
  def br do
    %{tag: "br"}
  end
end
```

And then, in Elixir:
```elixir
HTMLRecord.new([
  tr([
    td([
      font("hello", point_size: 24, color: "#ff0000"),
      br(),
      "world"
    ], rowspan: 3, color: "blue"),
    td("b", colspan: 3),
    td("g", rowspan: 3, border: 3),
    td("h", rowspan: 3)
  ]),
  tr([
    td("c"),
    td("d", port: "here"),
    td("e", bgcolor: "green")
  ]),
  tr([
    td("f", colspan: 3, cellpadding: 10)
  ])
], border: 0, cellborder: 1, cellspacing: 0)

```

## `.to_label/1`

Now that we have our API worked out, we need one final piece: converting it all to proper `.dot` notation. As with the records we looked at previously, our task is to convert an `HTMLRecord` into label text and attributes that can be fed into our existing functions for adding vertices to a graph. Since a table is a series of `tr` elements, which are each themselves a series of `td` elements, we can sketch out the basic shape of our `.to_label` function and its helper functions:

```elixir
defmodule Graphvix.HTMLRecord do
  ...
  def to_label(%__MODULE__{rows: rows, attributes: attributes}) do
    # convert all the rows to their string representation, and
    # wrap those in a `<table>` declaration + attributes
  end

  defp tr_to_label(%{cells: cells}) do
    # convert each cell to its string representation, and
    # wrap those in a `<tr>` declaration
  end

  defp td_to_label(%{label: label, attributes: attributes}) do
    # convert the label to its string representation, and
    # wrap it in a `<td>` declaration + attributes
  end

  defp attributes_for_label(attributes) do
    # helper method to convert a Keyword list of attributes
    # into a properly formatted string.
  end
end
```

Now let’s flesh these out. Because each function depends on those defined after it, let’s start at the bottom with the smallest piece.

### `attributes_for_label/1`

This function takes a keyword list of attribute names and values, and returns them in a string formatted as HTML tag attributes:

 `key1=“value1” key2=“value2”`

There are two possibilities for the argument `attributes` passed to this function

1. it is an empty list, in which case we want to return an empty string

```elixir
defp attributes_for_label([]), do: ""
```

2. it is not an empty list, in which case we need to map over the key/value pairs, format them, and join them together.

```elixir
defp attributes_for_label(attributes) do
  attr_str = attributes
  		|> Enum.map(fn {key, value} -> ~s(#{key}="#{value}") end)
      |> Enum.join(" ")
  " " <> attr_string
end
```

If you’re not familiar with it `~s()`, also called `sigil_s` is a function that returns a string, allowing interpolation, but providing an alternative to `"` or `'` as the delimiters, allowing those characters to be used inside the string without escaping them.

You’ll also notice that we return the attribute string with an empty space prepended. Since we will be interpolating the return value of this function into a string describing an HTML tag, we want there to be a space between the tag and the attributes if there are any attributes, but no space between the tag and the closing `>` otherwise. A quick example may illustrate this better than words:

```elixir
"<table#{attributes_for_label()}>" #=> <table>
"<table#{attributes_for_label(border: 0)}>" #=> <table border="0">
```

Next we move on to the next function up the hierarchy:

### `td_to_label/1`

The basic shape of this function, as we defined it above, is straightforward:

```elixir
def td_to_label(%{label: label, attributes: attributes}) do
  [
    "<td#{attributes_for_label(attributes)}>",
    label_to_string(label),
    "</td>"
  ] |> Enum.join("") |> indent()
end
```

`indent/1` is a small helper function I wrote to, well,  indent a string (or indent each line in a multiline string).[1] While this is not necessary, it provides cleaner output that is easier to parse, and looks much nicer when written to a file.

The trickiest part here is in `label_to_string/1`, since it the `label` argument can be one of

1. a string
2. a list of mixed elements (string, `<br>` tags, and `<font` tags)
3. a whole other `HTMLRecord` struct

We will return to `label_to_string/1`	 shortly, but first let’s finish working our way up the function chain until we reach the top-level `HTMLRecord.to_label/1`. With that in mind, onward to:

### `tr_to_label/1`

This is possibly the simplest of the functions we need to write here. Since `<tr>` tags do not have attributes, and every `<td>` tag it contains is generated by `td_to_label/1`, the entire function definition looks like this:

```elixir
def tr_to_label(%{cells: cells}) do
  [
    "<tr>",
    Enum.map(cells, &td_to_label/1),
    "</tr>"
  ] |> List.flatten |> Enum.join("\n")
```

Since `td_to_label/1` calls indent on the `<td>` tag it generates, the output from this function would look something like:

```elixir
<tr>
  <td>a</td>
  <td>b</td>
</tr>
```

And that’s all we need to write our top-level `HTMLRecord.to_label/1` function. Let’s take a look:

### `to_label/1`

```elixir
def to_label(%{rows: rows, attributes: attributes}) do
  [
    "<<table#{attributes_for_label(attributes)}>",
    Enum.map(rows, fn row -> row |> tr_to_row() |> indent() end),
    "</table>>"
  ] |> List.flatten |> Enum.join("\n")
end
```

Because `indent/1` knows how to indent each line of a multiline string, we can pass the entire string value of a `<tr>` tag to it and know that each enclosed `<td>` tag will also be indented another space. A simple example might look something like this:
```elixir
<<table>
  <tr>
    <td>a</td>
    <td>b</td>
  </tr>
  <tr>
    <td>c</td>
    <td>d</td>
  </tr>
</table>>
```

which you might recognize as being the label value for our first example table at the beginning of this post. Look at that!

You may have noticed that the entire table’s HTML is wrapped in a pair of `<` and `>`. This is part of the `.dot` syntax, wrapping an HTML label in brackets instead of double quotes.

With that in hand, let’s jump back to `label_to_string/1`

### `label_to_string/1`

To refresh, the `label` we pass to this function can be one of

1. a string
2. a list of mixed elements (string, `<br>` tags, and `<font` tags)
3. an `HTMLRecord` struct

The first case is the simplest to write. If it’s already a string, just return the string unmodified
`def label_to_string(str) when is_bitstring(str), do: str`

If we receive a list, we just want to iterate recursively over each element, until each element is itself a string, and then we can join them and move on

```elixir
def label_to_string(list) when is_list(list) do
  Enum.map(list, &label_to_string/1) |> Enum.join("")
end
```

This requires us to define `label_to_string` for `<br/>` and `<font>` tags. Remember we wrote helper functions above to generate these, in which we added a `tag` key to the map. We can use that now to help pattern match these cases

```elixir
def label_to_string(%{tag: "br"}), do: "<br/>"
def label_to_string(%{tag: "font", label: label, attributes: attributes}) do
  [
    "<font#{attributes_for_label(attributes)}>",
    label_to_string(label),
    "</font>"
   ] |> Enum.join("")
end
```

Remember, the label we pass to a font tag could be another list of smaller elements.

Finally, if we’re nesting an entire table inside a cell, we can just reuse `HTMLRecord.to_label/1`, can’t we?

```elixir
def label_to_string(table = %HTMLRecord{}) do
  HTMLRecord.to_label(table)
end
```

Not quite! Remember that `to_label` currently wraps the entire table in an extra pair of `<...>`, which we don’t want in this case. I won’t go through an entire code example, but suffice it to say this results in invalid syntax, which results in not displaying the node at all. This is easy enough to fix by using the trick we used previously to determine if a row in a record was at the top level of the label. This requires just a small change to `to_label/1`:

```elixir
def to_label(%{rows: rows, attributes: attributes}, top_level \\ true) do
  {extra_opening_tag, extra_closing_tag} = case top_level do
    true -> {"<", ">"}
    false -> {"", ""}
  end
  [
    "#{extra_opening_tag}<table#{attributes_for_label(attributes)}>",
    Enum.map(rows, fn row -> row |> tr_to_row() |> indent() end),
    "</table>#{extra_closing_tag}"
  ] |> List.flatten |> Enum.join("\n")
end
```

and a small change to `label_to_string/1`

```elixir
def label_to_string(table = %HTMLRecord{}) do
  HTMLRecord.to_label(table, false)
end
```

Now there is only one piece left: adding an `HTMLRecord` to our graph.

## `Graphvix.Graph.add_html_record/2`

After all the buildup, this turns out to be a rather simple function:

```elixir
defmodule Graphvix.Graph do
  ...
  def add_html_record(graph, record) do
    label = HTMLRecord.to_label(record)
    attributes = [shape: "plaintext"]
    add_vertex(graph, label, attributes)
  end
end
```

We take the `record`, generate the label, and pass that along to our existing `add_vertex/3` function. The only additional work we have to do is set the `shape` attribute of the node to `plaintext`. This keeps `DOT` from using  the default oval shape and surrounding the table with an oval. Other than that, all other styling attributes are included in the generated HTML.

There’s only one problem left. Our `vertices_to_dot/1` function passes the vertex’s label through `attributes_to_dot/1`, which unconditionally wraps every attribute value in a pair of double quotes. Currently, that includes our HTML label, which is already wrapped in `<...>`. If we generate a .dot file as is and compile, we end up with a node that looks something like this

[image:394BB1D9-BDA6-40F4-B125-B0C916EF0689-46251-00009B5CA64E7F3A/2E174293-E461-4DD4-BB96-36112C65B89E.png]

Which is decidedly not what we want. Fortunately, we can add a second pattern match to our `attribute_to_dot/2` function which looks for this case and does **not** include the double quotes:

```elixir
def attribute_to_dot(:label, value = "<<table" <> _) do
  ~s(label=#{value})
end
# our existing, default implementation
def attribute_to_dot(key, value) do
  ~s(#{key}="#{value}")
end
```

Recompiling, we get a node that looks much more like what we were hoping for:

[image:3FBAD9CF-2FA2-4A03-BC69-4CE9449A1F3F-46251-00009B7981C21D19/181CB9C9-505F-40FB-8EAD-60A2380F724D.png]

## Edges and ports

As we saw in an example above, giving a cell a port is as simple as passing it is as one of the attributes for `td/2`. The function `add_html_record/2` returns the id of the vertex, along with the graph, so that in conjunction with any port names, can be used to draw edges to and from an HTML table node the same as any other type of node.

## Wrapping up

Ok, that was a lot! Thanks for sticking through it with me.

With this post, my series on `Graphvix` is truly coming to a close. I won’t say it’s the last time I’ll write about this library, as there is additional functionality that could be added, but for now, I have a solid, 1.0.0-able version of the library.

If you’ve enjoyed this post, or this series more generally, please share it with your friends who might be interested in Elixir or Graphviz. And follow me on Twitter to keep up to date with future blogging.

