---
layout: post
title: Building an API Facade with Elixir
---

Coming from Ruby and growing a bit tired of resorting to caching every time things get a bit more complicated than a simple CRUD application, I’ve been meaning to try Elixir and its web framework [Phoenix](http://www.phoenixframework.org/) for a while. 

[Elixir](http://elixir-lang.org/) is the new cool kid in the block. It’s built on top of the Erlang and uses most of its concurrency and failure handling models.

Needless to say, it’s blazingly fast!

(this should probably go into the sum part) I’m quite impressed I must say, seeing render times measured in microseconds is just something you’re not used to when you’re developing a Rails application.

### Building API’s with phoenix

With the market revolving more and more around smartphone and multi-client applications, I’ve been writing API based applications for a while now.

Elixir and Phoenix come with everything you need to build an API, but one thing I’ve missed was a simple way to define how my models should be serialized into an API response.

Phoenix has the concept of (views)[http://www.phoenixframework.org/docs/views] whose responsibility is to expose variables and methods to be used when rendering templates.

```elixir
defmodule HelloPhoenix.UserView do
  use HelloPhoenix.Web, :view

  def render("index.json", %{users: users}) do
    Enum.map(users, fn user -> %{id: user.id, name: user.name} end)
  end
end
```

Although this works great for HTML views, I didn’t really think it felt natural for rendering an API response, so I set out to build a simple serialization mechanism.

When writing Ruby, I usually resort to either ActiveModel Serializers or Grape Entities, but this time around I wanted a simpler, more lightweight approach to go with the functional style Elixir has to offer.

### Building an API Facade Library

I started by defining the way I wanted to write my serializers knowing that I wanted them to be format agnostic (they shouldn’t really care wether we’re rendering JSON or XML or whatever else) and that a single method should handle both a single resource or a list of resources.

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
  def map([]) do
    Serializer.wrap_with_root [], __MODULE__
  end
  def map(list = [_h|_t]) do
     list
       |> Enum.map(fn item -> serialize item end)
       |> Serializer.wrap_with_root(__MODULE__)
  end
  def map(item) do
    Serializer.wrap_with_root serialize(item), __MODULE__
  end
  
  defp wrap_with_root([], serializer) do # when empty list
    Map.put(%{}, UserSerializer.root_plural, [])
  end
  defp wrap_with_root(list = [_h|_t], serializer) do # when non empty list
    Map.put(%{}, UserSerializer.root_plural, list)
  end
  defp wrap_with_root(item, serializer) do # when single item
    Map.put(%{}, UserSerializer.root, item)
  end

  # (...)
end
```

Elixir really shines here, as pattern matching makes it very easy to define behavior without resorting to gigantic if statements and the functional style lets you describe a pipeline of transformations from your original input to serializable hash in an idiomatic and simple to understand way.

We could have called it here, but it wouldn’t be great to have to duplicate the mapping and root wrapping logic in every serializer you write, right?

The first step is to remove this duplicated logic into its own module.

```elixir
defmodule HelloPhoenix.Serializer do
  def map([], serializer) do
    wrap_with_root serializer.serialize([]), serializer
  end
  def map(list = [_h|_t], serializer) do
    list
      |> Enum.map(fn item -> serializer.serialize item end)
      |> wrap_with_root(serializer)
  end
  def map(item, serializer) do
    wrap_with_root serializer.serialize(item), serializer
  end

  defp wrap_with_root([], serializer) do
    Map.put(%{}, serializer.root_plural, [])
  end
  defp wrap_with_root(list = [_h|_t], serializer) do
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

Elixir has a powerful macro mechanism that let's us inject functionality from a
module into another. Next we'll use these macros to refactor our serializer,
maintaining logic decoupled and our interface as described in the beginning.

```elixir
defmodule HelloPheonix.Serializer do
  defmacro __using__(_) do
    quote do # inject these code when :use macro is used inside a module
      @behaviour HelloPheonix.Serializer

      alias HelloPheonix.Serializer

      def map([]) do
        Serializer.wrap_with_root [], __MODULE__
      end

      def map(list = [_h|_t]) do
        list
          |> Enum.map(fn item -> serialize item end)
          |> Serializer.wrap_with_root(__MODULE__)
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

  def wrap_with_root([], serializer) do
    Map.put(%{}, serializer.root_plural, [])
  end
  def wrap_with_root(list = [_h|_t], serializer) do
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

Elixir is dynamically typed, meaning that explicit interfaces are not needed,
but defining them allows the compiler to assert at compile time that our serializers
abide the serialization contract avoiding weird bugs caused by unexpected return
types or necessary but undefined methods.

### Wrapping Up

- We managed to write a serialization abstraction that's both simple
  to use and write.

- We relied on Elixir macros to share functionality across modules and to
  guarantee our interfaces are correctly implemented.
