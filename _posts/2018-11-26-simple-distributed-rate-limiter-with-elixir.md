---
layout: post
title:  "Simple distributed rate limiter with Elixir"
author: Pedro G. Galaviz
date:   2018-11-26 12:00:00 -0600
comments: true
tags: [elixir, distributed systems]
---

While developing Elixir web apps you'll find there are some activities or
requests that can be way more expensive than others, something like calling an
external API, sending a password reset email, a CPU intensive task or any other
that fits your business model.

This is where a rate limiter comes handy, most web apps have a centralized rate
limiter using technologies like **Redis**, **Memcached** or some algorithms
such as
<a href="https://en.wikipedia.org/wiki/Token_bucket" target="_blank">Token Bucket</a>
or
<a href="https://en.wikipedia.org/wiki/Generic_cell_rate_algorithm" target="_blank">Generic cell rate algorithm</a>,
but we're using Elixir and with it we have different posibilities:

## Distributed Elixir

Let's make it clear, stateful distributed systems are hard to implement and
debug, even with the **BEAM**, which give us a set of tools to make it way easier.
Since Elixir runs on top of it we can take advantage of these and try to get
some distributed functionality without the need of external technologies or
different languages.

Currently there are several **hex** packages that make Elixir nodes clustering
eaiser, for our implementation we'll use
<a href="https://github.com/bitwalker/libcluster" target="_blank">libcluster</a>,
which support different strategies like **empd** (Erlang Port Mapper Daemon),
Multicast UDP gossip or Kubernetes.

Once we solve our clustering strategy we'll need a way to run a distributed rate
limiter, there are some **hex** packages out there such as
<a href="https://github.com/ExHammer/hammer" target="_blank">Hammer</a>,
which can only handle a local data storage or use **Redis**, neither is what we're
looking for so we'll create our own solution. As the title says, we'll keep it
simple.

## Helper Libraries

Having a cluster of nodes conected is just part of running our distributed system,
but what about net split healing? dynamic cluster size? supervised distributed
processes? Fortunately there are some **hex** packages to help with this:

<a href="https://github.com/bitwalker/swarm" target="_blank">Swarm</a>,
which is probably the most famous in the Elixir community, and the newcomer
<a href="https://github.com/derekkraan/horde" target="_blank">Horde</a> which is
based on **Swarm** but makes it easier to maintain a separation between a global
supervisor from a global registry and is based on delta-CRDTs
(conflict-free replicated data types).

For our implementation we'll use **Horde** since it makes it easier (in my
opinion) to know what's going on, the drawback is that it's less mature than
**Swarm**, though it looks very promising.

## The implementation

Let's get started, assuming you have Elixir > 1.5 already installed create a new application:

{% highlight shell %}
mix new limiter --sup
cd limiter
{% endhighlight %}

Open the project folder on your favorite text editor and add our dependencies:

{% highlight elixir %}
# mix.exs
# ...
defp deps do
  [
    {:libcluster, "~> 3.0"},
    {:horde, "~> 0.3.0"}
  ]
end
{% endhighlight %}

Install them with `mix deps.get` and we're all set to start our
implementation.

Lets start by configuring **libcluster** for our development environment (for
production and test environments refer to the package documentation), for
simplicity we'll use the **epmd** strategy and nodes with fixed names:

{% highlight elixir %}
# config/config.exs
# ...
import_config "#{Mix.env()}.exs"


# config/dev.exs
use Mix.Config

config :libcluster,
  topologies: [
    dev: [
      strategy: Cluster.Strategy.Epmd,
      config: [
        hosts: [
          :"a@127.0.0.1",
          :"b@127.0.0.1",
          :"c@127.0.0.1"
        ]
      ]
    ]
  ]
{% endhighlight %}

Since we're using Elixir > 1.5, I'll separate my main supervisor from my application
file, let's update it:

{% highlight elixir %}
# lib/limiter/application.ex
defmodule Limiter.Application do
  @moduledoc false
  use Application

  def start(_type, _args) do
    Limiter.Supervisor.start_link([])
  end
end
{% endhighlight %}

And now lets create the `supervisor.ex` file:

