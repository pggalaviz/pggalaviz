---
layout: post
title:  "Elixir releases with multi stage Docker builds"
author: Pedro G. Galaviz
date:   2019-12-13 12:00:00 -0600
comments: true
tags: [elixir, docker, phoenix]
---

If you've worked with Elixir for the last couple of years, you might be familiar
with OTP releases, what they are and how to generate them using libraries or
tools such as **Distillery**. Since Elixir version **1.9**, releases are now part of the
core language and in this post, we'll explore how to create one inside a Docker
container using multi-staged builds.

First lets remember what an OTP release is, from the **Erlang** docs:
> When you have written one or more applications, you might want to create a
> complete system with these applications and a subset of the Erlang/OTP
> applications. This is called a release.

So, a release is a single or multiple OTP applications packed as a standalone
system which is ready for distribution. On top of that, you can also bundle the
Erlang Runtime System (ERTS) so your release can run on a target machine without
the need of an Erlang or Elixir installation.
When you create a release, you should take into account that it must be build in
a machine that matches the target machine. This means that if I build the release
on a Linux machine it won't be able to run on a machine with Mac OS or Windows.

What are the benefits of producing an OTP release instead of running your
code in production mode on a server? These are some of them:

* The application is now self contained making its distribution much simpler.
* Application dependencies are already packed in the release, no need to fetch
  external resources.
* You don't need to provision a machine with a runtime, we're batteries
  included now.
* It's easier to connect a remote shell to a running release for instrospection.
* Favors explicitness of runtime vs build time configuration.
* You can easily control the BEAM VM flags, to configure how you want to run
  your application.

---

## Creating the App

In order to showcase Elixir's releases, first we need to create an application
we can work with. I'm assuming you already have **Elixir >= 1.9** and **Docker** installed
and running in your machine.

Let's install the latest **Phoenix** version (1.4.11 at the time of this
writing):

```shell
$ mix archive.install hex phx_new
```

Now lets create the project, on the path of your preference run:

```shell
$ mix phx.new exrelease --no-ecto
```

Select "Y" when prompted to install dependencies, and then type the following.

```shell
$ cd exrelease
$ mix phx.server
```

If everything goes well, you can visit `http://localhost:4000` in your browser
and you'll find the Phoenix homepage:

![Phoenix screenshot](/assets/img/posts/2019/vanilla_phoenix.png)

---

## Releasing the App

Now it's time to create the release, on the application path run:

```shell
$ mix release.init
```

This will generate some files in the `rel` folder (more on these later). Now we
should configure our release, to do this, create a `config/releases.exs` file,
this file an path is the default and will include its own runtime configuration.
Add the following to this file:

```elixir
# exrelease/config/releases.exs

import Config

secret_key_base = System.fetch_env!("SECRET_KEY_BASE")
app_port = System.fetch_env!("APP_PORT")

config :exrelease, ExreleaseWeb.Endpoint,
  server: true,
  http: [:inet6, port: String.to_integer(application_port)],
  secret_key_base: secret_key_base
```

The configuration provided in this file should be available at runtime
(when you run the release), unlike configuration found in
`config/dev.exs, config/test.exs, config/prod.exs` files, which is evaluated at
build-time (when you build the release).  As you can see, first we declare some
variables by calling `System.fetch_env!` function.  This function will fetch the
given environment variable or will rise an error if it can't be found and our
release won't be able to start.

Notice how we also added the `server: true` option to our
`ExreleaseWeb.Endpoint` configuration, this to instruct **Phoenix** to start our
endpoint.

Since we'll be fetching our configuration and secrets from this file, you can
safely delete the `config/prod.secret.exs` file, and remove the following line
from `config/prod.exs`:

```elixir
import_config "prod.secret.exs"
```

Its important to understand that we are still using the `config/prod.exs` file
to configure build-time options like `ExreleaseWeb.Endpoint` and other
dependencies for production.

To generate the first release and serve it, go to the project's path  and run
the following in your terminal:

