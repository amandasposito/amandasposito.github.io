---
layout: post
title:  "Preload nested associations using Ecto"
author: "Amanda Sposito"
date:   2018-08-08 15:21:00 -0300
categories:
  - elixir
  - ecto
---

![Photo by Mike Benna on Unsplash](/assets/images/ecto-preload-cover.jpg)

This was `TIL` for me, so I thought it would be worth a quick post. 🙂

**TL; DR;**

```elixir
Repo.all from u in User,
         preload: [{:investiment_funds, :institution}, :transfers]
```

Ecto is designed to not bring associations alongside the main data in the query to avoid N+1 problems.

If you want, you need to explicitly say what is that you want to bring together in the query.

To do that you need to use the `preload` macro to load the `associations` into the given query.

Imagine you are at investment scenario where you want to fetch all users and list their investment funds, you would need something like that to `preload` the `investment funds`.

```elixir
Repo.all from u in User,
         preload: [:investiment_funds]
```

Now, imagine you also need to list the transfers that the user did.

```elixir
Repo.all from u in User,
         preload: [:investiment_funds, :transfers]
```

No problem so far, this is all covered by the Ecto documentation.

But now imagine you need to show in the `investment funds` list, the fund `institution` that is stored in a different table.

`User -> Investment Fund -> Institution`

You don't need to use joins or anything like that in order to make it work, you will only need to pass the `nested associations` inside the curly brackets and everything should work now.

```elixir
Repo.all from u in User,
         preload: [{:investiment_funds, :institution}, :transfers]
```

Hope it helps!

**References:**

https://hexdocs.pm/ecto/2.2.10/Ecto.Query.html#preload/3
https://robots.thoughtbot.com/preloading-nested-associations-with-ecto
