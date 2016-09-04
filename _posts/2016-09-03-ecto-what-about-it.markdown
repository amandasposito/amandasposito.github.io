---
layout: post
title:  "Ecto, what about it?"
date:   2016-09-03 08:40:37 -0300
categories:
  - elixir
  - database
  - ecto
---
If you are studying Elixir and thought about how to connect with the database, you’ve probably heard of it.

From it’s github page: “Ecto is a domain specific language for writing queries and interacting with databases in Elixir.”

Basically, it is a lib you can use to interact with your database. Think about it as a layer between the application and the database itself.

It comes with a series of facilities and you can do things like writing migrations, insert, delete, update and querying your data, but most important, don’t think about it as if it was your ORM.

It is common people coming to Elixir from a OOP background and so thinking Ecto is your ORM. It is not.

ORM means object-relational mapping, it’s about objects. Elixir is a functional programming, there is no objects there. The idea is not to map your table into an object, the idea is to map your data source as needed.

It is split into 4 main components:

* Repo
* Schema
* Changeset
* Query

## Repo

Is your data storage, every time you want something from your database, you go to the repository. Think about it as a mediator between the domain and the access layer data.

Through the repo you will execute your queries. There is some functions you can use, such as: insert, insert\_all, delete, delete\_all, update, update\_all.

[https://hexdocs.pm/ecto/Ecto.Repo.html](https://hexdocs.pm/ecto/Ecto.Repo.html)

## Schema

It maps the source data into a Elixir structure. You define it once and can use to fetch data and coordinate changes.

In Ecto's first version, it was called Model, but in favor of a more data focused mindset, they changed it.

{% gist https://gist.github.com/amandasposito/d15213f63e89057646c01098ac7100f9#file-schema-ex %}

One thing that is important to be said, is that you don’t always need a schema to perform queries. You may want to do something that doesn’t necessary needs a pre-defined structure, for example, a report query.

Also, you can create different schemas depending on what you want to do. For example, you can create a schema to read and another one to write.

[https://hexdocs.pm/ecto/Ecto.Schema.html](https://hexdocs.pm/ecto/Ecto.Schema.html)

## Changeset

This one caused me a lot of trouble at the beginning. I couldn’t figure it out why did I need it in the first place. I came from a OOP background and when I want to validate something, the errors are stored in the object.

![mindblown](/assets/images/mindblown.gif)

Once again, there is no objects in Elixir, and then you can’t store validation data into the object. So that is what Changeset it is about, it is a guarantee that we wont insert incorrect data into the database. Think about it as a collection of changes.

> Changesets allow filtering, casting, validation and definition of constraints when manipulating structs.

{% gist https://gist.github.com/amandasposito/a52e61e6d4f9cbf55cfdc9cef1f7f1b1#file-changeset-ex %}

![ecto-changset-validation-error](/assets/images/ecto-changset-validation-error.png)

[https://hexdocs.pm/ecto/Ecto.Changeset.html](https://hexdocs.pm/ecto/Ecto.Changeset.html)

## Query

Written in Elixir syntax, they generate a SQL instruction, to retrieve information through the repo. The queries are sanitized and **protected from SQL Injection**.

{% gist https://gist.github.com/amandasposito/862636917da9ee7c4374908c1c45d159#file-simple_query-ex %}

Also, the queries are **composable**, you can continue to use a query in another part of the code if you want to.

{% gist https://gist.github.com/amandasposito/b427c446765800a750a4a7baa486ade0#file-composable-ex %}

One thing that is important to be said, it is that Ecto is **not lazy load** and it is designed to be this way to avoid N+1 problems.

When you have something that is lazy loaded, you don’t stop to think about how it will behave at the database level and it brings performance issues.

> “A lot applications have N+1 query issues, because things can be lazy loaded automatically, you basically don’t care and then when you see you are loading a huge chunk of your database dynamically and in a non performant way at all.” José Valim — [(https://changelog.com/208)](https://changelog.com/208)

So, if you want to bring together the association data, you need to tell the query to do so, using the preload. There is some ways you can do that.

In the example bellow, we are telling the query to bring the class association together. In this case it will perform two queries, one for fetching the courses and another one to fetch the classes.

{% gist https://gist.github.com/amandasposito/e38178bb9237348df9a71e8db809769b#file-query_preload-ex %}

But very often you want them to come in the same query. You can do this by adding this join to query. It will fetch the class records in a inner join.

{% gist https://gist.github.com/amandasposito/0a7b0eea6039d7524ee0f86a9f538ac3 %}

You can use `left_join`, `right_join` and `full_join`; the inner join is the default.

You can pass other queries to the preload. For example if you want to filter or customize how it will bring the results:

{% gist https://gist.github.com/amandasposito/9eea6e729ee63ad3fae34d592e6668e1#file-preload_query-ex %}

In the example above, it will fetch two queries, one for the courses and another one to bring the associated classes ordered by the creation date.

These are the main parts about Ecto, it is a great tool and it brings a lot of things that can help you interact with your data source, and it is definitely worth taking a look.

There was a lot of good changes in the second version, and it is a good ideia to take a look at the documentation, it is a great resource to understand better how it works.

Hope it helps.

![thats-all-folks](/assets/images/thats-all-folks.gif)