{% highlight elixir %}
# lib/limiter/supervisor.ex
defmodule Limiter.Supervisor do
  @moduledoc false
  use Supervisor

  def start_link(args) do
    Supervisor.start_link(__MODULE__, args, name: __MODULE__)
  end

  def init(_args) do

    children = [
      cluster_supervisor(),
      {Horde.Registry, [name: Limiter.GlobalRegistry, keys: :unique]},
      {Horde.Supervisor, [name: Limiter.GlobalSupervisor, strategy: :one_for_one]},
      %{
        id: Limiter.HordeConnector,
        restart: :transient,
        start: {
          Task, :start_link, [
            fn ->
              Limiter.HordeConnector.connect()
              Limiter.HordeConnector.start_children()
            end
          ]
        }
      }
    ]
    |> Enum.reject(&is_nil/1)

    Supervisor.init(children, strategy: :one_for_one)
  end

  # Run libcluster supervisor if configuration is set.
  defp cluster_supervisor() do
    topologies = Application.get_env(:libcluster, :topologies)

    if topologies && Code.ensure_compiled?(Cluster.Supervisor) do
      {Cluster.Supervisor, [topologies, [name: Limiter.ClusterSupervisor]]}
    end
  end
end
{% endhighlight %}

A lot is going on here, first we load the topologies configuration
from our config file if one exists, if found we'll start our
cluster supervisor (part of libcluster) with the name of:
`Limiter.ClusterSupervisor`.

Then we'll start our global supervisor and our global registry (part of Horde),
with names `Limiter.GlobalSupervisor` and `Limiter.GlobalRegistry` respectively.

Finally we'll start a `Task` that calls two function from the
`Limiter.HordeConnector` module, which we'll create next.

After we have the supervisor's children set we can call the `init` function.

Now let's create the `horde_connector.ex` file:

{% highlight elixir %}
# lib/limiter/horde_connector.ex
defmodule Limiter.HordeConnector do
  @moduledoc false
  require Logger

  def connect() do
    Node.list()
    |> Enum.each(fn node ->
      Logger.debug(fn ->
        "[limiter on #{inspect(Node.self())}]: Connecting Horde to #{inspect(node)}"
      end)
      Horde.Cluster.join_hordes(Limiter.GlobalRegistry, {Limiter.GlobalRegistry, node})
      Horde.Cluster.join_hordes(Limiter.GlobalSupervisor, {Limiter.GlobalSupervisor, node})
    end)
  end

  def start_children() do
    Horde.Supervisor.start_child(Limiter.GlobalSupervisor, Limiter.RateLimiter)
  end
end
{% endhighlight %}

Here we declare two functions, the first one `connect/0` connects every `GlobalSupervisor`
and `GlobalRegistry` when the app starts, with the ones on other nodes of the cluster.
For more info on this, check **Horde**'s documentation.

The second one `start_children/0` is where we start the processes we want our
`GlobalSupervisor` to supervise, in this case just our `Limiter.RateLimiter`
module, which will be a GenServer:

{% highlight elixir %}
# lib/limiter/rate_limiter.ex
defmodule Limiter.RateLimiter do
  @moduledoc false
  use GenServer
  require Logger

  @max_per_minute 2
  @clear_after :timer.seconds(60)
  @table :rate_limiter

  # ==========
  # Client API
  # ==========

  def start_link(_opts) do
    GenServer.start_link(__MODULE__, [], name: via_tuple())
  end

  def log(ip) do
    case Horde.Registry.whereis_name({Limiter.GlobalRegistry, __MODULE__}) do
      :undefined ->
        {:error, :not_found}
      pid ->
        node = Kernel.node(pid)
        :rpc.call(node, __MODULE__, :get_log, [ip])
    end
  end

  def get_log(ip) do
    case :ets.update_counter(@table, ip, {2, 1}, {ip, 0}) do
      count when count > @max_per_minute -> {:error, :rate_limited}
      _count -> :ok
    end
  end

  # ================
  # Server Callbacks
  # ================

  def init(_opts) do
    Logger.info("[limiter on #{inspect(Node.self())}][RateLimiter]: Initializing...")
    :ets.new(@table, [:set, :named_table, :public, read_concurrency: true, write_concurrency: true])
    schedule_clear()
    {:ok, %{}}
  end

  def handle_info(:clear, state) do
    Logger.info("[limiter on #{inspect(Node.self())}][RateLimiter]: Clearing table...")
    :ets.delete_all_objects(@table)
    schedule_clear()
    {:noreply, state}
  end

  # =================
  # Private Functions
  # =================

  defp schedule_clear do
    Process.send_after(self(), :clear, @clear_after)
  end

  defp via_tuple do
    {:via, Horde.Registry, {Limiter.GlobalRegistry, __MODULE__}}
  end
