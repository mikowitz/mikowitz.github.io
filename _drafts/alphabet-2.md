---
layout: post
title:  Alphabet, Part 2
categories: elixir music processing
---

Welcome back! Thanks for coming back! I know I promised we'd start off with turning
some numbers into music, but I wanted to take a quick detour back into generating those
numbers.

### Step 2, Part Deux

By the end of the last post I'd generated a set of graph coordinates for the letter `A`,
starting with

<img src="{{ site.url }}/assets/a.png" alt="A" style="padding-right:10px;" />

and ending up with a JSON array that looked, in part, like this:

{% highlight json %}
[
  [0, 30], [1, 30], [2, 30], [3, 30], [4, 30], [5, 30], [6, 30], [7, 30], [8, 30], [9, 30], [10, 30],
  [11, 30], [12, 30], [13, 30], [14, 30], [15, 30], [16, 30], [17, 30], [18, 30], [19, 30], [20, 30],
  [21, 30], [22, 30], [23, 30], [24, 30], [25, 30], [26, 30], [27, 30], [28, 30], [29, 30], [30, 30],
  [31, 31], [32, 31], [33, 31], [34, 31], [35, 32], [36, 32], [37, 32], [38, 32], [39, 33], [40, 33],
  [41, 33], [42, 33], [43, 33], [44, 34], [45, 34], [46, 35], [47, 35], [48, 36], [49, 36], [50, 37],
  ...
]
{% endhighlight %}

I said that we had to do this 25 more times, which is true, but after about 3 times, I noticed a few things.

First, it's really kind of annoying to open the Processing file, change the letter `"a"` to `"b"` every time, rerun it,
realize I missed one, and then have to go back and redo `"a"` because I overwrite some of the data.

Second, because of the not-particularly-scientific way I created the individual letter chart images, they're not all
the same height, meaning I need to find the dimensions of the image before I start.
Using `imagemagick` this is at least an easy task

{% highlight bash %}
> identify data/a.png
data/a.png PNG 202x49 202x49+0+0 8-bit sRGB 11.1KB 0.000u 0:00.000
{% endhighlight %}

And then I just need to open the Processing code, update the sketch dimensions, double check that I've
changed all instances of the letter character to the correct one, and...

Wait a minute! A series of easily performable but repetitive tasks? This sounds like a job for

### A script!

I've been getting really into [Elixir](http://elixir-lang.org), so let's use that.

Fortunately for us, Processing now includes a command line option, `processing-java`
that takes a path to our sketch's directory, a flag to `--run` or `--build`, and any
number of arguments.

Ideally, we'd want to run something like

`processing-java --sketch=/path/to/our/graph/detector --run <the letter> <width> <height>`

In Elixir, we can sequence our calls to `identify` and `processing-java` to do just that:

{% highlight elixir %}
defmodule CoordinateDetector do
  def process!(letter) do
    IO.write("Processing #{letter}...")
    dimensions = get_dimensions(letter)
    run_processing(letter, dimensions)
    IO.puts("Done")
  end

  defp get_dimensions(letter) do
    # Run imagemagick's `identify` and store the returned string
    {str, 0} = System.cmd("identify", ["data/#{letter}.png"])
    # Hooray pattern matching!
    # `Regex.scan/2` returns an array of arrays, where each inner array
    # is of the form [full_match | grouped_matches].
    # Since we want the two dimensions, we can match on
    # the second and third elements of the first array
    # and ignore everything else
    [[_, w, h] | _] = Regex.scan(~r/(\d+)x(\d+)/, str)
    # Return the dimensions as a tuple
    {w, h}
  end

  defp run_processing(letter, {w, h}) do
    # We'll be running this from the location of the Processing sketch,
    # so we can use `System.cwd/0` and not need to hard code our fs location
    System.cmd("processing-java", [
      "--sketch=#{System.cwd()}",
      "--run", letter, w, h
    ])
  end
end
{% endhighlight %}

Then, in our processing sketch, we just need to make a couple small changes:

{% highlight java %}
str letter;
str pngFilename;

void settings() {
  size(int(args[1]), int(args[2]));
}

void setup() {
  ...
  letter = args[0];
  // now we can use this anywhere we need to save an image.
  pngFilename = letter + ".png";
}
{% endhighlight %}

Next we add an Elixir funnction to iterate through the alphabet

{% highlight elixir %}
defmodule CoordinateDetector do
  @letters ~w(a b c d e f g h i j k l m n o p q r s t u v w x y z)

  def process_all! do
    Enum.each(@letters, fn letter ->
      process!(letter)
    end)
  end
  ...
end
{% endhighlight %}

And let's see how this goes:

{% highlight iex %}
iex(1)> :timer.tc(CoordinateDetector, :process_all!, [])
Processing a...Done
Processing b...Done
Processing c...Done
...
Processing z...Done
{71535134, :ok}
{% endhighlight %}

That time is in microseconds, so we're looking at 71 seconds. Not bad, but since
we're using Elixir, we might as well use one of its great features: spawning
processes to do work in a different thread.

A few changes to the Elixir code later

{% highlight elixir %}
defmodule CoordinateDetector do
  ...
  def process_all! do
    # spawn a new process to generate data for each letter
    # `Enum.map/2` here returns a list of PIDs...
    Enum.map(@letters, fn letter ->
      spawn(__MODULE__, :process!, [letter])
    end)
    # and we send a `:check` message to each PID, along with the PID of
    # the main process so we can send a response back
    |> Enum.map(fn pid ->
      send pid, {self(), :check}
    end)
    # start waiting for responses
    start_receiving(0)
  end

  def start_receiving(n) do
    # If we haven't received a response for all the letters...
    if n < length(@letters) do
      receive do
        # when we get a response back...
        {_pid, message} ->
          # print out the message and
          IO.puts message
          # call this function again with an incremented count
          start_receiving(n + 1)
      end
    # otherwise return `:ok`
    else
      {:ok, n}
    end
  end

  def process!(letter) do
    receive do
      # when the process receives `:check`
      {sender, :check} ->
        # run our Processing code
        IO.puts("Processing #{letter}...")
        dimensions = get_dimensions(letter)
        run_processing(letter, dimensions)
        # and send a done message back to the main thread
        send sender, {self(), "done processing #{letter}"}
    end
  end
  ...
end
{% endhighlight %}

and we run the code again

{% highlight iex %}
iex(1)> :timer.tc(CoordinateDetector, :process_all!, [])
Processing a...
Processing b...
Processing c...
...
Processing z...
Done processing x
Done processing a
Done processing r
...
Done processing o
{33144238, {:ok, 26}}
{% endhighlight %}

33 seconds! Much better. As you can see, we're not concerned about the order the
processes complete in, since they're entirely independent of each other.

And now, 33 seconds later, we've got 26 JSON files, each with a set of
coordinates. Now we can move on to the music!

<hr />
<br />

Thanks for reading! If you liked this post, and want to know when the next one
is coming out, follow me on Twitter (link below)!
