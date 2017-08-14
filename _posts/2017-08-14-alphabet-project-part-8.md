---
layout: post
title:  Alphabet Project, Part 8
date:   2017-08-14 09:33:00 -0400
categories: elixir music processing
---

1. [Alphabet Project, Part 1]({{ site.baseurl }}{% post_url 2017-07-20-alphabet-project-part-1 %})
1. [Alphabet Project, Part 2]({{ site.baseurl }}{% post_url 2017-07-22-alphabet-project-part-2 %})
1. [Alphabet Project, Part 3]({{ site.baseurl }}{% post_url 2017-07-23-alphabet-project-part-3 %})
1. [Alphabet Project, Part 4]({{ site.baseurl }}{% post_url 2017-07-27-alphabet-project-part-4 %})
1. [Alphabet Project, Part 5]({{ site.baseurl }}{% post_url 2017-08-01-alphabet-project-part-5 %})
1. [Alphabet Project, Part 6]({{ site.baseurl }}{% post_url 2017-08-06-alphabet-project-part-6 %})
1. [Alphabet Project, Part 7]({{ site.baseurl }}{% post_url 2017-08-10-alphabet-project-part-7 %})

### Refactoring LilyPond output - formatting measures

In the last post I refactored code around representing measures as structs, rather than
anonymous tuples. That version of the code is
[on Github here](https://github.com/mikowitz/alphabet_project/tree/v2).

#### Beaming

The goal of that refactor was to clean up the code base, not to make any changes to the resulting output,
but I did end up making one small change. In `Measure.events_to_lily/` I added a call to
`Measure.add_beaming/1`

{% highlight elixir %}
defmodule Measure do
  def events_to_lily(measure = %__MODULE__{events: [h|t]}) do
    with h <- h <> dynamic_markup(measure) <> phoneme_markup(measure) do
      [h|t] |> add_beaming() |> Enum.join(" ")
    end
  end

  def add_beaming(events) do
    events |> List.insert_at(1, "[") |> List.insert_at(-1, "]")
  end
end
{% endhighlight %}

All this does is insert `[` as the second element of the events list, and `]` as the last element.
When this is compiled by LilyPond, this ensures that all the notes in a measure will be beamed together.
To illustrate, this is the difference between

[![alphabet-part-8/unbeamed.png]({{ site.url }}/assets/alphabet-part-8/unbeamed.png)]({{ site.url }}/assets/alphabet-part-8/unbeamed.png)

and
[![alphabet-part-8/beamed.png]({{ site.url }}/assets/alphabet-part-8/beamed.png)]({{ site.url }}/assets/alphabet-part-8/beamed.png)

which I think looks rather nicer.

#### Full measure rests

However, on a less aesthetic note, this is also the difference between

[![alphabet-part-8/unbeamed_rests.png]({{ site.url }}/assets/alphabet-part-8/unbeamed_rests.png)]({{ site.url }}/assets/alphabet-part-8/unbeamed_rests.png)

and
[![alphabet-part-8/beamed_rests.png]({{ site.url }}/assets/alphabet-part-8/beamed_rests.png)]({{ site.url }}/assets/alphabet-part-8/beamed_rests.png)

Clearly the beaming doesn't work as well here. But more importantly, in both cases we should be able to display a full measure rest, instead of a tuplet
made of up individual rests.

This is a small change in the Elixir code. We can add a function to check
whether every event in a measure is a rest (`all_rests?/1`), and then `case`
on the value of that function call to either print the events as usual,
or use LilyPond's full measure rest syntax to print out the appropriate rest
symbol for the measure.

{% highlight elixir %}
defmodule Measure do
  def all_rests?(%__MODULE__{events: events}) do
    Enum.all?(events, &(&1 == "r8"))
  end

  def to_lily(measure = %__MODULE__{time_signature: {n, d}}) do
    case all_rests?(measure) do
      false -> "  \\time #{n}/#{d} #{events_to_lily(measure)}"
      true  -> "  \\time #{n}/#{d} R8 * #{n}"
    end
  end
  def to_lily(measure = %__MODULE__{tuplet: {n, d}}) do
    case all_rests?(measure) do
      false -> "  \\tuplet #{n}/#{d} { #{events_to_lily(measure)} }"
      true  -> "  R8 * #{d}"
    end
  end
end
{% endhighlight %}

This gives us the much more preferable

[![alphabet-part-8/full_measure_rest.png]({{ site.url }}/assets/alphabet-part-8/full_measure_rest.png)]({{ site.url }}/assets/alphabet-part-8/full_measure_rest.png)

#### Printing durations

So far every printed note has been an 8th note. While this works for getting the
basic layout of the piece set, it's hardly ideal for a final printed score.
Let's look at the opening measure

[![alphabet-part-8/m1_before.png]({{ site.url }}/assets/alphabet-part-8/m1_before.png)]({{ site.url }}/assets/alphabet-part-8/m1_before.png)

Already we see a few odd ratios: `60:30`, `3:30`, `7:30`, `10:30`, and so on.
In many cases these are reducible, but beyond that, the tuplet should not
be using 8th notes.

There are two issues it would behoove us to solve here:

1. find measures in which the tuplet divides evenly into the measure and "detupletify"
1. for measures that require tuplets, but for which the ratio should *not* be using
8th notes, and pick a more suitable duration

Issue 2 is slightly easier to work out, and helps us on our way to solving issue 1,
so we'll start there:

#### Correcting notated tuplet durations

When notating tuplets in music, and even more commonly now as rhythms in contemporary
music become more complex, there is some flexibility. For example, dividing a quarter
note beat into quintuplets: should they be notated as 5 sped up 16th notes, or 5 slowed down
32 notes? In almost all cases, composers and performers would prefer the 16th notes, but the
possibilty of alternate notations does remain.

To eliminate some of the subjectivity, or, rather, to render my general subjectivity into code,
I'm going to say that, for each tuplet, we will

* find the exact, probably fractional, number of 8th notes that would be required to fit the tuplet, let's call this value `x`
* round `x` based on standard float rounding (0.5 and higher rounds up, everything else rounds down)
* this gives us the base count of 8th notes to use per tuplet event
* find the closest untied, notateable duration given that 8th note count
* renotate the measure using this duration

Addendum: if `x` would round to 0, the ratio is greater than 2:1, in which case we should use 16th notes against 8th notes

An example:

Given the ratio `7:30`, we want to find the untied musical duration
that best fits into 30 seven times.

`30 / 7 = 4.285...` which rounds to `4` 8th notes, or a half note. So the measure
would be rewritten using 7 half notes instead of 7 8th notes.

In code, this looks something like this:

{% highlight elixir %}
defmodule Measure do
  defstruct [
    :time_signature, :tuplet, :events,
    :dynamic, :phoneme, :written_duration
  ]

  ...

  def set_proper_duration(measure = %Measure{time_signature: {n, d}, tuplet: nil}) do
    %Measure{ measure | written_duration: "8" }
  end
  def set_proper_duration(measure = %Measure{tuplet: {n, d})) do
    with x <- round(d, n) do
      duration = case x do
        0 -> "16"
        1 -> "8"
        2 -> "4"
        3 -> "4."
        n when n in [4, 5] -> "2"
        6 -> "2."
        7 -> "2.."
        _ -> "1"
      end
      %Measure{ measure | written_duration: duration }
    end
  end
end
{% endhighlight %}

Here we find the ratio for the tuplet, and based on our decisions above, pick the
most appropriate untied duration. If the time signature is set instead of the tuplet --
indicating the pulse part -- we return "8". I've also added the `:written_duration`
attribute to the Measure struct, and we store this duration in that attribute
to make it more easily accessible.

Now we need to update `Measure.to_lily/1` to make sure we use this new duration:

{% highlight elixir %}
def replace_durations(measure = %__MODULE__{events: events, tuplet: nil}), do: measure
def replace_durations(measure = %__MODULE__{tuplet: {_, _}}) do
  with measure <- set_proper_duration(measure) do
    new_events = Enum.map(measure.events, fn e ->
      Regex.replace(~r/\d+$/, e, measure.written_duration)
    end)
    %Measure{ measure | events: new_events }
  end
end
{% endhighlight %}

Simple enough! Let's see what this looks like:

[![alphabet-part-8/page1_problematic.png]({{ site.url }}/assets/alphabet-part-8/page1_problematic.png)]({{ site.url }}/assets/alphabet-part-8/page1_problematic.png)

Huh. Well, I won't pretend I don't think that looks pretty cool. But, it's not really what we're going for, so let's figure out what went wrong.

Turns out it's not too difficult to suss out the issue. Let's take a closer look at the 8th and 9th staves:

[![alphabet-part-8/h+i_problematic.png]({{ site.url }}/assets/alphabet-part-8/h+i_problematic.png)]({{ site.url }}/assets/alphabet-part-8/h+i_problematic.png)

If we look at the numbers above the staves in the 2nd measure, we see `7` and `10`, which are the tuplet values for these parts. But, the tuplet ratios are still
`7:30` and `10:30`, which were specifically for 8th notes. Now, in the top line, we're trying to fit 7 half notes into the space of 30 half notes, when instead we want
to fit 7 half notes into the space of 30 *8th notes*! No wonder everything looks off!

Fortunately, this is simple to fix. Based on the division and rounding done in `set_proper_duration/1` above, we know how many 8th notes are in each new notated tuplet
event, so all we have to do is multiply the numerator of the tuplet by that number. For example:

Our original ratio of 7 8th notes into 30 8th notes in now 7 half notes into 30 eigth notes. Each half note contains 4 8th notes, so we transform the tuplet into
`(7 * 4):30 = 28:30`. In order to do this, we should also store the multiplicand on the measure as well:

{% highlight elixir %}
defmodule Measure do
  defstruct [
    :time_signature, :tuplet, :events,
    :dynamic, :phoneme, :written_duration,
    :eigth_notes_per_duration
  ]
  ...

  def set_proper_duration(measure = %Measure{time_signature: {_, _}, tuplet: nil}) do
    %Measure{ measure | written_duration: "8", eigth_notes_per_duration: 1 }
  end
  def set_proper_duration(measure = %Measure{tuplet: {n, d}}) do
    with x <- round(d / n) do
      {x, duration} = case x do
        0 -> {x, "16"}
        1 -> {x, "8"}
        2 -> {x, "4"}
        3 -> {x, "4."}
        n when n in [4, 5] -> {4, "2"}
        6 -> {x, "2."}
        7 -> {x, "2.."}
        _ -> {8, "1"}
      end
      %Measure{ measure | written_duration: duration, eigth_notes_per_duration: x }
    end
  end
end
{% endhighlight %}

[![alphabet-part-8/page1_better.png]({{ site.url }}/assets/alphabet-part-8/page1_better.png)]({{ site.url }}/assets/alphabet-part-8/page1_better.png)

There we go! Except that some of those 8th notes really should be 16th notes, but they're not. And that's because my math is bad. When rounding, I said
that a rounded value of 0 should return 16th notes, but 8/16 is exactly 0.5, which gets rounded up to 1, which means we stay using 8th notes.

With just a couple of small code changes, to account for being able to map to 16th notes, and to round the tuplet numerator properly:

{% highlight elixir %}
defmodule Measure do
  ...
  def _to_lily(measure = %__MODULE__{tuplet: {n, d}, eigth_notes_per_duration: e}) do
    case all_rests?(measure) do
      true  -> "  R8 * #{d}"
      false -> "  \\tuplet #{round(n * e)}/#{d} { #{events_to_lily(measure)} }"
    end
  end

  ...

  def set_proper_duration(measure = %Measure{time_signature: {_, _}, tuplet: nil}) do
    %Measure{ measure | written_duration: "8" }
  end
  def set_proper_duration(measure = %Measure{tuplet: {n, d}}) when d / n <= 0.5 do
    %Measure{ measure | written_duration: "16", eigth_notes_per_duration: 0.5 }
  end
  def set_proper_duration(measure = %Measure{tuplet: {n, d}}) do
    with x <- round(d / n) do
      {x, duration} = case x do
        0 -> {x, "16"}
        1 -> {x, "8"}
        2 -> {x, "4"}
        3 -> {x, "4."}
        n when n in [4, 5] -> {4, "2"}
        6 -> {x, "2."}
        7 -> {x, "2.."}
        _ -> {8, "1"}
      end
      %Measure{ measure | written_duration: duration, eigth_notes_per_duration: x }
    end
  end
end
{% endhighlight %}

[![alphabet-part-8/page1_with_16ths.png]({{ site.url }}/assets/alphabet-part-8/page1_with_16ths.png)]({{ site.url }}/assets/alphabet-part-8/page1_with_16ths.png)

Great! There's only one more thing that stands out as affecting the readability of the score, and that's the text of the tuplets. In the fourth staff, we have a tuplet of `24:30`, but
it is notated with 3 whole notes. As calculated above, the tuplet ratio is between 8th notes on both sides, but when the notation uses whole notes, `24` is less descriptive than
we might like. Fortunately, LilyPond lets us put just about anything we want in the tuplet text, so let's go ahead and do that.

I also took this chance to address issue #1 from a ways back in this post: find measures in which
the tuplet divides evenly into the measure and "detupletify" In those cases,
we can print the proper generated duration without needing the tuplet markup, as you can see
in the score below in the second and ninth staves.

{% highlight elixir %}
  def _to_lily(measure = %__MODULE__{tuplet: {n, d}, eigth_notes_per_duration: e}) do
    case all_rests?(measure) do
      # All rests, print a full measure rest
      true  -> "  R8 * #{d}"
      false ->
        with ratio <- round(n * e) / d do
          case round(ratio) == ratio do
            # The ratio does not to be tupleted, no extra markup necessary
            true -> events_to_lily(measure)
            # Generate a descriptive tuplet text mark
            false ->
              "  \\once \\override TupletNumber #'text =\n" <>
              "    #(tuplet-number::non-default-fraction-with-notes #{n} \"#{measure.written_duration}\" #{d} \"8\")"
              <> "\n" <>
              "  \\tuplet #{round(n * e)}/#{d} { #{events_to_lily(measure)} }"
          end
        end
    end
  end
{% endhighlight %}

[![alphabet-part-8/page1_with_tuplet_text.png]({{ site.url }}/assets/alphabet-part-8/page1_with_tuplet_text.png)]({{ site.url }}/assets/alphabet-part-8/page1_with_tuplet_text.png)

The changes here are even more apparent on the last page of the score, where several measures of `4:2` tuplets
have been renotated as 4 untupletd 16th notes:

[![alphabet-part-8/page78_with_tuplet_text.png]({{ site.url }}/assets/alphabet-part-8/page78_with_tuplet_text.png)]({{ site.url }}/assets/alphabet-part-8/page78_with_tuplet_text.png)

I think that's enough for one post, but here's a list of what I want to tackle next time:

* attach the consonant phonemes to their parts
* clean up repeated dynamics and add some hairpins as dynamics change
* have dynamic and phoneme markup attached to the first note in a measure, not the first event,
which might be a rest
* work on being able to extract clean, legible parts for performers
* try to clean up repeated 8th and 16th rests, though this is a tricky task while retaining some semblance of
    traditional score legibility (i.e. notating rests on the downbeats of natural measure subdivisions, rather than
        willy-nilly)

The code, as it exists by the end of this post, can be found
[on Github here](https://github.com/mikowitz/alphabet_project/tree/v3).

<hr />
<br />

Thanks for reading! If you liked this post, and want to know when the next one
is coming out, follow me on Twitter (link below)!
