---
layout: post
title:  "Extending Ecto's on_conflict using metaprogramming"
author: "Amanda Sposito"
date:   2021-01-05 9:30:00 -0300
categories:
  - elixir
  - ecto
  - macro
---

One of these days, a friend came to talk to me about an Ecto problem that could be solved using metaprogramming. He was using the [insert_all/3](https://hexdocs.pm/ecto/Ecto.Repo.html#c:insert_all/3) function with the `on_conflict` option so it would become a [upsert command](https://hexdocs.pm/ecto/constraints-and-upserts.html#upserts).

![](/assets/images/extending-ectos-on-conflict/melanie-karrer-T1jw85v_2SE-unsplash.jpg)
Photo by [Melanie Karrer](https://unsplash.com/@fotokarussellmelanie?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText) on [Unsplash](https://unsplash.com/@fotokarussellmelanie?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText)

The problem is that it was necessary to update only the entries that were newer than the current one. To do that, we need a custom query to check the timestamp, but when we use a custom query, we lose the ability to use the `:replace_all` option to replace all the fields. According to the [doc](https://hexdocs.pm/ecto/Ecto.Repo.html#c:insert_all/3):

> `:on_conflict` - It may be one of `:raise` (the default), `:nothing`, `:replace_all`, `{:replace_all_except, fields}`, `{:replace, fields}`, a keyword list of update instructions or an `Ecto.Query` query for updates.

If we want to replace all the fields and still use the `Ecto.Query`, we will need to update all the fields manually, and this is error-prone.

```elixir
def update_user(users) do
  Repo.insert_all(
    User,
    users,
    on_conflict: on_conflict_query(),
    conflict_target: :last_event_id
  )
end
```

```elixir
def on_conflict_query do
  User
  |> where([u], f.last_event_timestamp < fragment("EXCLUDED.last_event_timestamp"))
  |> update([u],
    set: [
      first_name: fragment("EXCLUDED.first_name"),
      last_name: fragment("EXCLUDED.last_name"),
      phone_number: fragment("EXCLUDED.phone_number"),
      gender: fragment("EXCLUDED.gender"),
      date_of_birth: fragment("EXCLUDED.date_of_birth"),
      city: fragment("EXCLUDED.city"),
      state_code: fragment("EXCLUDED.state_code"),
      email: fragment("EXCLUDED.email"),
      full_profile: fragment("EXCLUDED.full_profile"),
      last_event_id: fragment("EXCLUDED.last_event_id"),
      last_event_timestamp: fragment("EXCLUDED.last_event_timestamp"),
      last_event_action: fragment("EXCLUDED.last_event_action"),
      updated_at: fragment("EXCLUDED.updated_at")
    ]
  )
end
```

Let's see how we can use metaprogramming to solve this.

You can skip to the final solution [here](#extending-ectos-on_conflict) if you want to.

### Metaprogramming to the rescue

#### Where do I start?

Elixir is a [AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree) based programming language. Meaning that when our code is compiled, its source is transformed into a tree structure, this structure is exposed in a form that can be represented in Elixir's own data structure.

Metaprogramming is the ability to write code using code, so we can extend the language to dynamically change the code, and we do it by manipulating Elixir's AST.

### The quote

The [quote](https://hexdocs.pm/elixir/Kernel.SpecialForms.html#quote/2) macro is responsible to get the representation of any expression. It returns a tuple with three elements:

1. An atom or another tuple in the same representation;
2. A keyword list of values that represents Elixir [metadata](https://hexdocs.pm/elixir/Macro.html#t:metadata/0) about the function;
3. The arguments to the function call.

```
iex(1)> quote do
...(1)>   1 + 1
...(1)> end
{:+, [context: Elixir, import: Kernel], [1, 1]}
```

In this example, the tuple represents a function call to the Kernel [arithmetic addition function](https://hexdocs.pm/elixir/Kernel.html#+/2), passing two arguments, 1 and 1.

### The unquote

The [unquote](https://hexdocs.pm/elixir/Kernel.SpecialForms.html#unquote/1) macro is responsible for injecting an AST expression into the AST. In the "Metaprogramming Elixir" book, Chris McCord says:

> You can think of quote/unquote as string interpolation for code. If you were building up a string and needed to inject the value of a variable into that string, you would interpolate it. The same goes when constructing an AST. We use quote to begin generating an AST and unquote to inject values from an outside context. This allows the outside bound variables, expression and block, to be injected directly into our if ! transformation.

-- *Chapter 1. The Language of Macros. Metaprogramming Elixir: Write Less Code, Get More Done (and Have Fun!), by Chris McCord, The Pragmatic Bookshelf, 2015.*

```
iex> number = 1
1

iex> "this is an interpolation of #{number}"
"this is an interpolation of 1"

iex> quote do
...>   unquote(number) + 1
...> end
{:+, [context: Elixir, import: Kernel], [1, 1]}
```

We can see that when we use the `unquote` macro, it injects the `number` value into the AST, just like it does when we are interpolating a string.


```
iex> number = 1
1

iex> "this is an interpolation of number"
"this is an interpolation of 1"

iex> quote do
...>   number + 1
...> end
{:+, [context: Elixir, import: Kernel], [{:number, [], Elixir}, 1]}

```

When we don't use the `unquote` macro it includes `:number` into the AST, instead of the value that we want.

```
iex> Macro.to_string(quote(do: number + 1))
"number + 1"

iex> Macro.to_string(quote(do: unquote(number) + 1))
"1 + 1"
```

### Macros

According to the documentation [Macros](https://hexdocs.pm/elixir/Macro.html#content) are compile-time constructs that are invoked with Elixir's AST as input and a superset of Elixir's AST as output. Once the construct finishes its processing, the response will then be injected back into the application.

```
defmodule MyModule do
  defmacro increment(number) do
    quote(do: unquote(number) + 1)
  end
end
```

```
iex> MyModule.increment(1)
2
```

### Extending Ecto's on_conflict

Back to our problem, the [update/3](https://hexdocs.pm/ecto/Ecto.Query.html#update/3) function expects a `set` operator with a [keyword list](https://hexdocs.pm/elixir/Keyword.html) with field-value pairs as values.

```elixir
User
|> update([u],
  set: [
    first_name: fragment("EXCLUDED.first_name")
  ]
)
```

We need to write a macro that will take a list of fields and build a fragment passing the field's name.

For every schema that we create on Ecto, we will have a [__schema__](https://hexdocs.pm/ecto/Ecto.Schema.html#module-reflection) function that can be used for runtime introspection of the schema. We will use it to fetch all non-virtual fields from the schema.

```elixir
User.__schema__(:fields)
```

Now that we have a list of all the fields we want to update, we will use `quote` and `unquote` to dynamically build a fragment with [Postgres `EXCLUDED` table](https://www.postgresql.org/docs/current/sql-insert.html) to reference what's being updated.

```elixir
:fields
|> User.__schema__()
|> Enum.map(fn f ->
  {f, quote(do: fragment(unquote("EXCLUDED.#{to_string(f)}")))}
end)
```

After that, we will pass the values we built for the `set` operator. Instead of `unquote` this time, we will use [`unquote_splicing`](https://hexdocs.pm/elixir/Kernel.SpecialForms.html#unquote_splicing/1) because we are passing a list of expressions, instead of one expression.

```elixir
defmacro custom_on_conflict_update_replace_all(queryable) do
  values =
    :fields
    |> User.__schema__()
    |> Enum.map(fn f ->
      {f, quote(do: fragment(unquote("EXCLUDED.#{to_string(f)}")))}
    end)

  quote(do: Ecto.Query.update(unquote(queryable), [u], set: [unquote_splicing(values)]))
end
```

We are almost done now, we just need to call the macro on the `on_conflict_query/0`.

```elixir
def on_conflict_query do
  User
  |> where([u], f.last_event_timestamp < fragment("EXCLUDED.last_event_timestamp"))
  |> custom_on_conflict_update_replace_all()
end
```

The final solution:

```elixir
def update_user(users) do
  Repo.insert_all(
    User,
    users,
    on_conflict: on_conflict_query(),
    conflict_target: :last_event_id
  )
end

defmacro custom_on_conflict_update_replace_all(queryable) do
  values =
    :fields
    |> User.__schema__()
    |> Enum.map(fn f ->
      {f, quote(do: fragment(unquote("EXCLUDED.#{to_string(f)}")))}
    end)

  quote(do: Ecto.Query.update(unquote(queryable), [u], set: [unquote_splicing(values)]))
end

def on_conflict_query do
  User
  |> where([u], f.last_event_timestamp < fragment("EXCLUDED.last_event_timestamp"))
  |> custom_on_conflict_update_replace_all()
end

```

Now, every time we make a change on the `User` schema we don't need to bother to come back and change this function.

I hope it helps!

