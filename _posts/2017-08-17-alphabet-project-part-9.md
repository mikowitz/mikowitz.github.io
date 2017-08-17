---
layout: post
title:  Alphabet Project, Part 9
date:   2017-08-17 08:35:00 -0400
categories: elixir music processing
---

1. [Alphabet Project, Part 1]({{ site.baseurl }}{% post_url 2017-07-20-alphabet-project-part-1 %})
1. [Alphabet Project, Part 2]({{ site.baseurl }}{% post_url 2017-07-22-alphabet-project-part-2 %})
1. [Alphabet Project, Part 3]({{ site.baseurl }}{% post_url 2017-07-23-alphabet-project-part-3 %})
1. [Alphabet Project, Part 4]({{ site.baseurl }}{% post_url 2017-07-27-alphabet-project-part-4 %})
1. [Alphabet Project, Part 5]({{ site.baseurl }}{% post_url 2017-08-01-alphabet-project-part-5 %})
1. [Alphabet Project, Part 6]({{ site.baseurl }}{% post_url 2017-08-06-alphabet-project-part-6 %})
1. [Alphabet Project, Part 7]({{ site.baseurl }}{% post_url 2017-08-10-alphabet-project-part-7 %})
1. [Alphabet Project, Part 8]({{ site.baseurl }}{% post_url 2017-08-14-alphabet-project-part-8 %})

### Refactoring LilyPond output - Part Deux

I ended the last post with a list of nice-to-have improvements. To refresh both our memories:

* attach the consonant phonemes to their parts
* clean up repeated dynamics and add some hairpins as dynamics change
* have dynamic and phoneme markup attached to the first note in a measure, not the first event,
  which might be a rest
* work on being able to extract clean, legible parts for performers
* try to clean up repeated 8th and 16th rests, though this is a tricky task while retaining some semblance of
    traditional score legibility (i.e. notating rests on the downbeats of natural measure subdivisions, rather than
                willy-nilly)

Today, I'm going to try to get through this list and end with a pretty, legible score with extracted parts.

#### Attach consonant phonemes

I've started here because this is by far the easiest task out of the lot. Currently our code looks like this:

{% highlight elixir %}
def consonant_phoneme_modulation_points(letter) do
  PolyrhythmGenerator.ordered_coordinates(letter)
  |> Enum.sort_by(fn {_, y} -> y end)
  |> Enum.with_index
  |> Enum.filter(fn { {x, _}, i} -> rem(i, 20) == 0 || x == 0 end)
  |> Enum.zip(Stream.cycle(["ei", "i:", "ai", "ou", "u:"]))
  |> Enum.map(fn { { {x, _}, _}, vowel} -> {x, vowel} end)
  |> Enum.sort
end
{% endhighlight %}

The important lines for our purposes today are

{% highlight elixir %}
# zip the selected measure indices with their respective phonemes
|> Enum.zip(Stream.cycle(["ei", "i:", "ai", "ou", "u:"]))
# return tuples of {index, phoneme}
|> Enum.map(fn { { {x, _}, _}, vowel} -> {x, vowel} end)
{% endhighlight %}

All we need to do is update the return tuple in the last line there to
prepend the consonant phoneme to the vowel.

In the American English International Phonetic Alphabet, most English consonants
are represented by their own character, so we only need to handle a few small cases
when mapping between them. So let's add a function to do that

{% highlight elixir %}
def consonant_phoneme_for(letter) do
  Map.get(%{
    "c" => "k", "j" => "d͡ʒ", "q" => "kʰ", "r" => "ɹ", "x" => "ks", "y" => "j"
  }, letter, letter)
end
{% endhighlight %}

If the letter we pass in to `consonant_phoneme_for/1` is a key in the map,
it returns the value for that key, otherwise, we can use the letter character
itself, so we just return that. Now we can easily plug this in to `consonant_phoneme_modulation_points/1`
like so

