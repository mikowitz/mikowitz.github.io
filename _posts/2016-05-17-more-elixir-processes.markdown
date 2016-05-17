---
layout: post
title:  "More Elixir Processes"
date:   2016-05-17 10:56:00 -0400
categories: elixir processes
---

Last time, we looked at the basic mechanisms of Elixir processes, using them to pass a series of messages, and to respond, both in the order they were received and out of order. Today, we’re looking at the next step: passing messages back and forth between more than one process; and the next step after that: doing all of this across a network.

To do this, we’re going to implement the simplest form of this sort of setup: a chat room. Boring? Perhaps. Clear and effective for our purposes here? Hopefully. Let’s find out!

For a chat room to work, we need two pieces: server and client. The server is responsible for receiving messages from users and passing any responses back to the users. The client manages the connection status of a user, and allows a user to send messages to be posted in the chat room.

Let’s start with the client. For the first, most basic implementation, the client needs to handle two things: joining a chat room and posting a message to that room.

    defmodule Client do
        def join_room(room, username) do
            spawn(__MODULE__, :start, [room, username])
        end

        def start(room, username) do
            send(room, {self, :join, username})
            loop(room, username)
        end

        def loop(room, username) do
            receive do
                {sender, :message, message} ->
                    IO.puts “#{username}’s client: #{message}”
                    loop(room, username)
            end
        end
    end

In `.join` we use `spawn/3` to start up a new process that calls the method `start` on the `__MODULE__` (`Client` in this case), and passes two arguments to the method, `room` and `username`. `start` does two things: it sends a message notifying the new client has joined the chat room, and then begins the `loop` method. `loop` will currently do nothing, since we have no way for the server to send messages back, but for when that day comes, we have it ready to listen for one type of message matching `{sender, :message, message}`. It will print the message (with a prefix to remind us whose client we’re looking at), and then call `loop` again. Remember from last time that, as long as the recursive call is the *very* last thing the function does, we do not need to worry about overflowing the stack.

Let’s take this for a quick spin in IEx, just to assure ourselves this will work:

    iex> c “client.ex”
    iex> client = Client.join_room(self, “Michael”)
    iex> send client, {self, :message, “Hello”}
    “Michael’s client - Hello”

Ok, fantastic. Let’s take a look at the server.

    defmodule Server do
        def start do
            spawn(__MODULE__, :loop, [[]])
        end

        def loop(clients) do
            receive do
                {sender, :join, username} ->
                    broadcast(“#{username} has joined the room”, clients)
                    new_clients = [{sender, username}|clients]
                    loop(new_clients)
                end
            end
        end

        def broadcast(message, clients) do
            Enum.each(clients, fn {pid, _} ->
                send pid, {self, :message, message}
            end)
        end
    end

`start` spawns another process, calling `Server.loop(clients)` with an empty list as its argument. The `loop` function is set up to handle the join message the Client defines above. Let’s take it for a spin.

    iex> c “server.ex”
    iex> c “client.ex”
    iex> s = Server.start
    iex> c1 = Client.join_room(s, "Michael") # Will not print anything since this is the first client to join
    iex> c2 = Client.join_room(s, "Mikey")
    Michael's client: Mikey has joined the room
    iex> c3 = Client.join_room(s, "Mick")
    Mikey's client: Mick has joined the room
    Michael's client: Mick has joined the room

There we go!

Let’s add one more message receiver, to let a user send a message to the chat room. Here’s what it will look like in the `Client`

    defmodule Client do
        …
        def post_message(client, room, message) do
            send room, {client, :message, message}
        end
        …
    end

and in the `Server`

    defmodule Server do
        …
        def loop(clients) do
            receive do
                {sender, :join, username} ->
                    broadcast(“#{username} has joined the room”, clients)
                    new_clients = [{sender, username}|clients]
                    loop(new_clients)
                {sender, :message, message} ->
                    {_, sender_name} = Enum.find(clients, fn {pid, _} -> pid == sender end)
                    broadcast(“#{sender_name} says: #{message}”, clients)
                    loop(clients)
            end
        end
    end

Now, when a client posts a message, the server can receive it and send it out to all connected clients. First, it finds the client who sent the message, matching on the PID, so that it can broadcast the message from the correct client. Then, it send that message back out to all the connected clients. Let’s put this into action:

    iex> s = Server.start
    iex> c1 = Client.join_room(s, "Michael")
    iex> c2 = Client.join_room(s, "Mikey")
    iex(6)> Client.post_message(c1, s, "Hello!")
    Mikey’s client: Michael says: Hello!
    Michael's client: Michael says: Hello!

### :boom:

Now, this is all well and good, but it’s all happening inside the same IEx session, so maybe this isn’t quite so magical after all. I mean, what good is a chat room when all the users have to be in the same place? Might as well just have a conversation. So let’s trying something a bit different:

Fire up to IEx sessions based in the same directory. In order for them to be able to communicate, we’ll need to give them names, so the command will look something like

    ~ iex —sname room1

and, in a different window

    ~ iex —sname room2

or whatever more exciting names you come up with for the rooms. In your IEx now, your prompt should look something like

    iex(room1@your-computers-local-name)>

load up the server code in one instance, and start a server

    iex(room1)> s = Server.start
    iex(room1)> Process.register(s, :chatroom)

`Process.register/2` takes a PID and an atom, and associates the name with the PID, so that it can be more easily referenced. This will come in handy in just a moment, when you join the chatroom with a client in your other IEx instance:

    iex(room2)> server = {:chatroom, :”room1@your-computers-local-name”}
    iex(room2)> c = Client.join_room(server, “Michael”)

By creating a 2-element tuple with the name of the process and the name of the instance it is running in, you can use that in place of a PID to reference a process running in another instance.

Now, let’s go crazy and start up a 3rd IEx console

    ~ iex —sname room3

    iex(room3)> server = {:chatroom, :”room1@your-computers-local-name”}
    iex(room3)> c2 = Client.join_room(server, “Mikey”)

Back in room2, you should see a message that Mikey has joined the chat room. Definitely more magic going on this time! For an additional dose of wizardry, load this code on two computers on the same network, and, using the trick to naming rooms and processes, connect two clients on two different computers to the same server and watch everything work!

Not a whole lot new here, necessarily, beyond seeing how simple distributed computing is in Elixir (which makes sense, since that was one of the main driving factors behind the development of Erlang, and thus Elixir). In the next installment, we’re going to look at a piece of the process puzzle I’m still working on understanding: what happens when processes crash.
