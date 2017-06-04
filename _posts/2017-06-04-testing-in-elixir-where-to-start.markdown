---
layout: post
title:  "Testing in Elixir, where to start?"
author: "Amanda Sposito"
date:   2017-06-04 16:17:37 -0300
categories:
  - elixir
  - test
  - exunit
---

After studying Elixir for a while and understanding how it works, I came across some questions about how writing tests would be in a functional language and where to start.

### ExUnit

To begin with, Elixir comes with ExUnit, a test framework that provides pretty much everything we need to test our code.

### How to use it ?

Everytime we start a project with `mix new project_name` we can see the tests folder in its output:

{% highlight bash %}
$ mix new hello_world
* creating README.md
* creating .gitignore
* creating mix.exs
* creating config
* creating config/config.exs
* creating lib
* creating lib/hello_world.ex
* creating test
* creating test/test_helper.exs
* creating test/hello_world_test.exs

Your Mix project was created successfully.
You can use "mix" to compile it, test it, and more:

    cd hello_world
    mix test

Run "mix help" for more commands.
{% endhighlight %}

If we open the  `hello_world_test.exs` file, we will find our tests basic structure.

{% highlight elixir %}
defmodule HelloWorldTest do
  use ExUnit.Case
  doctest HelloWorld

  test "the truth" do
    assert 1 + 1 == 2
  end
end
{% endhighlight %}

RIght at the beginning we can see that there is a line  `use ExUnit.case`, **ExUnit.case** is a module that we should use in another modules so we can configure and prepare them to be tested.

The test itself it is pretty simple and self-explained, but it already give us an idea of the structure we can use to write our tests.

### Assertions

There is some `asserts` that can help us to execute our tests. The most common being `assert` and `refute`.

`Assert` to compare something it is expected to be true and `refute` to compare something that is expect to be false.

| Assertion | Examples |
|-------|--------|
| `assert` | `assert to_string(:foo) == "foo"` |
| `assert_in_delta` | `assert_in_delta 3.141, 3.142, 0.001` |
| `assert_raise` | `assert_raise ArithmeticError, fn -> 1 + "test" end` |
| `assert_received` | `assert_receive :down, 500` |
| `assert_received` | `assert_received :hello, "Hi!"` |
| `refute` | `refute 1+1, 3` |
| `refute_in_delta` | `refute_in_delta 3.141, 3.142, 0.001` |


The `asserts` are pretty much similar to most framework tests, the hightlight here is for `assert_receive` and`assert_received`.

They help us to test async code, for example, you have a `Task` that executes some code and you wishe to monitor the  process and wait for some message. In this scenario, `assert_receive` would help us to test that we received the message.

{% highlight elixir %}
task = Task.async(fn -> "foo" end)
ref  = Process.monitor(task.pid)
assert_receive 	{:DOWN, ^ref, :process, _, :normal}, 500
{% endhighlight %}

In the above test, we are asserting that the Task exit is `:normal` and that it executed in up to 500 milliseconds.

There are a few more `asserts` that may help us and you can find them at the [ExUnit oficial documentation](https://hexdocs.pm/ex_unit/ExUnit.Assertions.html).

### Describe

To group our tests, there is a function called `describe` and It is a very used convention to organize the tests in a group of examples by function.

{% highlight elixir %}
describe "Some.Function" do
  test "does something" do
    ...
  end
end
{% endhighlight %}

Using Mix, you can run only one block of `describe` if you want to:

`$ mix test --only describe:"Some.Function"`

### Test Setup

In some cases, it may be necessary to do some kind of `setup` of the  `system under test` before run our tests. For this, there are two `macros` that can help us: `setup` and `setup_all`.

The difference between them is that `setup` is executed everytime before a test and `setup_all` is executed just one time before the `module` tests.

{% highlight elixir %}
  setup do
    ...
  end

  setup_all do
    ...
  end
{% endhighlight %}

### Running your test suit

There are a few ways we can run our test suit, to run the whole test suit we can use the command `$ mix test`

We can execute only one file, we just need to pass the file path to the command `$ mix test path/to/file.exs`.

We can also execute only one test of the file, passing the test line as argument `$ mix test path/to/file.exs:line`

Besides these options, you can also use a `tag` to mark your test and use it, either to skip it or run only it, according to what you need.

{% highlight elixir %}
defmodule HelloWorldTest do
  use ExUnit.Case
  doctest HelloWorld

  @tag :wip
  test "the truth" do
    assert 1 + 1 == 2
  end
end
{% endhighlight %}

To run only the tests marked as `wip`:

 `$ mix test --only wip`

Or, to exclude them:

`$ mix test --exclude wip`

If necessary, we can also configure our test suit to exclude tests with certain tags, to do it so we need to alter our `test_helper.exs` and add a config.

{% highlight elixir %}
ExUnit.configure(exclude: [wip: true])
ExUnit.start()
{% endhighlight %}

So when we run our test suit, all the tests with the `wip` tag will be ignored.

https://hexdocs.pm/mix/Mix.Tasks.Test.html

### Mocks and Stubs

The truth is, if you're used to writing tests in your day-to-day life, there's a good chance you've used mocks and stubs to handle contracts or simulate behaviors.

Some testing frameworks such as Ruby's `Minitest` or` RSpec` already come with options for mocks and stubs.

`ExUnit`, however, does not come with anything for that, [in this post](http://blog.plataformatec.com.br/2015/10/mocks-and-explicit-contracts/) José Valim explains why the use of mocks can be harmful to the design of your application.

In short terms, all Elixir applications come with standard configuration files and we can often use them to configure how a particular dependency will behave in different environments and use it in our tests, removing the need to use mock.

But sometimes this will not be enough, and if you still need to use mocks or stubs, there are some frameworks like [Meck](https://hex.pm/packages/meck) that can help you.

---

### Reference

[https://hexdocs.pm/ex_unit/ExUnit.html](https://hexdocs.pm/ex_unit/ExUnit.html)

[http://blog.plataformatec.com.br/2015/10/mocks-and-explicit-contracts/](http://blog.plataformatec.com.br/2015/10/mocks-and-explicit-contracts/)
