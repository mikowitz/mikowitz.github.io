---
layout: post
title:  "Introduction to Elixir Processes"
date:   2016-04-06 11:45:00 -0400
categories: elixir processes
---

# Introduction to Elixir Processes

(The) Space (of software development) is big. Really big.[^1]

And because of how vastly, hugely, mind-bogglingly big it is, you just can’t have all of it in your head all of the time. So patterns develop, and someone blogs about them, and someone else copies it from that blog into their code and tweaks it just enough so that it does what they, specifically, want, but maybe, just maybe, they don’t fully understand how or even *why* it does what they, specifically, want.

Now, of course, the “they” in that previous scenario is often me, and, probably, you, at least once in your life. And if you’re not a programmer, the same things applies to any language. I spent many years knowing what “chassis” and “chitin” meant, and how I’d use them in my own YA dystopian novels, but I had no idea how to pronounce them, because I read books instead of talking to people. (whether you took the verb “read” as past or present tense in the previous sentence, you’re right).

One of these patterns that I’ve been playing around with lately is basic processes in Elixir, particularly their `send` and `receive` paradigm. Fortunately, I’ve never had to implement one in a production setting, and my current reason to learn this is to play around with building some algorithmic composition tools, so even a production setting’s going to have a very small audience, but still. After a while of having to look up that one version of the pattern I found on that one blog that one time, it starts to feel like maybe I could just buckle down and figure out how to do this myself.

My goal for the rest of this is threefold:

1. By writing this down and explaining my thought process, finally understand how all these pieces work
2. By writing this down and explaining my thought process, help you understand how all these pieces work
3. Provide clear enough content to be my own “that one version of the pattern I found on that one blog post” (and maybe yours, if you feel the need to come back and visit)

We’re going to start with a simple example I found as a now-do-it-yourself example in Dave Thomas' excellent [Programming Elixir](https://pragprog.com/book/elixir12/programming-elixir-1-2):

“Write a program that spawns two processes and then passes each a unique token. Have them send the tokens back.”

If you’re following along, type or copy the following code into “echo.ex”

{% highlight elixir %}
defmodule Echo do
  def listen do
    receive do
      {sender, token} -> send sender, {self, token}
    end
  end

  def run(token1, token2) do
    pid1 = spawn(Echo, :listen, [])
    pid2 = spawn(Echo, :listen, [])

    send pid1, {self, token1}
    send pid2, {self, token2}

    receive do
      {^pid1, token} -> IO.puts token
    end
    receive do
      {^pid2, token} -> IO.puts token
    end
  end
end
{% endhighlight %}

Then, fire up iEX and run:

{% highlight iex %}
iex> c(“echo.ex”)
[Echo]
iex> Echo.run(“dear”, “reader”)
dear
reader
:ok
{% endhighlight %}

Let’s walk through this code quickly to see what’s going on here.

1. In the first two lines of `run` we `spawn` two new processes. In Elixir, spawn has two possible function signatures: `spawn/1` and `spawn/3`.
   `spawn/3`, which we use here, takes as its arguments a module, the name of a function on that module, and a list of arguments.
   In this case, since our `listen` function doesn't take any arguments, we pass an empty list.
   Elixir (and Erlang) requires an empty list. Providing `nil` or excluding the 3rd argument entirely will raise an `ArgumentError`.
   It spawns a new process running that function, and returns that process’s process identifier (PID).
   Here we are spawning two processes, each running `Echo.listen()`

2. Jumping to the definition of `listen`, we see it defines only a `receive` block. `receive` is an Elixir macro that works, for our intentions here, exactly as the `case` statement.
   It listens for messages being sent to its process, and, if one of its clauses matches the message, will perform the commands given.
   If nothing matches, the block will hang indefinitely or until a provided timeout is reached (more on this later). So here we have a process listening for a two-element tuple.

