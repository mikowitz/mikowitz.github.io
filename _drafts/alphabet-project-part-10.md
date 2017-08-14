---
layout: post
title:  Alphabet Project, Part 10
categories: elixir music processing
---

1. [Alphabet Project, Part 1]({{ site.baseurl }}{% post_url 2017-07-20-alphabet-project-part-1 %})
1. [Alphabet Project, Part 2]({{ site.baseurl }}{% post_url 2017-07-22-alphabet-project-part-2 %})
1. [Alphabet Project, Part 3]({{ site.baseurl }}{% post_url 2017-07-23-alphabet-project-part-3 %})
1. [Alphabet Project, Part 4]({{ site.baseurl }}{% post_url 2017-07-27-alphabet-project-part-4 %})
1. [Alphabet Project, Part 5]({{ site.baseurl }}{% post_url 2017-08-01-alphabet-project-part-5 %})
1. [Alphabet Project, Part 6]({{ site.baseurl }}{% post_url 2017-08-06-alphabet-project-part-6 %})
1. [Alphabet Project, Part 7]({{ site.baseurl }}{% post_url 2017-08-10-alphabet-project-part-7 %})

Well, friends, it's been quite journey, but I'm afraid this particular path is reaching its end. At the end of part 9,
there were two outstanding issues I told you I'd address in this post, so, without further ado:

#### Why I'm not messing with cleaning up repeated rests

In a more traditionally rhythmic piece, drawing rests to conform to the implied pulse of a measure
helps performers to keep time and their place. In a piece like this -- in which almost every measure
is a more irregular tuplet against an irregular number of beats -- there are less clearly defined pulses for
each part, and, because of that, my opinion is that it is most useful to performers to see each beat
of the tuplet discretely printed, to help with counting.

In addition, because in a measure with irregular beats, the implied pulse is uneven and shifting,
calculating *where* rests should be divided up becomes a much harder problem. Which I may well
tackle in the future just as a fun exercise, but it's beyond the scope of the work needed for this
particular piece.

So, on to our last issue:

#### Extracting parts

This is a difficult score, and it's a *very* difficult score to follow one line through. So, for the sake of any
potential performers, it would be in everyone's best interest to be able to extract clean parts for each performer.

When we generate the LilyPond score files, each part is written to its own file so we don't have one absolutely,
unparseably, massive file. This gets us a good bit of the way there. If we take the `b` part and add a quick
score output block to the LilyPond code:

{% highlight latex %}
\score {
  \new Staff { \bMusic }
}
{% endhighlight %}

and compile it, we get

[![alphabet-part-10/b_bad.png]({{ site.url }}/assets/alphabet-part-10/b_bad.png)]({{ site.url }}/assets/alphabet-part-10/b_bad.png)

Oh. Umm, ok. We seem to be missing all our irregular time signatures. This is because we only ever actually define them
in the pulse part's notation. When LilyPond is trying to compile a part without specified time signatures, it
defaults to `4/4`. So we need to get our time signatures into this score somehow. We have two options:

1. refactor our `Measure` struct to add and write the time signature out in every part's LilyPond content
1. include the pulse part in each extracted part

I think option 2 is the correct choice, for a couple of reasons. First, it reduces the Elixir code churn, and
keeps each part's LilyPond content (which is already pretty large) smaller. Second, with the pulse providing, well,
the pulse of the piece, that each part's tuplets are generated against, it makes sense for each performer to be
able to see that along with their own part.

In LilyPond, this looks like

{% highlight latex %}
\include "a.ly"

\score {
  \new Staff { \aMusic }
  \new Staff { \bMusic }
}
{% endhighlight %}

and in a PDF, this gets us

[![alphabet-part-10/b+a.png]({{ site.url }}/assets/alphabet-part-10/b+a.png)]({{ site.url }}/assets/alphabet-part-10/b+a.png)

That's definitely better, but it's still got a ways to go:

* the systems are too close together, and have no strong visual divider
* the pulse part should be a bit smaller, to call attention to the correct line
* the time signature reminder at the end of the first system is cut off

Why isn't "repeated time signatures when the time signature doesn't change" on that list? I'm glad you asked,
and I will answer you in footnote. This footnote[^1].