```shell
$ mix phx.digest
$ MIX_ENV=prod mix release
$ APP_PORT=4000 SECRET_KEY_BASE=$(mix phx.gen.secret) _build/prod/rel/exrelease/bin/exrelease start
```

The first command will prepare your **CSS**, **JavaScript** and other assets for production.
The second one will create a release for the **prod** environment, if you go to
the `_build/prod` folder, you'll notice it contains a `rel` folder inside: this
is where our release lives. The third and last command, is setting our runtime
environment variables, notice that here we are generating the secret key by
running the **Phoenix** generator `mix phx.gen.secret`, and then we start the
release. If you go to `http://localhost:4000` you should see the same **Phoenix**
landing page, this time served completely by our release.

---

## Things to consider

Lets make a quick parentheses here, I've you've been working with Elixir for
some time you might have noticed some libraries or apps use **Module
attributes** as constants. Lets look at an example:

```elixir
defmodule MyModule do
  @my_constant Application.get_env(:my_app, :my_configuration_constant)

  # Rest of module...
end
```

Do you notice why this wont work with our release? Exactly! Module attributes
are set at compilation time, but since our environment variable is set at
runtime it doesn't yet exist when we generate the release, so this approach won't
work for us.

What can we do instead? Fortunately it's easy to solve it by generating a
private function that will return the same constant from configuration:

```elixir
defmodule MyModule do
  defp my_constant, do: Application.get_env(:my_app, :my_configuration_constant)

  def return_constant do
    my_constant()
  end
end
```

Now that we're aware of this, we shouldn't have any issues.

---

## Dockerizing the release

To build the real release we're going to user **Docker**, since we're interested
in running our containerized application in a cluster of **Linux** machines. We'll
use a multi-stage `Dockerfile`.

Add a `Dockerfile` to the root of the project and copy the following:

```bash
# ===========
# Build Stage
# ===========

FROM elixir:1.9.4 AS builder

# Set environment variables for building the application
ENV MIX_ENV=prod \
    TEST=1 \
    LANG=C.UTF-8

# Install Hex and Rebar
RUN mix local.hex --force && mix local.rebar --force

# Install Node 12.x
RUN apt-get update
RUN apt-get -y -q install curl apt-utils
RUN curl -sL https://deb.nodesource.com/setup_12.x | bash -
RUN apt-get install -y nodejs

# Create the app build directory
RUN mkdir /app
WORKDIR /app

# Copy over all the necessary application files and directories
COPY config ./config
COPY lib ./lib
COPY priv ./priv
COPY rel ./rel
COPY assets/package*.json ./assets/
COPY mix.exs .
COPY mix.lock .

# Fetch and cache Elixir dependencies
RUN mix deps.get --only $MIX_ENV
RUN mix deps.compile

# Fetch Node dependencies and add our assets
WORKDIR /app/assets
RUN npm install
ADD assets ./
RUN npm run deploy

# Prepare assets for production and build the release
WORKDIR /app
RUN mix phx.digest
RUN mix release

# =========
# App Stage
# =========

FROM debian:stretch AS app

ENV LANG=C.UTF-8

# Install openssl
RUN apt-get update && apt-get install -y openssl

# Copy the build artifact from the builder stage and create a non root user
RUN useradd --create-home app
WORKDIR /home/app
COPY --from=builder /app/_build .
RUN chown -R app: ./prod
USER app

# Run the release
CMD ["./prod/rel/exrelease/bin/exrelease", "start"]
```

The first stage will be the builder stage, here we install **Elixir** from the
official image, at the time of this writing the latest is `elixir:1.9.4`, then
we provide some steps to follow, you can get a grasp of whats going on from the
comments before each section.

After the build stage, we need to create the application container, the next
stage will do so. The new stage will be based on **Debian** (the official Elixir
image is too), because we have the Erlang Runtime System (ERTS) bundled in the
release, we don't need Erlang or Elixir anymore in this step. This will allow us
to have a much smaller image size.

With our new `Dockerfile`, go ahead and build the application image by running
the following from the project's path:

```shell
docker build -t exrelease .
```

