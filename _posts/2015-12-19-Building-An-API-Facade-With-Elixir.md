---
layout: post
title: Building an API Facade with Elixir
date: 2015-12-19
tag: markdown
blog: true
#star: true
---

Coming from Ruby and growing a bit tired of resorting to caching every time things get a bit more complicated than a simple CRUD application, I’ve been meaning to try [Elixir](http://elixir-lang.org/) and its web framework [Phoenix](http://www.phoenixframework.org/) for a while. 

[Elixir](http://elixir-lang.org/) is the new cool kid in the block. It’s built on top of the Erlang and uses most of its concurrency and failure handling models.

Needless to say, it’s blazingly fast!

### Building API’s with phoenix

With the market revolving more and more around smartphone and multi-client applications, I’ve been writing API based applications for a while now.

Building an API with Phoenix is quite straightforward and, if you came from
Rails, you'll find many of the same concepts and ideas with a fresh touch of
functional programming on top of it.

This post will cover the serialization of your objects into beautifully crafted
JSON (or any other protocol if that matters).

Phoenix has the concept of [views]( http://www.phoenixframework.org/docs/views ) whose responsibility is to expose variables and methods to be used when rendering templates.

```elixir
defmodule HelloPhoenix.UserView do
  use HelloPhoenix.Web, :view

  def render("index.json", %{users: users}) do
    Enum.map(users, fn user -> %{id: user.id, name: user.name} end)
  end
end
```

While this works fine for HTML templates, it felt awkward for serializing
objects into JSON. I felt like I shouldn't worry about mapping over an array of
objects, or wrapping the response with a root key every time I wrote a new
serializer. Also, reusing these views for including nested objects wasn't easy either.

That being said, I ended up writing a small library to deal with this.

### Building an API Facade Library

When writing Ruby, I usually resort to either ActiveModel Serializers or Grape Entities, but this time around I wanted a simpler, more lightweight approach to go with the functional style Elixir has to offer.

I started by defining the way I wanted to write my serializers knowing that I wanted them to be format agnostic (they shouldn’t really care whether we’re rendering JSON or XML) and that a single method should handle both a single resource or a list of resources.

Writing and using a serializer should look like something like this:

```elixir
defmodule HelloPhoenix.UserSerializer do
  alias HelloPhoenix.User

  def root, do: “user”
  def root_plural, do: “users”

  def map(resources)
    # rendering logic
  end

  def serialize(user) do
    %{ id: user.id, name: name(user) }
  end

  defp name(user) do
    “#{user.first_name} #{user.last_name}”
  end
end

> user = %User(id: 1, first_name: “Hugo”, last_name: “Ribeira”)
> UserSerializer.map(user)
%{client: %{id: 1, name: “Hugo Ribeira”}}
> UserSer ializer.map([user, user])
%{clients: [%{id: 1, name: “Hugo Ribeira”}, %{id: 1, name: “Hugo Ribeira”}]}
```

Once defined the interface, I went on with the implementation:

```elixir
defmodule HelloPhoenix.UserSerializer do
  # (...)
  def map(list), when: is_list(list) do
     list
       |> Enum.map(fn item -> serialize item end)
       |> Serializer.wrap_with_root(__MODULE__)
  end
  def map(item) do
    Serializer.wrap_with_root serialize(item), __MODULE__
  end

  defp wrap_with_root(list, serializer), when: is_list(list) do # when non empty list
    Map.put(%{}, UserSerializer.root_plural, list)
  end
  defp wrap_with_root(item, serializer) do # when single item
    Map.put(%{}, UserSerializer.root, item)
  end

  # (...)
end
```

Elixir really shines here. Pattern matching and guard clauses give you some type
safety at compile time and avoid gigantic if clauses (that will be evaluated at
runtime) and the functional style provides a simple and idiomatic way to describe
the serialization process as a pipeline of transformations from your model to
the final API response.

We've got our implementation and the interface we defined is respected, but we
still have the problem of repeating common logic across our serializers. Gladly,
Elixir let's us DRY up our implementation quite easily.

The first step is to remove this duplicated logic into its own module.

```elixir
defmodule HelloPhoenix.Serializer do
  def map(list, serializer), when: is_list(list) do
    list
      |> Enum.map(fn item -> serializer.serialize item end)
      |> wrap_with_root(serializer)
  end
  def map(item, serializer) do
    wrap_with_root serializer.serialize(item), serializer
  end

  defp wrap_with_root(list, serializer), when: is_list(list) do
    Map.put(%{}, serializer.root_plural, list)
  end
  defp wrap_with_root(item, serializer) do
    Map.put(%{}, serializer.root, item)
  end
end

defmodule HelloPhoenix.UserSerializer do
  def root, do: 'user'
  def root_plural, do: 'users'

  def serialize(user) do
    %{ id: user.id, name: name(user) }
  end

  defp name(user) do
    “#{user.first_name} #{user.last_name}”
  end
end

> Serializer.map(%User{id: 1, first_name: "Jonh"}, UserSerializer)
%{user: %{id: 1, name: "Jonh"}}
```

That's better!

We decoupled the logic responsible for root wrapping and mapping from the
serialization logic itself but we ended up changing the interface to support
this refactor.

Hmmm, that's not nice!

Glady, Elixir has got us covered its powerful macro system that let's us inject functionality from a
module into another. Next we'll use these macros to refactor our serializer,
maintaining logic decoupled and our interface as described initially.

```elixir
defmodule HelloPheonix.Serializer do
  defmacro __using__(_) do
    quote do # inject this code when :use macro is used inside a module
      @behaviour HelloPheonix.Serializer

      alias HelloPheonix.Serializer

      def map(list), when: is_list(list) do
        list
          |> Enum.map(fn item -> serialize item end)
          |> Serializer.wrap_with_root(__MODULE__) # Pass extended module to :wrap_with_root
      end

      def map(item) do
        Serializer.wrap_with_root serialize(item), __MODULE__
      end
    end
  end

  @callback map(Struct.t) :: Struct.t
  @callback serialize(Struct.t) :: Struct.t
  @callback root() :: String.t
  @callback root_plural() :: String.t

  def wrap_with_root(list, serializer), when: is_list(list) do
    Map.put(%{}, serializer.root_plural, list)
  end
  def wrap_with_root(item, serializer) do
    Map.put(%{}, serializer.root, item)
  end
end

defmodule HelloPhoenix.UserSerializer do
  use HelloPhoenix.Serializer # Import functionality from Serializer

  def root, do: 'user'
  def root_plural, do: 'users'

  def serialize(user) do
    %{ id: user.id, name: name(user) }
  end

  defp name(user) do
    “#{user.first_name} #{user.last_name}”
  end
end

> UserSerializer.map(%User{id: 1, first_name: "Jonh"})
%{user: %{id: 1, name: "Jonh"}}
```

The @behaviour and @callback macros are elixir's ways of defining an interface.

Elixir is dynamically typed, meaning that explicit interfaces are not needed
but defining them allows the compiler to assert at compile time that our serializers
abide the serialization contract. This avoids weird bugs caused by unexpected return
types or necessary but undefined methods.

### Wrapping Up

I must say I'm quite impressed with Elixir. Not only it is great to see your
endpoints return in microseconds rather then hundreds of milliseconds, as the functional
paradigm seems to fit perfectly around the data transformation and request life
cycle we work with when developing web applications.

When working with dynamic languages I sometimes miss the ability to explicitly
define interfaces, as implicit interfaces do come at the cost of needing more
documentation and/or extra runtime checks. Elixir really shines here, as besides
supporting implicit interfaces, the mix between pattern matching and explicit
interface definition allows for type safety when needed without too much runtime overhead.

Elixir is really fun and I'll probably end up writing a few more posts about it,
if you enjoyed this, do consider following me on twitter [@eidgeare](https://twitter.com/eidgeare)!
