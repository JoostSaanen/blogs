## Real World Phoenix |> A LiveView Dashboard

There is some real excitement going on in the Elixir community after the fact that Chris McCord made his [PhoenixLiveView repo](https://github.com/phoenixframework/phoenix_live_view) available to the public. People have been experimenting wildly with the immense power that comes with Phoenix Live View. If you are totally new to LiveView and haven't heard about it, be sure to check out the [initial blog post](https://dockyard.com/blog/2018/12/12/phoenix-liveview-interactive-real-time-apps-no-need-to-write-javascript) on the Dockyard website and also [this talk](https://youtu.be/Z2DU0qLfPIY) by Chris.

Now that we have a common understanding of what LiveView is, let's see what we can explore today using LiveView. Most things I have seen thusfar have been standalone interactive little projects that contain either some nifty animations and/or some interactivity using forms. Very awesome stuff! I'd like to take the concept of LiveView to a slightly different direction and use it to create one of the use cases Chris has mentioned, a LiveView dashboard! With the upcoming ElixirConfEU in Prague, let's see if we can actually get something meaningful off the ground.

### A LiveView Template

Setting up a project for LiveView is explained step-by-step in the LiveView Repository, so we could of course just create a Phoenix Project and follow those steps and set everything up like that. But, 
I am always looking for ways to make sure we as a community don't have to do all kinds of manual tasks over and over again and I recently came accross a cool library created by Dave Thomas that provides a really nice and clean way to create template projects. You can read more about that [here](https://github.com/pragdave/mix_templates) and be sure to [watch the video](https://vimeo.com/213689412) made by Dave about this.

So, as a convenience I have created a phx_live_view template so that everyone can have a fresh Phoenix Live View project quick and easy. In just four easy commands you'll have your own Phoenix Live View project up and running with an example clock on the homepage.

It is as simple as:
```
// install libraries from Dave Thomas
mix archive.install hex mix_templates
mix archive.install hex mix_generator

// install my template
mix template.install hex gen_template_phx_live_view

// create your project
mix gen phx_live_view MyAwesomeLiveViewProject
```

For my dashboard I wanted to have some boilerplate styling and libraries to make my design work a little easier. So I created an additional template that sets up a few other things. Namely Bootstrap and Sass and an initial dashboard layout.

You can of course use this template as a starting point just as easily as the basic one:

```
//Install the template
mix template.install hex gen_template_phx_live_view_dashboard

// create your project
mix gen phx_live_view_dashboard MyAwesomeLiveViewDashboard
```

If you start up either one of the above templates you'll see that it already has a LiveView clock running. 

### Countdown To ElixirConf

With the upcoming ElixirConfEU in Prague, let's see if we can get some more realtime data on the screen that relates to this event. Let's start by creating a countdown timer to the start of the conference!

Adding a new LiveView component can be done in two simple steps.

1. create a LiveView in `lib/yourproject_web/live/live_view_module.ex`
This LiveView must implement two functions, `mount/2` and `render/1`. 

2. Render the view on a page.
This is simple using the `live_render/2` function.
Here is the example from the clock that is in the template.
```elixir
  <%= live_render(@conn, ElixirConfLiveViewWeb.Clock) %>
```

Let's create the countdown timer. I'll use the awesome [Timex](https://github.com/bitwalker/timex) library to make my life a little easier. Mainly because it has some nice convenience methods for [Intervals](https://hexdocs.pm/timex/Timex.Interval.html#content), [Duration](https://hexdocs.pm/timex/Timex.Duration.html#content) and a [Formatter](https://hexdocs.pm/timex/Timex.Format.Duration.Formatter.html#format/1) that humanizes the countdown in one go. 

So let's add that to `mix.exs`
```elixir
...
  {:timex, "~> 3.1"},
...
```

And here is the `ViewModule` implementation of the countdown timer. Once the `livesocket` is connected it fires off a timer that sends a message to the process every second to get the current duration to the start of ElixirConf. That triggers and update to the `socket.assigns`, which in it's turn triggers the `render` function to re-render the component on the page. Great stuff!

```elixir
defmodule ElixirConfLiveViewWeb.CountdownToElixirconf do
  use Phoenix.LiveView
  use Phoenix.HTML
  alias Timex.{ Interval, Duration, Format.Duration.Formatter }
  
  def render(assigns) do
    ~L"""
    <div class="countdown-to-elixirconf">
      <h3><%= @time_to_conf %></h3>
    </div>
    """
  end

  def mount(_session, socket) do
    if connected?(socket), do: :timer.send_interval(1000, self(), :tick)

    {:ok, put_time_to_conf(socket)}
  end

  def handle_info(:tick, socket) do
    {:noreply, put_time_to_conf(socket)}
  end

  defp put_time_to_conf(socket) do
    assign(socket, time_to_conf: time_to_conf())
  end

  defp time_to_conf do
    cond do
      time_to_elixirconf() > Timex.now ->
        "It's Alive!! Alive!"
      true ->
        Interval.new(from: Timex.now, until: time_to_elixirconf()) 
        |> Interval.duration(:seconds) 
        |> Duration.from_seconds 
        |> Formatter.format(:humanized)
    end
  end

  defp time_to_elixirconf do
    Timex.set(Timex.now("Europe/Prague"), [month: 4, day: 8, hour: 9, minute: 0, second: 0])
  end
end
```

And getting that on the page is done using the `live_render/2` function just like this:

```elixir
<div class="col-md-6">
  <div class="card">
    <div class="card-header">
      <h4 class="card-title">Countdown To ElixirConfEU - Prague</h4>
    </div>
    <div class="card-body">
      <%= live_render(@conn, ElixirConfLiveViewWeb.CountdownToElixirconf) %>
    </div>
  </div>
</div>
```

### Github

Cool now we at least know how long we have to wait... :) Now let's see if we can integrate some third-party data into our dashboard. As this post is about PhoenixLiveView which is officially not released yet, it would be nice to get some stats from Github to see how close we are to an actual release, right? 
Getting data from Github is a breeze if we use the [Tentacat](https://github.com/edgurgel/tentacat) library, so let's add that to the mix!

```elixir
...
  {:tentacat, "~> 1.0"},
...
```

Ok, let's now implement the naive way (I'll explain in a bit) of fetching data and inject that into our LiveView component. We'll fetch the data from the repo using the `Tentacat.Repositories.repo_get/3`, which will basically get us all kinds of data / stats about a repo given the org_name and repo_name. Here is a snippet of what we'll fetch:

```elixir
def statistics do
  token = Application.get_env(:elixir_conf_live_view, :github_api_key)
  client = Tentacat.Client.new(%{access_token: token})

  { 200, _, response} = Tentacat.Repositories.repo_get(client, "phoenixframework", "phoenix_live_view")
  
  stats = response.body 

  %{
    stars: stats["stargazers_count"],
    issues: stats["open_issues_count"],
    forks: stats["forks_count"]
  }
end
```

You'll notice that I am authenticating to fetch the data. You can actually get this data without authenticating, but you'll hit the rate limit very quickly, so that is why I chose to authenticate here. Using Tentacat, this is very easy, so that doesn't really make things more complicated at all.
What I also wanted to do here is to add a countdown timer to the talk by Chris @ ElixirConf as I'm hoping that he might announce an official release there... :) 
Below is the `render` function in the component with the Countdown timer to Chris' talk added. Notice how I now need to pass in `@socket` instead of `@conn` when adding a `live_render` call in this `.leex` template.

```elixir
def render(assigns) do
  ~L"""
  <div class="d-flex justify-content-center">
    <h4><%= live_render(@socket, ElixirConfLiveViewWeb.CountdownToTalkChris) %></h4>
  </div>
  <div class="live_view_stats">
    <div class="stat">
      <span class="heading">Stars</span>
      <span class="value"><%= @statistics.stars %></span>
    </div>
    <div class="stat">
      <span class="heading">Issues</span>
      <span class="value"><%= @statistics.issues %></span>
    </div>
    <div class="stat">
      <span class="heading">Forked</span>
      <span class="value"><%= @statistics.forks %></span>
    </div>
  </div>
  """
end
```

Ok, so far so good. Currently, aside from the clock, there is not much real-time data going on. Or at least you'd have to wait some time for the stars and issues to change, but it's a start. You could of course star/unstar the repo yourself(and wait 10 seconds) to see it change. 

### Loading external data through API endpoints

Now, let's stop and think about the way we are currently pulling data from Github. Currently I'm testing this by myself, but what if there are hundreds of clients connecting to this dashboard. They all get their own socket connection and they also get their own instance of the LiveView component. And that also means that they all individually make calls to the Github API because that actually happens when the component gets mounted... Ouch, Github is not going to be happy with me as all of these events are done using my authenticated key.
It should be pretty obvious that we shouldn't be fetching the data from the API in the ViewModule itself. In the instructions Chris mentions that we shouldn't load data in the template, but I think the same statement holds for the whole ViewModule, maybe not all the time, but especially when connecting to 3rd party services. So let's solve this problem by creating a service that polls the 3rd party for updates and pushes those updates down to the LiveView module. This is exactly the kind of thing we could use the `Phoenix.PubSub` module for! This is used internally in Phoenix to handle the implementation of `Phoenix.Channels`, but also has an API to use it for anyting else that needs a publish-subscribe pattern.

Let's start by creating the Github API module in the backend separately. What we'll need is a process that will fetch the data from github regularly. A GenServer seems to be the perfect solution for this. So let's move the logic from the LiveVirw Component into a new module powered by GenServer behaviour.

This should do the trick:

```elixir
defmodule ElixirConfLiveView.Api.Github do
  alias Tentacat.Repositories 
  use GenServer

  @timeout 10_000

  def start_link([]) do
    GenServer.start_link(__MODULE__, [])
  end
  
  def init(_) do
    {:ok, statistics(), @timeout}
  end
  
  def handle_info(:timeout, statistics) do
    new_stats = statistics()
    ElixirConfLiveViewWeb.Endpoint.broadcast("github", "new_stats", new_stats) 
    {:noreply, new_stats, @timeout}
  end

  defp statistics do
    token = Application.get_env(:elixir_conf_live_view, :github_api_key)
    client = Tentacat.Client.new(%{access_token: token})

    { 200, _, response} = Tentacat.Repositories.repo_get(client, "phoenixframework", "phoenix_live_view")
		
    stats = response.body 

		%{
			stars: stats["stargazers_count"],
			issues: stats["open_issues_count"],
			forks: stats["forks_count"],
			releases: stats["releases"]
		}
  end
end
```
And we'll need to add this to our application Supervisor in order to start this when the app boots.

```elixir
...
children = [
  # Start the endpoint when the application starts
  ElixirConfLiveViewWeb.Endpoint,
  # Starts a worker by calling: ElixirConfLiveView.Worker.start_link(arg)
  {ElixirConfLiveView.Api.Github, []},
]
...
```

Often I see people using `Process.send_after` to configure a timer-like scheduled function call (including myself implementing the first solution), but that functionality is actually built into GenServer as a :timeout callback. So you just have to pass the timeout value in the init function and configure a `handle_info/2` function to catch that and do whatever you please. We could create a separate module to implement our PubSub functionallity, but we could also use the one that is already running in the Endpoint. So all we have to do to publish our new state is broadcast a message in a topic using `Endpoint.broadcast/3`. In the case above I'm broadcasting the message `new_stats` to the `github` topic.
In our LiveView module we can easily subscribe to these message when the component is mounted:

```elixir
if connected?(socket), do: ElixirConfLiveViewWeb.Endpoint.subscribe("github")
```

And we then listen for the "new_stats" message in a `handle_info/2` callback.

```elixir
def handle_info(%{event: "new_stats", payload: payload}, socket) do
  {:noreply, assign(socket, :statistics, payload)}
end
```

### Adding a Twitter Stream

Since the data we are retrieving thusfar is not very dynamic, let's see if we can get some more interactivity by fetching an actual twitter stream. The cool thing here is that we actually don't have to poll twitter for data because we can use the Twitter Stream API!
It is a bit more hassle to set up, so I won't bore you with those details. You'll have to apply for an application as a developer @ twitter, so be sure to read their docs if you want to set something up yourself. You can of course check my setup in this repo.

What we want to achieve here is to display the latest 10 tweets mentioning @ElixirConfEU. We could do this using the Search API, but a much nicer solution would be to open a Twitter stream and push through any new events immediately. Just like the Github implementation, we'll fetch the data using a GenServer and subscribe our LiveView module to the topic we create there.

To integrate Twitter we can use the `extwitter` library that wraps the Twitter API.

In mix.exs:
```
...
  {:extwitter, "~> 0.8"},
...
```

And here is our GenServer that will connect to the twitter stream.

```elixir
defmodule ElixirConfLiveView.Api.Twitter do
  use GenServer

  def start_link([]) do
    GenServer.start_link(__MODULE__, [])
  end
  
  def init(_) do
    {:ok, start_stream()}
  end
  
	def start_stream do
		spawn(fn ->
      stream = ExTwitter.stream_filter(track: "@ElixirConfEU")
      for tweet <- stream do
				ElixirConfLiveViewWeb.Endpoint.broadcast("tweets", "new_tweet", %{tweet: tweet.text}) 
      end
    end)
	end
end
```

And, just like the github LiveView module, we'll fetch the "new_tweet" messages

```elixir
def handle_info(%{event: "new_tweet", payload: payload}, socket) do
  {:noreply, put_tweets(socket, payload.tweet)}
end

defp put_tweets(socket, tweet) do
  tweets = [ tweet | Enum.take(socket.assigns.tweets, 9) ]
  assign(socket, :tweets, tweets)
end
```

So far so good. At the moment that I'm writing this, the stream is not very active, so if you want to try out something fun, you could change the stream_filter to track something with more tweets (ie. "apple"), and you'll see a nice firehouse effect on your page, with unreadable tweets as they fly by very fast... 

I am really excited about being able to create a dashboard using only Elixir in this way and I'll definitely be exploring some more ideas the coming week to see if I can get some more real-time stats about ElixirConf going. I'll be in Prague next week, so if anyoone wants to chat or get together, be sure to hit me up on [twitter](https://twitter.com/drumusician).

You can find the implemented dashboard in this [repo on Github](https://github.com/drumusician/elixir_conf_live_view_dashboard).

Hope you learned something from this post.

Until next time!

