---
title: Unit testing integrations with external services
date: 2023-11-11
tags: ["elixir", "testing", "mox", "bypass", "req"]
toc: true
draft: true
---

## The problem

Your Elixir app is making connections to the outside world, and that modules need to be tested as well! Lets say you want to integrate with SmartAccounts Currency API to pull currency rates.

We would have a `App.Currency` module with a function that makes the request, so that our signature is something like:

```elixir
%{status: 200} = App.Currency.get_rates()
```

There are a few possible solutions for testing this code, and in this blog post we will first go thru the conceptual solutions, pick one and implement it in our demo app.

You can follow along with this blog post by using the Github repo I prepared.

## Conceptual solutions

### Making real requests

Provided you actually use the service and integrate aginst the response, it's probably the best to let your tests make real requests against a testing enviroment a service offers.

This will **with every run** assert that the response is what we expect and that our integration with that request actually works.

Now this isn't always possible, and from my expirience there are a few problems:

- not all services you're going to use are fast enough, having a test suite of 100 tests making real requests to the testing environment can take hours to run in the worst case scenario
- not all services offer a testing environment that you can use like it's ephemeral
  - in this case now you need to seed, and clean the env after each use
  - and this could also mean your tests can't run async

### Mocking the request module

You could always just mock the http module that does the reqest. This will prevent the code actually ever making the request. I would call this solution "the polar oposite of the previous one".

This soultion relys on you specifying the return value of a http library, meaning that the choice of an http library is no longer an implementation detail of your `Currency` module.

Let's look at a quick example so you can better understand what I mean. Given this code:

```elixir
defmodule App.Currency do
  def get_rates do
    Req.get!("https://smartaccount.hr/api/rates?currency=USD")
  end
end
```

In the test using the `mock` library as an example:

```elixir
test "returns the rates" do
  with_mock Req,
    [get!: fn(_url) -> %Req.Response{status: 200, body: %{}} end]
  do
    assert %Req.Response{status: 200} = App.Currency.get_rates()
  end
end
```

Now this approach has a few issues I don't like:

- while we are providing the body and testing that we use the body correctly, if we ever want to switch out our http library for what ever reason we now need to go and modify the tests as well
- the way how `mock` library works is requring your tests are run sync, which in small codebases might actually be ok, but in larger test suites it's going to slow down exectuion significantly
- we need to mock either the http library or the `App.Currency` module everywhere we use it

### Mocking the response

And finally, mocking the response. This means we want to intercept either the HTTP request or http client methods and make it return a repsonse we can later use. This is both the easiest option to maintain and in my opinion to use.

One of the libraries in this space is exvcr which is a great idea, but unfortunalty it falls short by using mock in the background with all it's downsides.

Today we're going to explore the solution that uses `bypass` and `mox` to mock the response from the external service, while allowing our http libraries to actually make a request.

## Our example

In our example we're going to be writing a module that will make a `GET` request to `https://smartaccount.hr/api/rates?currency=USD`. For this example I'm going to use `req` but you can really use any http library for this.

We need to make the base url (the `https://smartaccount.hr/api` part of the URL) configurable. We're going to use our `config/config.exs` file for this:

```diff
diff --git a/config/config.exs b/config/config.exs
index 871a3d1..4e72929 100644
--- a/config/config.exs
+++ b/config/config.exs
@@ -1,3 +1,5 @@
 import Config

+config :app, App.Currency, base_url: "https://smartaccount.hr/api"
+
 import_config "#{Mix.env()}.exs"
```

And then we can use this in our module:



## Introducing `bypass`

## Setting up config

## Setting up tests

## Mocking the response json
