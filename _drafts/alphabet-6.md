---
layout: post
title:  Alphabet Project, Part 6
categories: elixir music processing
---

1. [Alphabet Project, Part 1]({{ site.baseurl }}{% post_url 2017-07-20-alphabet-project-part-1 %})
1. [Alphabet Project, Part 2]({{ site.baseurl }}{% post_url 2017-07-22-alphabet-project-part-2 %})
1. [Alphabet Project, Part 3]({{ site.baseurl }}{% post_url 2017-07-23-alphabet-project-part-3 %})
1. [Alphabet Project, Part 4]({{ site.baseurl }}{% post_url 2017-07-27-alphabet-project-part-4 %})

### Step 4 - adding (even more) musicality

At the beginning of [the last post]({{ site.baseurl }}{% post_url 2017-07-27-alphabet-project-part-4 %}),
I listed the three timbral components that make up each note. In that post,
we dealt with the first two: dynamic and pitch. Now, we're left with the final
component: phoneme sound/quality.

For the vowel lines, the natural sound of the letters provides a sustained sound
rather than the more percussive consonants. To start providing some variation,
the vowel parts (not including `Y`) will modulate back and forth between
the long and short versions of the vowels. These are

| vowel | short | long |
|:-----:|:-----:|:----:|
| A     | [æ] (cat, apple) | [ei] (late, make) |
| E     | [e] (let, tell) | [i:] (be, see) |
| I     | [i] (tip, pick) | [ai] (ice, find) |
| O     | [o] (not, rock) | [ou] (go, note) |
| U     | [ʌ] (cut, love) | [u:] (rude, June) |

<br/>

For the consonants, let's give them a bit of sustaining power, so they will
modulate through the long vowel sounds: [ei], [i:], [ai], [ou], [u:].

In order to decide when each part modulates, we're going back to the original
coordinate sets we generated in [the first post]({{ site.baseurl }}{% post_url 2017-07-20-alphabet-project-part-1 %})

{% highlight java %}
[
  [0, 30], [1, 30], [2, 30], [3, 30], [4, 30], [5, 30], [6, 30], [7, 30], [8, 30], [9, 30], [10, 30],
  [11, 30], [12, 30], [13, 30], [14, 30], [15, 30], [16, 30], [17, 30], [18, 30], [19, 30], [20, 30],
  [21, 30], [22, 30], [23, 30], [24, 30], [25, 30], [26, 30], [27, 30], [28, 30], [29, 30], [30, 30],
  [31, 31], [32, 31], [33, 31], [34, 31], [35, 32], [36, 32], [37, 32], [38, 32], [39, 33], [40, 33],
  [41, 33], [42, 33], [43, 33], [44, 34], [45, 34], [46, 35], [47, 35], [48, 36], [49, 36], [50, 37],
  ...
]
{% endhighlight %}

To determine the inflection points, in English:

{% highlight pseudocode %}
sort coordinates by Y value
select the coordinates at indices where rem(i, 20) == 0 # 10 total modulations
sort these coordinates by X value
attach a cycle of long and short vowel sounds
et voila! the inflection points
{% endhighlight %}

and in Elixir

{% highlight elixir %}
def vowel_phoneme_modulation_points(letter) do
  @polyrhythm_generator.ordered_coordinates(letter)
  |> Enum.sort_by(fn {_, y} -> y end)
  |> Enum.with_index
  |> Enum.filter(fn { {x, y}, i} -> rem(i, 20) == 0 || x == 0 end)
  |> Enum.map(fn { { {x, _}, _}, vowel} -> {x, vowel} end)
  |> Enum.sort
  |> Enum.zip(Stream.cycle(Map.get(vowel_phoneme_pairs(), letter)))
end
{% endhighlight %}

so that

{% highlight elixir %}
iex> vowel_phoneme_modulation_points("a")
{% endhighlight %}

gives us

{% highlight elixir %}
[
  {0, "æ"}, {18, "ei"}, {45, "æ"}, {60, "ei"}, {68, "æ"}, {95, "ei"},
  {103, "æ"}, {124, "ei"}, {142, "æ"}, {162, "ei"}, {191, "æ"}, {192, "ei"}
]
{% endhighlight %}