As you might expect from a framework whose entire goal is the beautiful output of musical scores, LilyPond comes with
easy ways to solve the aforementioned issues.

* For adding a system divider, it gives us the `system-separator-markup` command
* For system spacing, we have `system-system-spacing`
* And for adjusting staff size, we can use `#(set-global-staff-size n)` and `\magnifyStaff`

Put all together, our part template looks like this:

{% highlight latex %}
#(set-global-staff-size 16)

\paper {
  system-separator-markup = \slashSeparator
  % basic-distance is the distance LilyPond tries to draw
  % minimum-distance is the minimum it can be shrunk to depending on other
  %   layout constraints
  % padding ensures the given amount of white space around elements
  system-system-spacing =
    #'((basic-distance . 25)
       (minimum-distance . 15)
       (padding . 3))
}

\score {
  \new StaffGroup <<
    \new Staff \with {
      \magnifyStaff #5/7
    } { \aMusic }
    \new Staff { \bMusic }
  >>
}
{% endhighlight %}

And that gives us, in PDF

[![alphabet-part-10/b_good.png]({{ site.url }}/assets/alphabet-part-10/b_good.png)]({{ site.url }}/assets/alphabet-part-10/b_good.png)

Look at that! Now let's add some code to generate 26 of these for us.

We already have code that iterates through each letter and generates the full part, so it's easy to add an extra
line to write these part scores as well.

{% highlight elixir %}
def generate_parts(pulse) do
  # Make sure our output directory exists
  File.mkdir_p("score/parts")
  Enum.map(@alphabet, fn letter ->
    case letter do
      ^pulse -> pulse_part(pulse)
      _      -> letter_part(letter, pulse)
    end
  end)
end

def pulse_part(pulse) do
  # Writing full part elided
  ...
  File.write!("score/parts/#{pulse}.ly", part_template(pulse, pulse))
  {:ok, pulse}
end

def letter_part(letter, pulse) do
  # Writing full part elided
  ...
  File.write!("score/parts/#{letter}.ly", part_template(letter, pulse))
  {:ok, letter}
end
{% endhighlight %}

`part_template/2` contains the following LilyPond code - almost identical
to what we had above

{% highlight latex %}
\version "2.19.61"
\language "english"

\include "../#{pulse}.ly"
\include "../#{letter}.ly"

#(set-default-paper-size "11x17")
#(set-global-staff-size 16)

\paper {
  system-separator-markup = \slashSeparator
  system-system-spacing =
    #'((basic-distance . 25)
       (minimum-distance . 15)
       (padding . 3))
}

\score {
  \new StaffGroup <<
    \new Staff \\with {
      \magnifyStaff #5/7
    } { \#{pulse}Music }
    \new Staff { \#{letter}Music }
  >>
}
end
{% endhighlight %}

The `include` lines look back up one directory, since we want to be able to keep our extracted parts in their
own folder, but they need access to the full part content that resides in our main lilypond directory.

To help visualize this, if we run `ScoreGenerator.generate_parts/1` again, we end up with this directory
structure

{% highlight shell %}
score/
|-- parts/
|   |-- a.ly
|   |-- b.ly
|   |-- (c..z).ly
|
|-- a.ly
|-- b.ly
|-- (c..z).ly
|-- score.ly
|-- score.pdf
{% endhighlight %}

In the top level `score/` directory we have the full parts, as well as the LilyPond and resulting PDF for the
whole score. In the `parts/` subdirectory we have the files we just generated for extracted parts, as well
as their resulting PDFs.

### Wrapup

And that, in 10 blog posts, is how to generate a 26-part, polyrhythmic score from a bunch of cool looking
charts your friend posted on Facebook. It's been a fun journey for me, and I hope, if you've made it this far,
you've enjoyed yourself well enough, too. And maybe, like me, you've even learned something, be it about Processing,
Elixir, composition, musical notation, or just how my brain works.

Thanks for reading along! Until next time!

[^1]: I've gone this far without addressing the fact that, even when a measure has the same time signature as the previous measure, I'm still printing it out. In most music, when the time signature does not change, it is not restated. But since the time signatures are shifting more often than not and are also, as mentioned, heavily irregular, I'm choosing to err on the side of caution and call them out for every measure.

