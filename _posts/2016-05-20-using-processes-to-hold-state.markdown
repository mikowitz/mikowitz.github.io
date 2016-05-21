---
layout: post
title:  "Using Elixir Processes to Hold State"
date:   2016-05-20 16:51:00 -0400
categories: elixir processes
---

In the past two articles we've seen Elixir processes used to send and receive messages. Today I want to quickly look into another use for processes. That use is preserving state.

Because Elixir processes are so lightweight, it's trivial to spawn one and pass some data off to it. Then you can retrieve that process and read or update the data you've stored there. It's a cheap and isolated form of state mutability, without any worry of accidentally contaminating your code. And Elixir makes it just about as cheap as possible, via the `Agent` module.

## Agents

### Starting

To start an agent, all it takes is a call to `Agent.start/2` (or `Agent.start_link/2`, if you're building a supervision tree) and pass in a function defining your initial state, and an optional keyword list of arguments. Let say we're implementing a basic counter starting at 0, so we’d start with something like this:

    iex> {:ok, counter} = Agent.start(fn -> 0 end)

Successfully starting a process in Elixir returns a tuple of {status, pid}.

### Querying

Now we can query our counter to make sure it’s correctly started at 0:

    iex> Agent.get(counter, fn x -> x end)
    0

Here we're just asking for the value stored in our agent back without any modification. We can make this even shorter using Elixir's anonymous function shorthand:

    iex> Agent.get(counter, &(&1))
    0

### Updating

The corollary to `get` is `update`, and it works in much the same way, taking our agent as its first argument, and an updating function is its second. This function takes as its single argument the current state stored in our agent. For example, to increment our counter:

    iex> Agent.update(counter, &(&1 + 1))
    :ok

`update` returns only the status of the function, so we would have to `Agent.get` again to see our state.

    iex> Agent.get(counter, &(&1))
    1

Or would we?

We actually wouldn’t, because `Agent` gives us the function `get_and_update`, which takes the same arguments as `update`, with one small difference. The function we pass as the second argument must return a two-element tuple, of the form {state_before_update, state_after_update}. The return value of the call to `get_and_update` will the our state before the update, and the state after the update will now be stored in the agent.

    iex> Agent.get_and_update(counter, fn x -> {x, x + 1} end)
    2
    iex> Agent.get(counter, &(&1))
    3

You could also define the `get_and_update` handler function using shorthand as `&({&1, &1 + 1})`, but personally I find that starts to become harder to read quickly as the function becomes more complex. But that is purely a matter of taste, so do what you will.

It should be noted that neither `update` nor `get_and_update` need to perform any operations on the current value of the agent. For example, we could easily do

    iex> Agent.update(n, fn _ -> “Hello” end)

This would rather spoil our expectation of `n` as a counter, but it’s not illegal.

To wrap up, how about a brief real-world example?

## Well, how about it?

Ok, great. Let’s say you’re building a testing framework in Elixir, because you kind of miss [RSpec](http://rspec.info/) from Ruby but haven’t yet discovered the amazing [ESpec](https://github.com/antonmi/espec). So you need a way to store the count of passed and failed specs as your tests run, but you don't want to pollute your code by having to pass a map or keyword list around. You might write something like this:

```elixir
defmodule Tester do
  def start_run do
    Agent.start(fn -> [total: 0, passed: 0, failed: 0] end, name: :test_stats)
  end

  def record_pass do
    Agent.update(:test_stats, fn [total: t, passed: p, failed: f] ->
      [total: t + 1, passed: p + 1, failed: f]
    end)
  end

  def record_fail do
    Agent.update(:test_stats, fn [total: t, passed: p, failed: f] ->
      [total: t + 1, passed: p, failed: f + 1]
    end)
  end

  def report_results do
    stats = Agent.get(:test_stats, &(&1))
    IO.inspect stats
    Agent.stop(:test_stats)
  end
end
```

A couple new things here:

* In `start_run`, passing a `:name` as part of the `Agent.start` options allows you to reference the agent by that name elsewhere,
allowing us to not have to worry about setting a module- or project-level variable.

* `Agent.stop`, as it suggests, stops the process. Here, once the results are in, we want to make sure, the next time we run tests,
we don't append our results to the same stats as our previous run. `Agent.start` returns `{:error, {:already_started, pid}}` if
another live process with the same name exists.

Let's see how this works:

```elixir
iex> {:ok, agent} = Tester.start_run
iex> Tester.record_pass
iex> Tester.record_pass
iex> Tester.record_fail
iex> Tester.report_results
[total: 3, passed: 2, failed: 1]
iex> Process.alive?(agent)
false
```

Perfect!

Certainly there are less contrived examples in the world, but hopefully this is enough to give you a basic taste of how state can be managed using agents in Elixir.