So this means that the `A` part would begin by singing [æ] in the first measure, and
by measure 19 (the list of measures above, like all lists in Elixir, is 0-indexed),
the part would have modulated to singing [ei], and then would stay on [ei] until
modulating back to [æ] by measure 46, and so on.

For the consonant parts, I want to do something a bit more involved, to make sure they're not
all modulating between vowels in the same order. To wit, I want it to work like this:

{% highlight pseudocode %}
sort coordinates by Y value
select the coordinates at indices where rem(i, 20) == 0 # 10 total modulations
NEW: attach, in order, a cycle of the 5 vowel sounds
sort these coordinates by X value
et voila! the inflection points
{% endhighlight %}

This is almost identical to above, except that we assign vowel phonemes while they're
still sorted by Y value, so that each part, when sorted again by X value, would
have its own ordering (or, at least, not a universal ordering) for the vowel additions.

As in the description above, in the Elixir code this is simply
a matter of changing the order of operations:

{% highlight elixir %}
def consonant_phoneme_modulation_points(letter) do
  @polyrhythm_generator.ordered_coordinates(letter)
  |> Enum.sort_by(fn {_, y} -> y end)
  |> Enum.with_index
  |> Enum.filter(fn { {x, y}, i} -> rem(i, 20) == 0 || x == 0 end)
  |> Enum.zip(Stream.cycle(["ei", "i:", "ai", "ou", "u:"]))
  |> Enum.map(fn { { {x, _}, _}, vowel} -> {x, vowel} end)
  |> Enum.sort
end
{% endhighlight %}

And as we can see, this does indeed provide us with some amount of phoneme variation
between the consonant parts:

{% highlight elixir %}
iex> consonant_phoneme_modulation_points("b")
[
  {0, "ei"}, {11, "i:"}, {21, "u:"}, {40, "ou"}, {62, "ai"}, {74, "i:"},
  {100, "ei"}, {133, "ou"}, {153, "u:"}, {160, "ai"}, {174, "ei"}, {184, "i:"}
]

iex> consonant_phoneme_modulation_points("c")
[
  {0, "ei"}, {21, "u:"}, {24, "ou"}, {36, "i:"}, {46, "ou"}, {66, "u:"},
  {74, "ai"}, {106, "ei"}, {124, "ai"}, {182, "i:"}, {191, "ei"}
]
{% endhighlight %}

Cool! Now let's add it into the score.

{% highlight elixir %}
defmodule ScoreGenerator do
  def letter_part_to_lily(letter, pulse) do
    {^letter, pitches} = Enum.find(measure_pitches(pulse), fn {l, _} ->
      l == letter
    end)
    modulation_map = phoneme_modulation_points(letter) |> Enum.into(Map.new)
    part = letter |> @dynamics_generator.measures(pulse)
    |> Enum.with_index |> Enum.map(fn { { {n, d}, notes}, i} ->
      pitch = Enum.at(pitches, i)
      # Make sure each note has the correct pitch
      notes = Enum.map(notes, fn n ->
        case n do
          "r8" -> "r8"
          note -> Regex.replace(~r/c/, note, pitch)
        end
      end)
      # if the measure is in the modulation map, attach the new phoneme
      # to the first note event
      notes = case Map.get(modulation_map, i) do
        nil -> notes
        phoneme ->
          [note|ns] = notes
          [note <> "^\\markup \"[#{phoneme}]\""|ns]
      end
      "\\tuplet #{n}/#{d} { #{Enum.join(notes, " ")} }"
    end) |> Enum.join("\n")
    write_lilypond_file(letter, part)
    {:ok, letter}
  end
end
{% endhighlight %}

Let's look at a few sample pages

[![final/v1/page1]({{ site.url }}/assets/final/v1/page1.png)]({{ site.url }}/assets/final/v1/page1.png)
[![final/v1/page44]({{ site.url }}/assets/final/v1/page44.png)]({{ site.url }}/assets/final/v1/page44.png)
[![final/v1/page85]({{ site.url }}/assets/final/v1/page85.png)]({{ site.url }}/assets/final/v1/page85.png)

Great! This has come a long way from the first draft. With the musical elements, and
the code that generates them, in place, it's time to move on to some refactoring, both
of the code and the resulting LilyPond output. Stay tuned!

<hr />
<br />

Thanks for reading! If you liked this post, and want to know when the next one
is coming out, follow me on Twitter (link below)!