{% highlight elixir %}
def consonant_phoneme_modulation_points(letter) do
  # Find the appropriate consonant character
  with consonant_phoneme <- consonant_phoneme_for(letter) do
    PolyrhythmGenerator.ordered_coordinates(letter)
    |> Enum.sort_by(fn {_, y} -> y end)
    |> Enum.with_index
    |> Enum.filter(fn { {x, _}, i} -> rem(i, 20) == 0 || x == 0 end)
    |> Enum.zip(Stream.cycle(["ei", "i:", "ai", "ou", "u:"]))
    # And prepend it to the vowel before returning
    |> Enum.map(fn { { {x, _}, _}, vowel} -> {x, consonant_phoneme <> vowel} end)
    |> Enum.sort
  end
end
{% endhighlight %}

[![alphabet-part-9/page1_consonants.png]({{ site.url }}/assets/alphabet-part-9/page1_consonants.png)]({{ site.url }}/assets/alphabet-part-9/page1_consonants.png)

And there we go! The vowel parts remain the same, but the consonant parts also print their consonant phoneme.

Onwards!

#### Clean up repeated dynamics

Let's look at the last page of the score:

[![alphabet-part-8/page78_with_tuplet_text.png]({{ site.url }}/assets/alphabet-part-8/page78_with_tuplet_text.png)]({{ site.url }}/assets/alphabet-part-8/page78_with_tuplet_text.png)

Every measure has a dynamic attached to it, even when that dynamic is the same as the measure before. Traditionally, we
would only display the dynamic when it changes, and doing so here will make the score look cleaner.

All we need to do is compare each measure with the measure before it, and, if it has the same dynamic, remove it so it won't print.

My first attempt at this was a very Elixir-esque, recurse-with-an-accumulator:

{% highlight elixir %}
# start off with the first measure in the accumulator
# and its dynamic as the `current_dynamic`
def clean_dynamics([m|ms]), do: clean_dynamics(ms, m.dynamic, [m])

# if there are 0 or 1 measures left, we don't need to worry about
# the next measure, so we just reverse the accumulator and return
def clean_dynamics([], _, acc), do: Enum.reverse(acc)
def clean_dynamics([m], _, acc), do: Enum.reverse([m|acc])
# otherwise
def clean_dynamics([m|ms], current_dynamic, acc) do
  # if the next measure has the same dynamic
  # as the `current_dynamic`, update the measure to
  # delete the dynamic
  case m.dynamic == current_dynamic do
    true ->
      new_m = %Measure{ m | dynamic: nil}
      clean_dynamics(ms, current_dynamic, [new_m|acc])
    # if the dynamic is different, leave the dynamic
    # on the measure and set the new `current_dynamic`
    false ->
      clean_dynamics(ms, m.dynamic, [m|acc])
  end
end
{% endhighlight %}

But then I read a blog post[^1] that mentioned how, while this is indeed a very Erlang/Elixir way
of doing things, the `Enum` module often provides sufficient functionality
to be able to accomplish the same work in a single `Enum.map` loop, so I thought
I'd give it a try:

{% highlight elixir %}
def clean_dynamics(measures = [m|_]) do
  # create 2-element, overlapping chunks for each measure + the following measure
  # passing :discard ensures that any incomplete chunk
  # (for example, the last measure by itself)
  # is ignored. This saves us having to explicitly remove it later.
  new_measures = Enum.chunk_every(measures, 2, 1, :discard)
  |> Enum.map(fn [m1, m2] ->
       # compare the dynamics for the two measures
       case m1.dynamic == m2.dynamic do
         # if they're the same, delete the second measure's dynamic
         # and return it
         true -> %Measure{ m2 | dynamic: nil }
         # otherwise, just return the second measure unchanged
         false -> m2
       end
  end)
  # since the map above returns the second element of each chunk,
  # we've lost the very first measure, so we push that
  # back on the front of the measures and return them all
  [m|new_measures]
end
{% endhighlight %}

As far as code size goes, the second solution is definitely shorter, and does reduce a bit of the
mental overhead required to parse it, since it's only one function. Admittedly, there's
something fun about tail recursion, but since I'd like to be able to read this code again
in the future, and since they both return the same results, let's stick with the
second, shorter solution. Thanks for the tip, forgotten blog author!

