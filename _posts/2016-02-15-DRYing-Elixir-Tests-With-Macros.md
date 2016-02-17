---
layout: post
title: DRY’ing Elixir Tests with Macros
date: 2016-02-15
tag: markdown
blog: true
#star: true
---

I’ve been experimenting with Elixir/Phoenix for a while now and I must say it has been an excellent journey. Phoenix brings Rails happiness and productivity without imposing to much of its ideology upon you. Which to be honest is kind of refreshing.

This week I set up user authentication using [Guardian](https://github.com/ueberauth/guardian) and, as an avid [RSpec](http://rspec.info/) user, I wondered what the ```shared_examples_for``` [[1](https://www.relishapp.com/rspec/rspec-core/v/2-0/docs/example-groups/shared-example-group)] would like in Elixir.

If you don’t know RSpec, ```shared_examples_for``` let you share test cases between different test collections. One common use for such would be making sure your endpoints are being authenticated correctly.

It looks mostly like this:

```ruby
# Shared Code
module AuthenticatedEndpoint
  shared_examples_for ‘AuthenticatedEndpoint’ do |method, api_path|
    it “returns unauthorized when no authentication token is given” do
      send :method, api_path, nil, nil # body, headers
      expect(response.status_code).to eq 401
    end
  end
end

# Test Case
describe SomeEndpointController do
  # setup
  behaves_like “AuthenticatedEndpoint”, :get, some_path
  # the rest of your test code
end
```

# Elixir

A lifetime with Ruby has grown in me a distaste for DSLs, albeit good looking, when used too often they tend to obscure simple concepts.

With Elixir, I didn't want to add another DSL just for sharing tests across
units. A spice of metaprogramming should be all that's needed to enable dynamic
test generation at compile time, plus, I've been wanting to mess with Elixir's
macro system for a while now and this seemed like the perfect excuse for it.

Let’s start with a simple, vanilla exUnit test:

```elixir
defmodule MacroTests.BiciclesControllerTest do
  use MacroTests.ConnCase

  alias MacroTests.Bicycle

  test “creating a bicycle requires authentication”, %{conn: conn} do
    assert conn |> post(conversation_path(conn, :create)) |> json_response(401)
  end

  test “listing bicycles requires authentication”, %{conn: conn} do
    assert conn |> get(conversation_path(conn, :index)) |> json_response(401)
  end

  # ...

end
```

While this works, my OCD can’t bare the amount of repetition that’s going around. And this is only one endpoint that we’re talking about... What happens if I decide to return a 404 instead of a 401? How many files will I have to go through to make my tests reflect that?

It just doesn’t seem right.

What I would prefer is to have that authentication test logic abstracted away somewhere... Something that I could use like this:

```elixir
defmodule MacroTests.BiciclesControllerTest do
  use MacroTests.ConnCase

  test_authentication_required_for(:post, :bicycle_path, :create)
  test_authentication_required_for(:get, :bicycle_path, :index)

  # ...

end
```

To have ExUnit run dinamically generated tests, we need to turn
```test_authentication_required_for(:post, :bicycle_path, :create)```
into ```test "test doc string", do: test_stuff``` clause at compile time.

Luckily enough, Elixir provides us both ways of dinamically evaluating functions
and writing macros that are expanded at compile time for you. The following
paragraphs will cover how to write such a dinamically generated test using
Elixir's metaprogramming features.

# Dynamic function calling

The first step we need to accomplish to generate our test cases is to take the
```path_name``` argument and dinamically invoke it from ```Router.Helpers``` at run-time.

Elixir gives you the ability to call a function dynamically using ```Kernel.apply/3```.
This function takes a module, a function name and a list of arguments the
function will be invoked with.

To evaluate a path at runtime from its name all we need to do is:

```elixir
path_action = :get
path_name = :bicycle_path

path = apply(MacroTests.Router.Helpers, path_name, [ConversationApi.Endpoint, path_action])
```

Knowing the path, we'll have to make the actual request.

To simulate a request going through a plug in a test, Phoenix provides you with ```get,
post, put, delete, (...)``` methods. These are just macros that call
```Phoenix.ConnTest.dispatch\5``` with the given method name as an argument, a bit like so:

```elixir
  defmacro get(path, options), do: dispatch(conn, @endpoint, :get, path, options)
```

Then to dinamically trigger a request all we would need to do is:

```elixir
def make_request(path_name, path_action) do
  path = apply(MacroTests.Router.Helpers, path_name, [ConversationApi.Endpoint, path_action])
  dispatch(conn, @endpoint, path_action)
end
```

# Putting it all together

Now that we can retrive the paths and make requests dinamically, all that's left
is generating test cases at compile time and Elixir macros let you do just that.

```elixir
defmodule MacroTests.AuthConnTest do
  alias MacroTests.Router

  import Plug.Conn, only: [delete_req_header: 2]
  import Phoenix.ConnTest, only: [dispatch: 5, json_response: 2]

  defmacro test_authentication_required_for(method, path_name, action) do
    quote do
      test "requires authentication", %{conn: conn} do
        method    = unquote(method)
        path_name = unquote(path_name)
        action    = unquote(action)
        assert make_unauthenticated_request(conn, @endpoint, method, path_name, action)
      end
    end
  end

  def make_unauthenticated_request(conn, endpoint, method, path_name, action) do
    path = apply(Router.Helpers, path_name, [conn, action])
    conn = conn |> delete_req_header("authorization")
    dispatch(conn, endpoint, method, path, nil) |> json_response(401)
  end
end
```

The ```quote``` method takes your Elixir code and converts it into an
[AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree). While ```unquote```
let's you access variables defined outside of the ```quote``` block.

The best I've seen it explained (by Chris Mccord) is to think of ```quote```
like a string for code where ```unquote``` lets you do string (in this case code)
interpolation. This way you can access computed properties inside your block of otherwise
static code.

Explanations aside, it's a case of importing ```MacroTests.AuthConnTest```
on your controller tests and you should be good to go.

```elixir
defmodule MacroTests.BiciclesControllerTest do
  use MacroTests.ConnCase

  import MacroTests.AuthConnTest

  test_authentication_required_for(:post, :bicycle_path, :create)
  test_authentication_required_for(:get, :bicycle_path, :index)

  # ...

end
```

Happy DRY testing! :)

If you want to know more about Elixir and Metaprogramming
Chris Mccord [Metaprogramming Elixir Talk](https://vimeo.com/131643017) is an
excellent start. It covers the basics and some real use case scenarios (such as
Ecto and Logging utilities).

If you liked what you read and want to get in touch, give me a shout at
[@eidgeare](https://twitter.com/eidgeare).