3. Now let’s jump back into `run`. We have our processes running, and we have their PIDs saved as `pid1` and `pid2`.
   Next we `send` each of those processes a message of the form `{self, token}`.
   `send/2` takes as its first argument a destination, and the message as its secord argument.
   `self` is a PID for the current process we are running in (IEx, in this case). As we saw above, this is exactly the message format our two `listen` processes are listening for.

4. `listen` now receives a tuple containing our IEx process’s PID and a token, and sends its own tuple containing its own PID and the token back to the `sender` process, which in this case is IEx.

5. Back in `run`, after we send our messages off, we need to get ourselves ready to receive a response. For each of our listener processes, we create a `receive` block that listens for a tuple containing the matching PID and a token, and simply `puts` the token.

6. Since `run` has no explicit return value, it returns `:ok` upon completion

Simple enough. But what’s with the two `receive` blocks at the end of `run`? Shouldn’t we be able to combine the two and have a single `receive` block with two match clauses? Let’s give that a try.

{% highlight elixir %}
defmodule Echo do
  def listen do
    receive do
      {sender, token} -> send sender, {self, token}
    end
  end

  def run(token1, token2) do
    pid1 = spawn(Echo, :listen, [])
    pid2 = spawn(Echo, :listen, [])

    send pid1, {self, token1}
    send pid2, {self, token2}

    receive do
      {^pid1, token} -> IO.puts token
      {^pid2, token} -> IO.puts token
    end
  end
end
{% endhighlight %}

And we run it:

    iex> r(Echo)
    [Echo]
    iex> Echo.run(“dear”, “reader”)
    dear
    :ok

What happened to the `reader`? This was one of the first things it took me a little while to remember: any `receive` block only runs once. It’s listening to a potentially endless stream messages coming into the current process's mailbox), but once it receives a matching message, it’s done. We
can also demonstrate this by sending multiple messages to one of our `listen` processes:

{% highlight elixir %}
defmodule Echo do
  def listen do
    receive do
      {sender, token} -> send sender, {self, token}
    end
  end

  def run(token1, token2) do
    pid1 = spawn(Echo, :listen, [])
    pid2 = spawn(Echo, :listen, [])

    send pid1, {self, token1}
    send pid1, {self, token1}
    send pid2, {self, token2}

    receive do
      {^pid1, token} -> IO.puts token
    end
    receive do
      {^pid2, token} -> IO.puts token
    end
  end
end
{% endhighlight %}

And we run it:

    iex> r(Echo)
    [Echo]
    iex> Echo.run(“dear”, “reader”)
    dear
    reader
    :ok


We receive and print the first "dear," but then that first `receive` is finished. We move on to the next one, but it doesn't match the first message in the process mailbox, which is another "dear," so it goes on to the second one, which is "reader," which matches, and the function terminates.

So we need a `receive` block to listen for each incoming message. And, if you run the code a number of times, you’ll see that the tokens are always printed in the same order. That’s because the receive blocks are processed in order. One waits until it receives a message, then goes on to the next one. You can confirm this for yourself by switching their order in your code and running it. You should see that the second token is now always printed first.

This is all simple enough when we know we’ll only be receiving two messages. But out in the real world, it’s rare that we can predict this with any measure of reasonability. So let’s make `Echo` behave a bit more like a real piece of code might:

{% highlight elixir %}
defmodule Echo do
  def listen do
    receive do
      {sender, token} -> send sender, {self, token}
    end
  end

  def run(tokens) do
    pids = Enum.map(tokens, fn token ->
      {spawn(Echo, :listen, []), token}
    end)
    Enum.map(pids, fn {pid, token} ->
      send pid, {self, token}
    end)

    ...
  end
end
{% endhighlight %}

So far, so good, we can spawn the correct number of processes, and send a token to each one. But how do we make sure we’ll be listening for each response? One way is to continue using the Enum.map:

{% highlight elixir %}
defmodule Echo do
  def listen do
    receive do
      {sender, token} -> send sender, {self, token}
    end
  end

  def run(tokens) do
    pids = Enum.map(tokens, fn token ->
      {spawn(Echo, :listen, []), token}
    end)
    Enum.map(pids, fn {pid, token} ->
      send pid, {self, token}
    end)

    Enum.map(pids, fn {pid, token} ->
      receive do
        {^pid, ^token} -> IO.puts token
      end
    end)
  end
end
{% endhighlight %}

And we run it:

    iex> r(Echo)
    iex> Echo.run(["wizard", "people", "dear", "reader"])
    wizard
    people
    dear
    reader
    [:ok, :ok, :ok, :ok]

Ok, so this works, but let’s say we change `listen` to send a token twice?

{% highlight elixir %}
def listen
  receive do
    {sender, token} ->
      send(sender, {self, token})
      send(sender, {self, token})
    end
  end
end
{% endhighlight %}



Sure, we can just double the block in our last `Enum.map` call.

{% highlight elixir %}
Enum.map(pids, fn {pid, token} ->
  receive do
    {^pid, ^token} -> IO.puts token
  end
  receive do
    {^pid, ^token} -> IO.puts token
  end
end)
{% endhighlight %}


But even though it works, and is more or less acceptable because we’re dealing with a finite (and self-determined) number of calls, hopefully you can see that we are heading down a dangerous road here. Say we change `listen` to do this:

{% highlight elixir %}
def listen
  receive do
    {sender, token} ->
      x = :random.uniform(100)
      for _ <- 1..x, do: send(sender, {self, token})
    end
  end
end
{% endhighlight %}


Now, for every PID we create, we need to listen for between 1 and 100 responses. The problem we’re facing is that every time a `receive` block matches, it’s done. It won’t be listening for the next matching message. But what if we could tell the `receive` block to keep listening after a match? Or, even better, what if we could tell the `receive` block to tell itself to keep listening after a match? Yes, that’s right, it’s time to pull out that old magic bullet and try not to shoot ourselves with it: recursion. And so:

{% highlight elixir %}
defmodule Echo do
  def listen do
    receive do
      {sender, token} -> send sender, {self, token}
    end
  end

  def run(tokens) do
    pids = Enum.map(tokens, fn token ->
      {spawn(Echo, :listen, []), token}
    end)
    Enum.map(pids, fn {pid, token} ->
      send pid, {self, token}
    end)

    start_receiving
  end

  def start_receiving do
    receive do
      {_pid, token} ->
        IO.puts token
        start_receiving
    end
  end
end
{% endhighlight %}