If everything goes well and you run `docker images` you should see something similar to this:

```shell
REPOSITORY       TAG            IMAGE ID            CREATED             SIZE
exrelease        latest         d585f3effd7a        20 seconds ago      193MB
elixir           1.9.4          69ef33caf2c7        2 days ago          1.08GB
debian           stretch        5c43e435cc11        3 weeks ago         101MB
```

As you can see the `elixir` image is over **1 GB** while our application image
`exrelease` is less than **200 MB** in size. That's a big win from my point of
view.

Now, lets launch our new container, in your terminal run:

```shell
docker run --name exrelease --publish 4000:4000 --env SECRET_KEY_BASE=$(mix phx.gen.secret) --env APP_PORT=4000 exrelease:latest
```

If you visit `http://localhost:4000` you should see once again the
**Phoenix** home page, our containerized release is working!

---

## Start Scripts

If you previously worked with **Distillery** you might remember it provided a
**"boot hook"**, a feature that allowed to execute commands or run some code
when the app started (eg. run **Ecto** migrations). Since we're using `mix release`
now, we should change the way to accomplish this.

Remember that running `mix release.init` created some files in the `rel` folder?
Well, one of these is going to be helpful. `rel/env.sh.eex` will run whenever our
release starts, here we can call the commands we need.

For now lets asume we have **Ecto** in our dependencies, some migrations to
run, and a helper module that provides our release tasks:

```elixir
# lib/exrelease/release_tasks.ex
defmodule Exrelease.ReleaseTasks do
  @moduledoc false

  @required_apps [:crypto, :ssl, :postgrex, :ecto, :ecto_sql]
  @repos Application.get_env(:exrelease, :ecto_repos, [])

  def migrate do
    IO.puts("Running release tasks...")
    start_apps()
    run_migrations()
    stop_apps()
  end

  defp start_apps do
    # Ensure required apps for our tasks are running.
    Enum.each(@required_apps, &Application.ensure_started/1)
    # Use a pool_size of 2 for Ecto > 3.0
    Enum.each(@repos, & &1.start_link(pool_size: 2))
  end

  defp run_migrations do
    IO.puts("Running Ecto migrations...")
    Enum.each(@repos, &run_migrations_for/1)
  end

  defp stop_apps do
    IO.puts("Release tasks finished with success!")
    :init.stop()
  end
end
```

In order to run this functions, we should modify the `rel/env.sh.eex` file, at
the end of it add:

```shell
if [ "$RELEASE_COMMAND" = "start" ]; then
 echo "Starting release tasks..."
 ./prod/rel/exrelease/bin/exrelease eval "Exrelease.ReleaseTasks.migrate()"
fi
```

As you can see, we're using `eval` when the release is started with the `start`
command, if you look at the final line of our `Dockerfile` thats what we're
doing. We're passing `Exrelease.ReleaseTasks.migrate()` to `eval`, and by doing
that the `migrate/0` function will be called everytime our release is started.

Beware, this won't work until we add `ecto_sql` to our dependencies, once we do
that, configure **Ecto** and rebuild our image, we will see the `IO` lines from the `ReleaseTasks`
module printed on our terminal when we start the application.

The `rel/env.bat.eex` is used when generating a release for  **Windows**, and the
`rel/vm.args.eex` is used to provide any desired configuration VM flags passed to
the BEAM. For more information check: [mix release docs](https://hexdocs.pm/mix/Mix.Tasks.Release.html#content){:target="_blank"}.

---

## Conclusions

With Elixir 1.9, we are now able to build releases without the addition of any
external dependencies, which in my opinion is a good thing since it creates a
standard way to do it. Something I really enjoy is how simple it's to set build-time
vs runtime configuration and variables. All this plus the help of **Docker** to
create lightweight containers and an friendly way  to run start up scripts makes
application distribution much easier than it used to be.

I just want to thank all the amazing people who put their time and
effort to provide this amazing tools. Hope you find this post helpful, and as
always, if you have any observation or comment you're welcome to leave it below.
