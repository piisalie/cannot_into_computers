---

title: Elixir Runtime Configuration
date: 2017-11-18 00:30 UTC
tags: elixir, deployment, distillery, other

---

tldr: Don't want to configure / learn a new external library just to use a dang environment variable in your release? This is for you: skip to the Solution section.


### Context

You've started your first Elixir web application; it's functional and fast and delightful and you're ready to deploy. The Internet suggests you [always build a release](https://elixirforum.com/t/always-use-releases/4573) for production using Distillery (they're right btw because Distillery is awesome). It seems to be going smoothly but your environment variable configuration does not work quite as expected. You return to the docs and notice this section [outlining your problem](https://hexdocs.pm/distillery/getting-started.html#application-configuration). It helpfully offers a few solutions:

* Set the special `REPLACE_OS_VARS=true` and add you variable to `vm.args`: `-my_app key #{ENV_VAR}`
* Use a library ([Conform](https://github.com/bitwalker/conform), [Flasked](https://github.com/asaaki/flasked), etc)

Both options are great solutions. However, you've already spent time learning and adding one additional tool (Distillery) in order to build a release. If it's a side project, or a small tool for work adding an industrial grade library or taking time to learn what a `vm.args` file is seems out of reach. These feelings are valid; you just want it to work. If this sounds familiar then you're in the right place.


### Solution

Good news! It is possible to use the Application `start` function to load environment variables into the [Application config](https://hexdocs.pm/elixir/Application.html#module-application-environment). IMPORTANT CAVEAT FROM THAT DOCUMENTATION:

> Keep in mind that each application is responsible for its environment. Do not use the functions in this module for directly accessing or modifying the environment of other applications (as it may lead to inconsistent data in the application environment).

The steps:

1. Find your Application's Application
1. Add runtime configuration to the `start/2` function
1. ??? (refactor probably)
1. profit


#### Find your Application's Application

In a Phoenix Application or Mix Application generated with `--sup` look for: `.lib/application_name/application.ex`

In an umbrella application, each Application should be responsible for its own config.

In a project you built yourself a week ago that has no discernible file structure because you were curious (yes we all do this): it is whatever the `mod` atom references in your `mix.exs` `application` function. (eg: `mod: {Bananas.Application, []}` means you should find the `Bananans.Application` module)


#### Add Runtime Configuration

There are two key parts to this step:

* `Application.put_env/3` puts a named variable directly into the Application config (similarly to the `config/config.exs` file)
* `System.get_env/1` This actually gets the plain string environment variable.

Locate the `start` function and add a line like `Application.put_env(:some_app, :port, System.get_env("PORT"))`

Where `:some_app` is the name of the Application you're using the variable in, and `:port` is what you'd like the variable named in Elixir land.

Now you may use `Application.get_env(:some_app, :port)` wherever you need to use the value of the `PORT` environment variable. It is generally a good idea to put defaults in the config file to at least document what environment vars you're expecting and depending on.

You may also do any sort of parsing on the loaded variable (since `System.get_env/1` will return a string). For instance if you need to coerce it into an Integer or split it on `,` you can do that before putting the variable in the Application env. Don't get too fancy though.


#### ??? (refactor probably)

The solution above will work when you have only a few environment variables, but as an application grows (and it will grow) it will require better ways to manage complexity. One possibility is extracting the configuration to a module and then calling the appropriate function for the Application from `start`. This also gives us a place to raise errors and keep the application from starting if an environment variable is missing.

```elixir
defmodule MyApp.Configuration
  def build(:my_app = app_name) do
    Application.put_env(app_name, :port, get_port_var())
  end

  def build(:some_other_app = app_name) do
    Application.put_env(app_name, :var_1, System.get_env("VAR1"))
  end

  defp get_port_var do
    "PORT"
    |> System.get_env
    |> String.to_integer
  end
end
```

And use it in the Application `start` function:

```elixir
  def start(_type, _args) do
    MyApp.Configuration.build(:my_app)

    children = [
      # ...
    ]

    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end
```


### Closing

Is this a viable longterm solution? The answer to that depends. You can benefit from less cognitive overhead and inital effort using this method; it's just plain Elixir. However, the cost of configuration may become unwieldy later down the road.