[![alphabet-part-9/page71_dynamics.png]({{ site.url }}/assets/alphabet-part-9/page71_dynamics.png)]({{ site.url }}/assets/alphabet-part-9/page71_dynamics.png)

No repeated dynamics! Well, there are still a few, but they only reappear after an entire measure
of rest (see the bottom line). This is acceptable, since after rests it can be helpful to re-state
the dynamic.

#### Add hairpins

To give the piece a bit of movement, and, honestly, make the score a bit more interesting
to read, I want to add some crescendos and decrescendos when the dynamics shift. My plan is
to add a full measure dynamic to the measure before a dynamic change.

Because we need to know the last printed dynamic in the piece, we need to keep an
accumulator of sorts for dynamic to keep track of the last non-`nil` value. To do this,
we need to return to the tail recursion approach we tried and rejected above:

{% highlight elixir %}
# call to a private method, also passing along the current dynamic and
# an empty measure accumulator
def add_hairpins(measures = [m|_]), do: _add_hairpins(measures, m.dynamic, [])

# if the list has 0 or 1 elements remaining (we will never draw a hairpin on
# the final measure), we can add anything remaining to the front
# of the accumulator, reverse it, and return it
defp _add_hairpins(l, _, acc) when length(l) <= 1, do: Enum.reverse(l ++ acc)
# otherwise, if there are at least 2 elements remaining
defp _add_hairpins([m,m2|ms], current_dynamic, acc) do
  # if the 2nd measure has no dynamic, we keep the same current dynamic
  # otherwise we set that measure's dynamic to pass for the next iteration
  next_dynamic = case m2.dynamic do
    nil -> current_dynamic
    d -> d
  end
  # a shortcut value to check whether either measure is entirely composed of rests
  # we don't need to print a hairpin over or pointing towards rests
  either_measure_all_rests = Measure.all_rests?(m) || Measure.all_rests?(m2)
  new_m = cond do
    # if the second measure's dynamic is nil, the dynamic doesn't change
    # so we don't need to print a hairpin
    m2.dynamic == nil -> m
    # if the next measure's dynamic is less than the current dynamic
    # we need to print a decrescendo over the first measure
    dynamic_index(m2.dynamic) < dynamic_index(current_dynamic) && not either_measure_all_rests ->
      %Measure{m | hairpin: ">" }
    # if the next measure's dynamic is greater than the current dynamic
    # we need to print a crescendo over the first measure
    dynamic_index(m2.dynamic) > dynamic_index(current_dynamic) && not either_measure_all_rests ->
      %Measure{m | hairpin: "<" }
    # any other case doesn't require any change
    true -> m
  end
  # push the new measure onto the front of the accumulator, and recurse
  _add_hairpins([m2|ms], next_dynamic, [new_m|acc])
end
{% endhighlight %}

Then we also need to make a couple small changes to our `Measure` struct to print out these hairpins:

{% highlight diff %}
defmodule Measure do
  ...
  def events_to_lily(measure = %__MODULE__{}) do
    with  [h|t] <- reduce(measure).events |> add_beaming(),
-         h <- h <> dynamic_markup(measure) <> phoneme_markup(measure)
+         h <- h <> dynamic_markup(measure) <> hairpin_markup(measure) <> phoneme_markup(measure)
    do
      [h|t] |> Enum.join(" ")
    end
  end

+ def hairpin_markup(%__MODULE__{hairpin: nil}), do: ""
+ def hairpin_markup(%__MODULE__{hairpin: hairpin}), do: "\\#{hairpin}"
end
{% endhighlight %}

And...

[![alphabet-part-9/page71_hairpins.png]({{ site.url }}/assets/alphabet-part-9/page71_hairpins.png)]({{ site.url }}/assets/alphabet-part-9/page71_hairpins.png)

Great! But there's always one more thing, isn't there?

Yes there is, and in this case it's that we're always attaching dynamics and phonemes
to the first event of a measure, whether or not it's a rest:

[![alphabet-part-9/rest_attachments.png]({{ site.url }}/assets/alphabet-part-9/rest_attachments.png)]({{ site.url }}/assets/alphabet-part-9/rest_attachments.png)

Rests can have neither dynamics nor pronunciation[^2], and so it makes little sense to attach such markup
to them. Right now we're attaching markup with this code

{% highlight elixir %}
def events_to_lily(measure = %__MODULE__{}) do
  with  [h|t] <- reduce(measure).events |> add_beaming(),
        h <- h <> dynamic_markup(measure) <> hairpin_markup(measure) <> phoneme_markup(measure)
  do
    [h|t] |> Enum.join(" ")
  end
end
{% endhighlight %}

where we deconstruct the list of events and write all the markup to the head of the list. Instead, we want to find
the first event that *isn't* a rest and attach to that. Let's see what that looks like

{% highlight elixir %}
def events_to_lily(measure = %__MODULE__{}) do
  with events <- reduce(measure).events,
       # find the first note in the measure
       first_note <- Enum.find(events, fn e -> not Regex.match?(~r/^r/, e) end),
       # find the index of the first note
       first_note_index = Enum.find_index(events, &(&1 == first_note))
  do
    # add markup to that note
    marked_up_note = first_note <> dynamic_markup(measure) <> hairpin_markup(measure) <> phoneme_markup(measure)
    # replace the un-marked-up note with its marked up version
    new_events = List.replace_at(events, first_note_index, marked_up_note) |> add_beaming()
    Enum.join(new_events, " ")
  end
end
{% endhighlight %}

[![alphabet-part-9/note_attachments.png]({{ site.url }}/assets/alphabet-part-9/note_attachments.png)]({{ site.url }}/assets/alphabet-part-9/note_attachments.png)

Hey, much better! Except, where did our beaming go? Let's look at the code that generates the beaming

{% highlight elixir %}
def add_beaming(events) do
  case Enum.all?(events, &Regex.match?(~r/(8|16)\.?$/, &1)) do
    true -> events |> List.insert_at(1, "[") |> List.insert_at(-1, "]")
    false -> events
  end
end
{% endhighlight %}

There's the issue in the `case` statement. We're making sure all the events in the measure
can be beamed (i.e. are 8th or 16th notes), by checking that each measure event ends
with an 8th/16th note (dotted or otherwise). But now that we're adding the markup *before*
we check for beaming, the markup keeps the event string from *ending* with the duration
notation, thus causing our `case` statement to always return false, and thus, no beaming.

Fortunately, this is an easy fix. We can just change the Regex match statement to check for
the presense of the correct duration notation anywhere in the event string, rather than just at the end

{% highlight diff %}
def add_beaming(events) do
- case Enum.all?(events, &Regex.match?(~r/(8|16)\.?$/, &1)) do
+ case Enum.all?(events, &Regex.match?(~r/(8|16)\.?/, &1)) do
    true -> events |> List.insert_at(1, "[") |> List.insert_at(-1, "]")
    false -> events
  end
end
{% endhighlight %}

[![alphabet-part-9/with_bindings.png]({{ site.url }}/assets/alphabet-part-9/with_bindings.png)]({{ site.url }}/assets/alphabet-part-9/with_bindings.png)

And there we go!

That's 3/5 of the remaining issues I listed at the beginning of the article.
Extracting parts will be the topic of the next post, and cleaning up repeated 8th and 16th notes
is a task I'm likely to pass on for now, the reasoning behind which will be a brief subtopic in the next post.
So thanks for reading, and stay tuned!

The code, as it exists by the end of this post, can be found
[on Github here](https://github.com/mikowitz/alphabet_project/tree/v4).

<hr />
<br />

Thanks for reading! If you liked this post, and want to know when the next one
is coming out, follow me on Twitter (link below)!

<hr />
<br />

[^1]: I'm pretty sure it was by José Valim or Chris McCord, either of whom definitely know what they're talking about as far as Elixir is concerned, but I can't track it down again.
[^2]: Though I'm sure there are composers out there who have determined they do in the context of a piece. This is not such a piece.
