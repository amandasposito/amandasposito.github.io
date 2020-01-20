---
layout: post
title:  "Elixir: What about tests?"
author: "Amanda Sposito"
date:   2020-01-02 15:21:00 -0300
categories:
  - elixir
  - test
---

There is no arguing about how important tests are for our application.

![Photo by Ben Kolde on Unsplash](/assets/images/2020-01-elixir-tests/ben-kolde-H29h6a8j8QM-unsplash.jpg)

But from time to time, when we are dealing with it, some questions came up on a daily basis.

A very common day-do-day case is our application relying on APIs and external libs, but one of the things we don't want our test suite to do is to access the external world.

This can lead us to many undesired scenarios, for example: if we have a connection problem on an API test that hits the internet, the test will fail, causing our test suite to be intermittent.

### How do we mock or stub a request?

Some libs can help us, I will talk about two of them.

### Bypass

The first one I wanna talk about is [Bypass](https://github.com/PSPDFKit-labs/bypass).
It is a lib that simulates an External Server, it works to stub the layer closer to the request. The idea is that you remove the external layer and start to call Bypass, so now, instead of talking to the internet, you are talking to a local server that you have control of. This way you can keep your tests protected.

Let's think about an example, we will build an application that needs to fetch data from an API, and we will create a `LastfmClient` module that will be responsible for doing that.

```elixir
defmodule MyApp.Lastfm.Client do
  @moduledoc"""
  Search tracks

  https://www.last.fm/api/show/track.search
  """

  @api_url "https://ws.audioscrobbler.com/2.0/"

  def search(term, url \\ @api_url) do
    response = Mojito.request(method: :get, url: search_url(url, term))

    case response do
      {:ok, %{status_code: 200, body: body}} ->
        {:ok, response(body)}
      {:ok, %{status_code: 404}} ->
        {:not_found, "Not found"}
      {_, response} ->
        {:error, response}
    end
  end
end
```

What this client does is fetch the tracks from our API and according to the type of the response return something. Not that different from most of the clients we wrote on a daily basis. Let's test it now.

```elixir
describe "search/2" do
  test "searches tracks by the term" do
    response = Client.search("The Kooks")

    assert {:ok, [
      %{
        "artist" => "The Kooks",
        "name" => "Seaside",
        "url" => "https://www.last.fm/music/The+Kooks/_/Seaside"
      }
    ]} = response
  end
end
```

### What's the problem with this test?
It seems pretty straight forward we exercise the [SUT](https://en.wikipedia.org/wiki/System_under_test) and verify the expected outcome, but this is a fragile test because it is accessing the external world.

Every time we call the `Client.search/2` we are hitting the internet. A lot of problems can happen here: if the internet is down the test will fail, if the internet is slow your suit test will be slow, you won't have the feedback as fast as you need and will be less inclined to run the test suit, or your suit test will become intermittent and you won't trust your tests anymore, meaning that when a real failure happens you won't care.

### How can we fix it?
And that's when [Bypass](https://github.com/PSPDFKit-labs/bypass) comes to our aid.

First, you will need to setup Bypass in your tests.

```elixir
setup do
  bypass = Bypass.open()

  {:ok, bypass: bypass}
end
```

And in your test, you will setup which scenarios you want to test. Is it a success call? A not found? How should your code behave in each scenario? Tell Bypass that.

```elixir
describe "search/2" do
  test "searches tracks by the term", %{bypass: bypass} do
    Bypass.expect bypass, fn conn ->
      Plug.Conn.resp(conn, 200, payload())
    end

    response = Client.search("The Kooks", "http://localhost:#{bypass.port}/")

    assert {:ok, [
      %{
        "artist" => "The Kooks",
        "name" => "Seaside",
        "url" => "https://www.last.fm/music/The+Kooks/_/Seaside"
      }
    ]} = response
  end
end

defp payload do
  ~s(
    {
      "results": {
        "@attr": {},
        "opensearch:Query": {
          "#text": "",
          "role": "request",
          "startPage": "1"
        },
        "opensearch:itemsPerPage": "20",
        "opensearch:startIndex": "0",
        "opensearch:totalResults": "51473",
        "trackmatches": {
          "track": [
            {
              "artist": "The Kooks",
              "image": [
                {
                  "#text": "https://lastfm.freetls.fastly.net/i/u/34s/2a96cbd8b46e442fc41c2b86b821562f.png",
                  "size": "small"
                },
                {
                  "#text": "https://lastfm.freetls.fastly.net/i/u/64s/2a96cbd8b46e442fc41c2b86b821562f.png",
                  "size": "medium"
                },
                {
                  "#text": "https://lastfm.freetls.fastly.net/i/u/174s/2a96cbd8b46e442fc41c2b86b821562f.png",
                  "size": "large"
                },
                {
                  "#text": "https://lastfm.freetls.fastly.net/i/u/300x300/2a96cbd8b46e442fc41c2b86b821562f.png",
                  "size": "extralarge"
                }
              ],
              "listeners": "851783",
              "mbid": "c9b89088-01cd-4d98-a1f4-3e4a00519320",
              "name": "Seaside",
              "streamable": "FIXME",
              "url": "https://www.last.fm/music/The+Kooks/_/Seaside"
            }
          ]
        }
      }
    }
  )
end
```

### When to use Bypass?
When you have code that needs to make an HTTP request. You need to know how your application will behave. For instance, if the API is down, will your application stop working?

But then comes some questions, if I have a module that depends on this `Client` implementation, will I need to repeat this Bypass code every time in my tests?
Why does another module need to know these implementation details if it is not dealing with the request?

### Mox
[Mox](https://github.com/plataformatec/mox) can help us with that. It forces you to implement explicit contracts in your application so we know what to expect.

Going back to our example, let's implement a module called `Playlist` that will be responsible for fetching a list of songs by artist and give it a name.

```elixir
defmodule MyApp.Playlist do
  alias MyApp.Lastfm.Client

  def artist(name) do
    {:ok, songs} = Client.search(name)

    %{
      name: "This is #{name}",
      songs: songs
    }
  end
end
```

The simplest test we can write to this code would be something like:

```elixir
describe "artist/1" do
  test "returns the songs by artist" do
    result = Playlist.artist("The Kooks")

    assert result["name"] == "This is The Kooks"
    assert Enum.any?(result["songs"], fn song ->
      song["artist"] == "The Kooks"
    end)
  end
end
```

Since the `Playlist` depends on the `Client`, to have an accurate test we would need to stub the request with the payload response from the Lastfm API so we can make sure the `Playlist` behaves accordingly.

You don't need to stub all the `Client` requests in the `Playlist` tests, you need to know what it returns and handle the responses, you need to have a [explicit contract](http://blog.plataformatec.com.br/2015/10/mocks-and-explicit-contracts/).

Let's see how we can implement those contracts.

### Behaviours

Elixir uses [behaviours](https://elixir-lang.org/getting-started/typespecs-and-behaviours.html#behaviours) as a way to define a set of functions that have to be implemented by a module. You can compare them to [interfaces](https://en.wikipedia.org/wiki/Interface_(computing)) in OOP.

Let's create a file at `lib/my_app/music.ex` that says what our `Client` expects as an argument and what it returns:

```elixir
defmodule MyApp.Music do
  @callback search(String.t()) :: map()
end
```

In our `config/config.exs` file, let's include two lines. The first one says which client we are using and the second one is the Lastfm API that we will remove from the default argument, just to keep the callback simple.

```elixir
config :my_app, :music, MyApp.Lastfm.Client
config :my_app, :lastfm_api, "https://ws.audioscrobbler.com/2.0/"
```

In our `config/test.exs` file, let's include our mock module.

```elixir
config :my_app, :music, MyApp.MusicMock
```

In the `test/test_helper` file, let's tell Mox which is the mock module that is responsible for mocking the calls to our behaviour.

```elixir
Mox.defmock(MyApp.MusicMock, for: MyApp.Music)
```

Let's go back to our `Playlist` module, and let's change the way we call the `Client` module, to use the config we just created.

```elixir
defmodule MyApp.Playlist do
  @music Application.get_env(:my_app, :music)

  def artist(name) do
    {:ok, songs} = @music.search(name)

    %{
      "name" => "This is #{name}",
      "songs" => songs
    }
  end
end
```

In our `Client` module let's adopt the behaviour we created, and let's change the API url to fetch from the config we created.

```elixir
defmodule MyApp.Lastfm.Client do
  @moduledoc"""
  Search tracks

  https://www.last.fm/api/show/track.search
  """

  @behaviour MyApp.Music

  def search(term) do
    url = lastfm_api_url()
    response = Mojito.request(method: :get, url: search_url(url, term))

    case response do
      {:ok, %{status_code: 200, body: body}} ->
        {:ok, response(body)}
      {:ok, %{status_code: 404}} ->
        {:not_found, "Not found"}
      {_, response} ->
        {:error, response}
    end
  end

  defp lastfm_api_url do
    Application.get_env(:my_app, :lastfm_api)
  end
end
```

We will need to change the `test/my_app/lastfm/client_test.exs` to change the env config for the API url on the setup of the test, but I'll leave it to you to do that.

Finally, in our `PlaylistTest` we will need to import `Mox`.

```elixir
import Mox

# Make sure mocks are verified when the test exits
setup :verify_on_exit!
```

And in our test, we need to tell our `MusicMock` what is expected to return.

```elixir
describe "artist/1" do
  test "returns the songs by artist" do
    MusicMock
    |> expect(:search, fn _name ->
      {
        :ok,
        [
          %{
            "artist" => "The Kooks",
            "name" => "Seaside",
            "url" => "https://www.last.fm/music/The+Kooks/_/Seaside"
          }
        ]
      }
    end)

    result = Playlist.artist("The Kooks")

    assert result["name"] == "This is The Kooks"
    assert Enum.any?(result["songs"], fn song ->
      song["artist"] == "The Kooks"
    end)
  end
end
```

### What's the difference?
Looking at the code it seems that we still need to pass the list of music in the tests. But there is a difference, the music's list is a dependency of the `Playlist` module, but we don't know its internals, we don't know from where we are fetching it, how the request response works, and all these details. The only thing we need to know at the `Playlist` module is that it depends on a list of songs.

### One last thing before we go
We went through all that trouble to make sure the tests are protected from the outside world, but you know, Elixir has this amazing `Doctest` feature, and one can argue that this replaces the application tests.

That's not the case, `Doctests` are not `tests` and you shouldn't rely on it to make sure your application behaves the way you expect it to. Make sure you don't use any code that hits the external world when you are writing your documentation, there is no way to mock calls and this can cause all the problems we already discussed.

### That's all folks

The code from this example can be found [here](https://github.com/amandasposito/my_app), I hope it helps!
