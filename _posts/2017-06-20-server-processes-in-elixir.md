---
layout: post
title:  "Server processes in Elixir"
author: Pedro G. Galaviz
date:   2017-06-20 08:00:00 -0600
comments: true
tags: [elixir]
---

Elixir is a functional programming language with built in concurrency based on the actor model, this creates an
easy way to parallelize work as a side effect. The main unit for achieving
concurrency are Elixir's processes. These are not OS processes, they are
lightweight units with separate contexts, they don't share any memory, they run
isolated from each other, which leads to fault tolerance: if a process crashes
it shouldn't affect other processes, our app or the whole system.

Processes can be created with the `spawn/1` function, they can communicate with
other processes by sending and receiving messages, these are stored in the
process' mailbox, and processes can compute them in the order they
arrived.

The simplest process will be created, it will compute it's function and then
it will  be destroyed, allowing garbage collector to clean the memory the
process once occupied. Elixir programs can and often run millions of these processes.

Sometimes we need processes to run longer, theoretically forever, these
processes are created just as any other process, but they relay on recursion to
keep running even when they've computed their function once. These type of processes
are called **Server Processes**.

## Enter Server Processes

Server processes allow our programs to have an inmemory state, this state can be
a constant or it can be mutated (even the state is mutated, functional
programming and immutability characteristics are working behind). Let's see this
in action by creating a simple CLI Todo List (Elixir/Erlang have a helper
for implementing Server Processes called **GenServer** but we won't use it
here).

First let's create a file called `todo.exs` and on this file we'll create 2
modules: **TodoServer** and **TodoList**. The first one will have our server
process implementation, and the sencond one will have our data abstraction (Todo
Lists).

{% highlight elixir %}
# todo.exs
defmodule TodoServer do
  ...
end

defmodule TodoList do
  ...
end
{% endhighlight %}

## The Todo List

For the sake of simplicity I'll add the **TodoList** implementation, comments
should help understand what is going on.

It's always recommended to keep our data abstractions in a separate module, and
to follow all functional programming workflow.

{% highlight elixir %}
# todo.exs

...

defmodule TodoList do
  # Define a struct, todos will be collected on a Map
  # A numerical ID will be given to each todo :next
  defstruct todos: Map.new, next: 1

  # Each todo item will be a Map, this is how it should look:
  # %{date: {2017, 06, 20}, title: "This is a todo!"}

  # Initialize a new Todo List, a list with todos can be entered
  # defaults to a blank list and adds each element to the struct: %TodoList{}
  def new(todos \\ []) do
    Enum.reduce(todos, %TodoList{}, &add_todo(&2, &1))
  end

  # Add a new item to the list
  def add_todo(%TodoList{todos: todos, next: next} = list, todo) do
    # Add  the next numerical ID to the entry
    todo = Map.put(todo, :id, next)
    new_todos = Map.put(todos, next, todo)
    # Return updated list and updates next ID number
    %TodoList{list | todos: new_todos, next: next + 1}
  end

  # Return all todos with specific 'date'
  def todos(%TodoList{todos: todos}, date) do
    todos
    |> Stream.filter(fn({_, todo}) ->
        todo.date == date
      end)
    |> Enum.map(fn({_, todo}) ->
        todo
      end)
  end

  # Update an existing todo, an ID and updater function is required
  def update_todo(%TodoList{todos: todos} = list, todo_id, updater) do
    case todos[todo_id] do
      nil -> list
      old_todo ->
        # Call the updater function on the entry
        new_todo =  updater.(old_todo)
        # Update Todo List with the updated entry
        new_todos = Map.put(todos, new_todo.id, new_todo)
        # Return updated list
        %TodoList{list | todos: new_todos}
    end
  end

  # Delete a specific todo from list, requires an ID
  def delete_todo(%TodoList{todos: todos} = list, todo_id) do
    new_todos = Map.delete(todos, todo_id)
    # Return updated list
    %TodoList{list | todos: new_todos}
  end
end
{% endhighlight %}

## The Server Process

Now we can implement our Server Process. It's easier if we divide it in two
parts:

### Client

The client section will have all the functions that are accesible out of our
module, we can call these functions from another process (such as IEx) to get
the server functionality.

