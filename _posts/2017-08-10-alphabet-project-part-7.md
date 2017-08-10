---
layout: post
title:  Alphabet Project, Part 7
date:   2017-08-10 08:16:00 -0400
categories: elixir music processing
---

1. [Alphabet Project, Part 1]({{ site.baseurl }}{% post_url 2017-07-20-alphabet-project-part-1 %})
1. [Alphabet Project, Part 2]({{ site.baseurl }}{% post_url 2017-07-22-alphabet-project-part-2 %})
1. [Alphabet Project, Part 3]({{ site.baseurl }}{% post_url 2017-07-23-alphabet-project-part-3 %})
1. [Alphabet Project, Part 4]({{ site.baseurl }}{% post_url 2017-07-27-alphabet-project-part-4 %})
1. [Alphabet Project, Part 5]({{ site.baseurl }}{% post_url 2017-08-01-alphabet-project-part-5 %})
1. [Alphabet Project, Part 6]({{ site.baseurl }}{% post_url 2017-08-06-alphabet-project-part-6 %})

### Refactoring Elixir - structuring measures

Looking at the [full version 1](https://github.com/mikowitz/alphabet_project/tree/v1) of the code I've written over the past
6 blog posts, there is a lot of refactoring to be done. A lot of it falls under the category of naming cleanup and
extracting common code -- especially around writing out to LilyPond files -- which is all self-explanatory enough not to
necessitate going into any further detail. There is one aspect of the project that I found frustrating even during
the original development, but by the time I began to feel hampered by it, the drive to complete a working version
was stronger than my desire to go back, so I let it slide.

But no longer. Looking through the code, the way I've handled structuring the measures, which in many ways are the
fundamental building blocks of the score, left a bit to be desired.

In the first version of the polyrhythm generator, there was no structure at all. Instead, each measure was
represented first and only by a single constructed string:

{% highlight elixir %}
# pulse
"\\time #{c}/8 \\repeat unfold #{c} { c8 }"

# all other parts
"\\tuplet #{c}/#{pulse_count} { \\repeat unfold #{c} { c8 } }"
{% endhighlight %}

Versions 2-4 were no better. When we got to version 5, this tuple started to show up:

{% highlight elixir %}
# pulse
{ {c, 8}, Stream.cycle(["c8"]) |> Enum.take(c)}

# all other parts
{ {c, pulse_count}, Stream.cycle(["c8"]) |> Enum.take(c)}
{% endhighlight %}

Now, instead of having a simple string for the measure, we have a tuple of

{% highlight elixir %}
{tuplet/time signature, notes}
{% endhighlight %}

with which we can more easily modify the notes in the measure, since they are
a list of items, rather than a string that would need to be split and parsed.

But this is still rather a cumbersome form to pass around, especially when,
as happens in many cases, we want to include the index of the measure in the part:

{% highlight elixir %}
{ { {n, d}, notes}, index}
{% endhighlight %}

For pattern matching, this is far from optimal, especially if we want to ignore
some of the fields or further match on the head/tail of the notes. Plus, we need
to know what each element in the form means. `{ { {n, d}, ns}, i}` is hardly the
most descriptive Elixir form.

Fortunately, Elixir provides [structs](https://elixir-lang.org/getting-started/structs.html),
which will help us accomplish what we want.

{% highlight elixir %}
defmodule Measure do
  defstruct [
    :time_signature, :tuplet, :events,
    :dynamic, :phoneme
  ]
end
{% endhighlight %}

With this data format, instead of needing to write code like this to attach
dynamics and phonemes to the first event of a measure

{% highlight elixir %}
density = Measure.density(measure)
new_events = [first_event <> dynamic_for_float(density, i) | rest_events]
{tuplet, new_events}

*and*

[note|ns] = notes
[note <> "^\\markup \"[#{phoneme}]\""|ns]
{% endhighlight %}

we can simply set the dynamic and phoneme in the struct attributes

{% highlight elixir %}
measure = %Measure{events: events}
measure = %Measure{ measure | dynamic: calculated_dynamic_for(measure) }
measure = %Measure{ measure | phoneme: calculated_phoneme_for(measure) }
{% endhighlight %}

and push LilyPond formatting off onto the module itself:

{% highlight elixir %}
defmodule Measure do
  defstruct [
    :time_signature, :tuplet, :events,
    :dynamic, :phoneme
  ]

  def density(%__MODULE__{tuplet: nil}), do: 1.0
  def density(%__MODULE__{tuplet: {0, _}}), do: 0.0
  def density(%__MODULE__{tuplet: {n, _}, events: events}) do
    Enum.count(events, fn e -> e == "c8" end) / n
  end

  def to_lily(measure = %__MODULE__{time_signature: {n, d}}) do
    "  \\time #{n}/#{d} #{events_to_lily(measure)}"
  end
  def to_lily(measure = %__MODULE__{tuplet: {n, d}}) do
    "  \\tuplet #{n}/#{d} { #{events_to_lily(measure)} }"
  end

  def events_to_lily(measure = %__MODULE__{events: [h|t]}) do
    with h <- h <> dynamic_markup(measure) <> phoneme_markup(measure) do
      [h|t] |> add_beaming() |> Enum.join(" ")
    end
  end

  def add_beaming(events) do
    events |> List.insert_at(1, "[") |> List.insert_at(-1, "]")
  end

  def phoneme_markup(%__MODULE__{phoneme: nil}), do: ""
  def phoneme_markup(%__MODULE__{phoneme: phoneme}) do
    ~s(^\\markup "[#{phoneme}]")
  end

  def dynamic_markup(%__MODULE__{dynamic: nil}), do: ""
  def dynamic_markup(%__MODULE__{dynamic: dynamic}), do: dynamic
end
{% endhighlight %}

This way, when generating the LilyPond files, all we need to call in our generators is

{% highlight elixir %}
Enum.map(measures, &Measure.to_lily/1)
{% endhighlight %}

and we have a properly formatted LilyPond string for each measure without needing
to fumble around with spacing and beaming in every generator.

The code, refactored to use this `Measure` struct consistently, can be found
[on Github here](https://github.com/mikowitz/alphabet_project/tree/v2).

<hr />
<br />

Thanks for reading! If you liked this post, and want to know when the next one
is coming out, follow me on Twitter (link below)!