end
{% endhighlight %}

As you can see, when calling the `start_link/1` function, we register our
GenServer with a name returned by the `via_tuple/0` private function, this is
how **Horde** registers the process in our `GlobalRegistry`.

In the module attributes we can see that there's a limit of 2 every 60 seconds.

Our storage happens on an **ETS** table with name `:rate_limiter` which is
cleared after the given number of seconds. All handled by GenServer's messaging,
nothing new here.

The main functionality is provided by the `log/1` function, which receives a
unique key to identify who is logging to the table, you can use a user's ID or
an IP address, etc. for our implementation we'll use the IP address.

In this function, first we need to know the PID of our `RateLimiter` process,
which owns the **ETS** table, for this we can use the `whereis_name/1` function
provided by **Horde** (at the moment of this writing this function doesn't appear in
the documentation).

If a PID is returned we can make an RPC (Remote Procedure Call) after knowing
which node from our cluster owns the process.

The `get_log/1` function will be called on the node that owns the GenServer and
ETS, returning `:ok` or `{:error, :rate_limited}` if we try to log 3 or more
times in 60 seconds.

Let's try our app now, on 3 different shell sessions run:


{% highlight shell %}
iex --name a@127.0.0.1 -S mix

iex --name b@127.0.0.1 -S mix

iex --name c@127.0.0.1 -S mix
{% endhighlight %}

This will start and connect our 3 nodes with each other. You can use Erlang's
observer `:observer.start()` to see there's only one `Limiter.RateLimiter` process running across
the cluster, if you kill or gracefully terminate the owner node, you'll se
another node will immediately start a new one thanks to our `GlobalSupervisor`.

The next figure shows the functionality we can expect: we successfully call the
`log/1` function and the third time we get an error, after the table is cleared
we can try again, receiving the same behavior.

{% highlight shell %}
iex(a@127.0.0.1)1> Limiter.RateLimiter.log("127.0.0.1")
:ok
iex(a@127.0.0.1)2> Limiter.RateLimiter.log("127.0.0.1")
:ok
iex(a@127.0.0.1)3> Limiter.RateLimiter.log("127.0.0.1")
{:error, :rate_limited}
18:34:03.912 [info]  [limiter on :"a@127.0.0.1"][RateLimiter]: Clearing table...
iex(a@127.0.0.1)1> Limiter.RateLimiter.log("127.0.0.1")
:ok
iex(a@127.0.0.1)2> Limiter.RateLimiter.log("127.0.0.1")
:ok
iex(a@127.0.0.1)3> Limiter.RateLimiter.log("127.0.0.1")
{:error, :rate_limited}
{% endhighlight %}

Notice we don't have to worry where our rate limiter was started, we can call the
`log/1` function from any node without caring where in the cluster the **ETS** lives.

We now have a simple rate limiter working. If we use **Phoenix** we can limit
some actions like this:

{% highlight elixir %}
# in some controller
# ...
def my_action(conn, params) do
  ip_address = conn.remote_ip
    |> Tuple.to_list()
    |> Enum.join(".")

  with \
    :ok <- Limiter.RateLimiter.log(ip_address),
    {:ok, result} <- MyApp.my_expensive_action(params)
  do
    {:ok, result}
  else
    {:error, :rate_limited} ->
      conn
      |> put_status(:too_many_requests)
      |> put_view(ErrorView) # since phoenix 1.4
      |> render("429.json", reason: "Rate limit reached")
      |> halt()
    other ->
      # Handle other errors
  end
end
{% endhighlight %}

And we're done with our simple rate limiter example, a working app can be found
in my
<a href="https://github.com/pggalaviz/limiter" target="_blank">Github</a>.

## Conclusion

This experiment shows us that Elixir, most times, can help us to get solutions
without requiring external dependencies or writing an implementation on
another language.

This rate limiter is obviously very basic and has is caveats, for example if
there's a topology change, **Horde** can termiante and restart our **GenServer** and
**ETS** on another node, losing its state. For basic functionality where
you can afford to lose state from time to time, this solution can work.
However if you can't afford this, maybe **Redis** or another technology is what
you're looking for.

Also keep in mind that having our rate limiter on a remote node will add latency
to our request. As always pros and cons should be evaluated according to our
business model.

If you have any observation or something to add please don't hesitate and leave
a comment below.