Let's start by creating the `start` function, when called, this function will create a new
process, call the `loop` function (we'll create it later) and will delegate to the **TodoList** module the creation of a new list
as the starting server's state. Calling this function will return a **PID**
(Process ID) that we can keep in a variable for using it later: `server_pid =
TodoServer.start`.

{% highlight elixir %}
# todo.exs
defmodule TodoServer do
  def start do
    spawn(fn -> loop(TodoList.new) end)
  end
end
...
{% endhighlight %}

Now let's create the `add_todo` function, this one will send a message to the
server's mailbox with the following structure: `{:add_todo, todo}`. We can
send a new entry to our previously created server by calling:

{% highlight elixir %}
iex(1)> TodoServer.add_todo(server_pid, %{date: {2017, 06, 20}, title: "I'm a new
entry!"})
{% endhighlight %}

The function should look like this:

{% highlight elixir %}
# todo.exs
defmodule TodoServer do
  ...
  def add_todo(server_pid, todo) do
    send(server_pid, {:add_todo, todo})
  end
end
...
{% endhighlight %}

In the same way we can create the `update_todo` and the `delete_todo`
functions:

{% highlight elixir %}
# todo.exs
defmodule TodoServer do
  ...
  def update_todo(server_pid, todo_id, updater_function) do
    send(server_pid, {:update_todo, todo_id, updater_function})
  end
  def delete_todo(server_pid, todo_id) do
    send(server_pid, {:delete_todo, todo_id})
  end
end
...
{% endhighlight %}

Notice that to call these we need to pass parameters such as the todo ID or the
updater function.

All these previous functions can help us modify our Todo List instance
asynchronously, let's say we call them from **iex**, even they spend 10 senconds
to complete, our **iex** process won't be blocked and we would be able to
continue working as spected, the work is happening concurrently on the Server
Process.

Now, let's implement 2 different functions, these will run synchronously because
they'll expect a response from the server. To run something synchronously in
Elixir, we should manually create the functionality using it's asynchronous
message passing. Lets say process **A** needs a response from server **B**, then
**A** should send a message to **B** wich will compute what requested and then
send **A** another message with the response. While **B** is working, **A** is
blocked waiting for a response. This is exactly what we'll do.

The `todos` function will return all entries with the same date (passed in the
arguments), while the `list` function will return our complete Todo List instance. Both
functions when called will send a message to the server, however the caller's
PID (such as IEx's) will be passed via the `self/0` function.

Just after these functions are called and the message to the server is sent, they wait to `receive` a response message and
then return that message to the caller process. These functions will block the
caller (remember they run synchronously) until a response is received. In order
to rescue our caller process from a permanent blocking if a failure occurs or
the message passed can't be pattern matched we can use the `after` clause and
return an error as the message.

Functions should look like this:

{% highlight elixir %}
# todo.exs
defmodule TodoServer do
  ...
  def todos(server_pid, date)
    # Here self() returns the PID of the caller
    send(server_pid, {:todos, self(), date})
    receive do
      {:todo_entries, entries} -> entries
    after 5000 ->
      {:error, :timeout}
    end
  end
  def list(server_pid)
    # Here self() returns the PID of the caller
    send(server_pid, {:list, self()})
    receive do
      {:todo_list, list} -> list
    after 5000 ->
      {:error, :timeout}
    end
  end
end
...
{% endhighlight %}

The caller PID should be passed so our server knows where to send the response,
notice that the `after` clause will return a timeout error after 5 seconds if no
response is received, this will unblock our caller process in case something
goes wrong.

That's the first part of our Server Process, let's add the second one.

### Server

The server part is where our implementation lives, these functions should be
hidden to the clients and the only way to interact with them should be client
section's functions. That's why all these functions will be declared as private
with the `defp` keyword.

Let's start by creating the most important function on our server, the `loop`
function. This function is the one that keeps our **Server Process** running
continuously, keeps the Todo List instance as the server's state and receives all
server's messages, it then delegates messages computation to helper functions.

Let's create it:

{% highlight elixir %}
# todo.exs
defmodule TodoServer do
  ...
  defp loop(current_list) do
    new_list = receive do
      message ->
        process_message(current_list, message)
    end
    loop(new_list)
  end
end
...
{% endhighlight %}

A lot is happening here, first notice the function receives a paramenter which
contains the Todo List. Remember that when we started our server
`TodoServer.start` the initial list is just a struct created by `TodoList.new`.

When the `loop` function is called it will wait to `receive` a message, when
received it will delegate the computing to the `process_message` function (which
we'll implement next) passing the current state (the current Todo List) and the
message received.

The response from this function is then saved to the `new_list` variable, and
when we have this variable we can then use recursion to call `loop` again with
the new Todo List, updating our server's state.

Notice we're using tail recursion here, so calling `loop` again won't need more
memory, and while the process is waiting for a message no CPU resources are
used, making Elixir's processes very cheap to create and use.

Now let's implement the `process_message` function, remember our client
implementation can send different types of messages, if we want to create an
entry the `{:add_todo, todo}` message is passed, and if we want the full list
we pass the `{:list, self()}` message. So we need our function to compute what
we expect according to the message it received, and for this we'll use different
clauses:

{% highlight elixir %}
# todo.exs
defmodule TodoServer do
  ...
  defp process_message(todo_list, {:add_todo, new_todo}) do
    TodoList.add_todo(todo_list, new_todo)
  end
  defp process_message(todo_list, {:update_todo, id, updater}) do
    TodoList.update_todo(todo_list, id, updater)
  end
  defp process_message(todo_list, {:delete_todo, id}) do
    TodoList.delete_todo(todo_list, id)
  end
  defp process_message(todo_list, {:todos, caller, date}) do
    send(caller, {:todo_entries, TodoList.todos(todo_list, date)})
    todo_list
  end
  defp process_message(todo_list, {:list, caller}) do
    send(caller, {:todo_list, todo_list})
    todo_list
  end
  defp process_message(todo_list, _) do
    todo_list
  end
end
...
{% endhighlight %}

All our clauses receive the current list as the `todo_list` parameter, and a
different message, by pattern matching the message we know exactly what to
compute. The last implementation returns the current list in case an unknown
message is passed. Notice how most of the inner functionality is delegated to
the **TodoList** module, which is in charge of the data abstraction.

It's  very importat that every implementation of the `process_message` function returns a new or the same Todo List,
because remember our `loop` function will expect it and pass it to the recursion
call.

## Conclusion

This is how basic **Server Processes** work in Elixir, we can start multiple
`TodoServer` processes and each will have a different Todo List as it's state
and work concurrently.

The **OTP** helper **GenServer** is an abstraction of this, with less
boilerplate and more functionality, but it relays on recursion as well to keep
the process running continuously. That's why it's important to know how
**Server Processes** work internally and their pitfalls, like becoming a
bottleneck for our app, but that's something for another post.

You can now start an **iex** session in your terminal with our `todo.exs` file
and start using our **Server Processes**.

{% highlight elixir %}
iex todo.exs

iex(1)> server_pid = TodoServer.start
{% endhighlight %}