But wait, you might ask, won’t continuing to call `start_receiving` from inside itself blow up memory usage? (You may also be asking another question in addition to that one, but I'll get to that in a moment)

Normally, yes, since each successive call to `start_receiving` should push a new frame onto the stack without removing one (since each call to `start_receiving` wouldn't end until it's recursive call completes, which wouldn't end until it's recursive call completes, which...) But thanks to Elixir’s implementation of tail-call optimization (stay tuned for an entry on this in a hopefully continuing series of blogs posts about things I make use of but don’t understand), if the recursive call is the *VERY LAST THING* executed, "the runtime simply jumps back to the start of the function." [cit.](https://pragprog.com/book/elixir12/programming-elixir-1-2) Here, `start_receiving` is the last call we make inside of `start_receiving`, so we’re good.

So let’s run that, and you’ll get something that looks like

    iex> r(Echo)
    iex> Echo.run(["wizard", "people", "dear", "reader"])
    reader
    wizard
    people
    dear
    reader

and then...

well, nothing. And this is the question you may have been asking before (and don’t worry if you weren’t. I certainly didn’t the first several times I tried to implement something like this). What’s happening is that every time `start_receiving` receives a matching message, it calls itself and starts listening again. And this includes when `start_receiving` receives the final message we’re sending it. It gets told to start receiving again, and it does. But nothing ever comes in.

"Hah!" you might say. "We should’ve configured `listen` to return the number of extra processes we spawned so that we could create exactly that many `receive` blocks. Then we wouldn’t have this problem."

And to you I say: No. Just. No.

Because I love you, and I want you to be happy in your coding. And Elixir feels the same way, and so has given us the `after` clause, which is what might be more familiarly called a timeout. It works like this:

{% highlight elixir %}
def start_receiving do
    receive do
        {_pid, token} ->
            IO.puts token
            start_receiving
    after
        1000 -> nil
    end
end
{% endhighlight %}

Here, if `start_receiving` doesn’t receive a matching message within 1 second, it will do what we’ve told it to do. Which, in this case, is nothing. The loop will end, and we’ll jump back into the `run` function, where `start_receiving` was the last call, and so the function will end. If we wanted, we could replace 1000 with a larger or smaller number, depending on how much latency we might expect. We can also use `0`, which will cause the loop to end as soon as there is no message to process.

Making that change and running your code, you'll get something that looks, at the end, like this:

    ...
    wizard
    wizard
    wizard
    wizard
    people
    people
    people
    wizard
    dear
    dear
    wizard
    dear
    dear
    wizard
    reader
    reader
    reader
    reader
    people
    people
    people
    people
    dear
    dear
    dear
    dear
    reader
    reader
    reader
    people
    people
    people
    dear
    dear
    dear
    reader
    reader
    reader
    people
    people
    :ok

Ok, great, so we can listen for an arbitrary number of responses, and stop listening once we stop receiving messages. There’s one more thing to look at. You may have noticed, in the previous output example, that the responses weren’t necessarily coming back in order. There were some “wizard” mixed in with the “people,” and some “reader” mixed in with the “dear”. Now, depending on the system you’re building, maybe that’s fine. Maybe it’s not important in what order the messages are received, just that they *are* received. But just for the sake of education, let’s say you did want this.

Let’s think back to our earlier example, where we were using the `^`-binding in the `receive` blocks to ensure a specific matching sending process. We got rid of this in `start_receiving` because we didn’t care about order, we just needed to match against a two-element tuple. So we need to get some of that logic back, but it would be a shame to give up this lovely recursion we’ve got going.

There may well be a better way to do this, but the best I’ve found is this particular combination of the two approaches. I’ll show you the code, then talk us through what’s going on:

{% highlight elixir %}
defmodule Echo do
  def listen do
    receive do
      {sender, token} -> send sender, {self, token}
    end
  end

  def run(tokens) do
    pids = Enum.map(tokens, fn token ->
      {spawn(Echo, :listen, []), token}
    end)
    Enum.map(pids, fn {pid, token} ->
      send pid, {self, token}
    end)

    Enum.map(pids, fn {pid, token} ->
      start_receiving(pid, token)
    end)
  end

  def start_receiving(pid, token) do
    receive do
      {^pid, ^token} ->
        IO.puts token
        start_receiving(pid, token)
    end
  end
end
{% endhighlight %}

This loops through the pids we created and starts a recursive receiver listening for messages sent from that specific process. Because they are created in order, they will listen in order, and we see that the blocks of identical token *do* now appear grouped

    iex> Echo.run([“wizard”, “people”, “dear”, “reader”])
    wizard
    wizard
    wizard
    wizard
    wizard
    wizard
    wizard
    people
    people
    people
    people
    people
    people
    people
    people
    people
    people
    people
    dear
    dear
    dear
    dear
    dear
    dear
    dear
    dear
    dear
    dear
    dear
    dear
    dear
    dear
    dear
    dear
    reader
    reader
    reader
    reader
    reader
    reader
    reader
    reader
    reader
    reader
    reader
    reader
    reader
    reader
    reader
    reader
    reader
    reader
    reader
    reader
    :done

### :done

[^1]: Apologies and credit to Douglas Adams
