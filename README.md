# Elixir real-time twitter clone tutorial

Build a real-time Twitter clone in 15 minutes with LiveView and Phoenix 1.6

This is a tutorial by Chris McCord - Build a real-time Twitter clone in 15 minutes with LiveView and Phoenix 1.5 (https://youtu.be/MZvmYaFkNJI) but using latest Phoenix 1.6

## Requirements
1. elixir 1.14
2. phx_new (```mix archive.install hex phx_new```)
3. local postgres db installed

or use docker:

1. ```docker-compose run --rm --service-ports --entrypoint /bin/bash elixir```
2. phx_new (```mix archive.install hex phx_new```)

## What you will learn?
1. Phoenix 1.6 framework basics
2. Basics of Phoenix LiveView library (altarnative for classic SPA)
3. Basics of Phoenix PubSub
4. Ecto (ORM for Elixir)

## Steps for building a real-time Twitter clone in 15 minutes with LiveView and Phoenix 1.6
1. ```mix phx.new elixir_realtime_twitter_clone --live```
2. ```cd elixir_realtime_twitter_clone```
3. ```git init```
4. ```code .```
5. ```mix phx.gen.live Timeline Post posts username body likes_count:integer repost_count:integer```
6. Add following lines in ```lib/elixir_realtime_twitter_clone_web/router.ex:22```
   ```
    live "/posts", PostLive.Index, :index
    live "/posts/new", PostLive.Index, :new
    live "/posts/:id/edit", PostLive.Index, :edit

    live "/posts/:id", PostLive.Show, :show
    live "/posts/:id/show/edit", PostLive.Show, :edit
    ```
7. Edit ```config/dev.exs```. Change line 7 to ```hostname: "postgres",``` and line 22 to ```http: [ip: {0, 0, 0, 0}, port: 4000],```
8. ```mix phx.server```
9. Go to http://localhost:4000/posts in browser
10. Click "Create database for repo" and next "Run migrations for repo"
11. A webpage "Listing Posts" should apear in browser with empty post list
12. Open form component template ```lib/elixir_realtime_twitter_clone_web/live/post_live/form_component.html.heex``` and remove non editable form fields (lines 12-16, 20-22 and 24-26). Change body input type from ```text_input``` to ```textarea``` (line 17)
13. Check if "New Post" form chenged in the browser
14. Open ```lib/elixir_realtime_twitter_clone/timeline/post.ex``` and add default values in post schema. Change line 7 to ```field :likes_count, :integer, default: 0```, line 8 to ```field :repost_count, :integer, default: 0``` and line 9 to ```field :username, :string, default: "your_username"```
15. In file ```lib/elixir_realtime_twitter_clone/timeline/post.ex``` change line 17 to ```|> cast(attrs, [:body])``` and line 18 to ```|> validate_required([:body])``` and add ```:body``` field validation in line 19 - ```|> validate_length(:body, min: 2, max: 250)```
16. Check "New Post" form in the browser. You should have immediate validation ("can't be blank"; "should be at least 2 character(s)"; "should be at most 250 character(s)")
17. Open ```lib/elixir_realtime_twitter_clone_web/live/post_live/index.html.heex```. Change line 1 to ```<h1>Timeline</h1>``` and hange lines 16-43 to 
    ```
    <div id="posts">
    <%= for post <- @posts do %>
        <%= live_component @socket, ElixirRealtimeTwitterCloneWeb.PostLive.PostComponent, id: post.id, post: post %>
    <% end %>
    </div>
    ```
18. Create post component in ```lib/elixir_realtime_twitter_clone_web/live/post_live/post_component.ex```
    ```
    defmodule ElixirRealtimeTwitterCloneWeb.PostLive.PostComponent do
        use ElixirRealtimeTwitterCloneWeb, :live_component

        def render(assigns) do
            ~H"""
            <div id={"post-#{ @post.id }"} class="post">
            <div class="row">
                <div class="column column-10">
                <div class="post-avatar"><img src={"https://ui-avatars.com/api/?background=random&name=#{ @post.username }"} /></div>
                </div>
                <div class="column column-90 post-body">
                <b>@<%= @post.username %></b>
                <br/>
                <%= @post.body %>
                </div>
            </div>
            <div class="row">
                <div class="column">
                <i class="far fa-heart"></i> <%= @post.likes_count %>
                </div>
                <div class="column">
                <i class="far fa-hand-peace"></i> <%= @post.repost_count %>
                </div>
                <div class="column">
                <%= live_patch to: Routes.post_index_path(@socket, :edit, @post.id) do %>
                    <i class="far fa-edit"></i>
                <% end %>
                <%= link to: "#", phx_click: "delete", phx_value_id: @post.id do %>
                    <i class="far fa-trash-alt"></i>
                <% end %>
                </div>
            </div>
            </div>
            """
        end
    end
    ```
19. Go to ```lib/elixir_realtime_twitter_clone_web/templates/layout/root.html.heex``` and add ```<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.2.0/css/all.min.css" />``` on line 9
20. Let's make timeline realtime now. Go to ```lib/elixir_realtime_twitter_clone/timeline.ex``` and add following code:
on line 56
```
    |> broadcast(:post_created)
```
and on line 75
```
    |> broadcast(:post_updated)
```
and on line 107
```
  def subscribe do
    Phoenix.PubSub.subscribe(ElixirRealtimeTwitterClone.PubSub, "posts")
  end

  defp broadcast({:error, _reason} = error, _event), do: error
  defp broadcast({:ok, post}, event) do
    Phoenix.PubSub.broadcast(ElixirRealtimeTwitterClone.PubSub, "posts", {event, post})
    {:ok, post}
  end
```
21. Go to ```lib/elixir_realtime_twitter_clone_web/live/post_live/index.ex``` and subscribe for posts events.
on line 9 add
```
    if connected?(socket), do: Timeline.subscribe()

```
and on line 45
```
  @impl true
  def handle_info({:post_created, post}, socket) do
    {:noreply, update(socket, :posts, fn posts -> [post | posts] end)}
  end
```
22. Open http://localhost:4000/posts in two browsers and try to add new post in one of them. It should show up instantaneously in second browser.
23. Let's fix post order between browsers. Go to ```lib/elixir_realtime_twitter_clone/timeline.ex``` and change line 21 to:
    ```
        Repo.all(from p in Post, order_by: [desc: p.id])
    ```
24. Now let's add broadcast on post update. Go to ```lib/elixir_realtime_twitter_clone_web/live/post_live/index.ex``` and add following on line 50
    ```
    def handle_info({:post_updated, post}, socket) do
        {:noreply, update(socket, :posts, fn posts -> [post | posts] end)}
    end
    ```
25. Now we can add collection optimization to prevent holding all posts in memory. In file ```lib/elixir_realtime_twitter_clone_web/live/post_live/index.ex``` change line 11 to:
```
{:ok, assign(socket, :posts, list_posts()), temporary_assigns: [posts: []]}
```
and in file ```lib/elixir_realtime_twitter_clone_web/live/post_live/index.html.heex``` change line 12 to
```
<div id="posts" phx-update="prepend">
```
26. Go back to the browser and try to edit post. It should update instantaneously in second browser.
27. Now we need to implement retweets and likes functionality. Go to ```lib/elixir_realtime_twitter_clone_web/live/post_live/post_component.ex``` and change lone 19 to:
    ```
          <a href="#" phx-click="like" phx-target={ @myself }>
            <i class="far fa-heart"></i> <%= @post.likes_count %>
          </a>
    ```
    and line 22 to:
    ```
          <a href="#" phx-click="repost" phx-target={ @myself }>
            <i class="far fa-hand-peace"></i> <%= @post.repost_count %>
          </a>
    ```
    and add event handlers on line 39:
    ```
    def handle_event("like", _, socket) do
        ElixirRealtimeTwitterClone.Timeline.inc_likes(socket.assigns.post)
        {:noreply, socket}
    end

    def handle_event("repost", _, socket) do
        ElixirRealtimeTwitterClone.Timeline.inc_reposts(socket.assigns.post)
        {:noreply, socket}
    end
    ```
28. Now in ```lib/elixir_realtime_twitter_clone/timeline.ex``` on line 24 add:
```
  def inc_likes(%Post{id: id}) do
    {1, [post]} =
      from(p in Post, where: p.id == ^id, select: p)
      |> Repo.update_all(inc: [likes_count: 1])

    broadcast({:ok, post}, :post_updated)
  end

  def inc_reposts(%Post{id: id}) do
    {1, [post]} =
      from(p in Post, where: p.id == ^id, select: p)
      |> Repo.update_all(inc: [repost_count: 1])

    broadcast({:ok, post}, :post_updated)
  end
```
29. Go back to the browser and check if likes and reposts works correctly.

## Futher reading
1. https://www.poeticoding.com/how-phoenix-liveview-works/