---
layout: post
title: Using Elixir Telemetry and Tesla
date: 2022-08-03
description: >-
  How to set up Elixir Telemetry using the default Tesla Middleware and group requests by the URL template.
tags: [elixir, tesla, telemetry]
categories: elixir
comments: true
archived: true
---

Tesla provides a [Telemetry] middleware which is pretty straightforward to configure for your clients. However, I find that the middleware on its own is not sufficient to get more **detailed metrics**.

My goal here is to be able to extract the domain and the path in the metrics. This post will show you how to do that.

## API Client

As an example, we'll create an API client to [hex.pm].

```elixir
defmodule MyApp.HexPm do
  use Tesla, only: [:get], docs: false

  # required middleware for basic requests to hex.pm
  plug Tesla.Middleware.BaseUrl, "https://hex.pm/"
  plug Tesla.Middleware.Headers, [{"user-agent", "MyApp/0.0.1"}]
  plug Tesla.Middleware.JSON, decode_content_types: ["application/vnd.hex+json"]

  # required middleware for telemetry
  plug Tesla.Middleware.KeepRequest
  plug Tesla.Middleware.Telemetry, metadata: %{api: "hex.pm"}
  plug Tesla.Middleware.PathParams

  def get_package(name) do
    get("/api/packages/:name", opts: [path_params: [name: name]])
  end
end
```

The middleware that will enable [Telemetry] events are:

[`KeepRequest`][Tesla.Middleware.KeepRequest]
: Store request **URL**, **body** and **headers** into `:opts`. The options in our case would look like this:

```elixir
[
  req_url: "https://hex.pm/api/packages/:name",
  req_headers: [{"user-agent", "MyApp/0.0.1"}],
  req_body: nil,
  path_params: [name: "phoenix"]
]
```

[`PathParams`][Tesla.Middleware.PathParams]
: Use templated URLs with separate params. In our example, it enables `/api/packages/:name` to receive the `:name` dynamically when sending the request but allows to group requests by the _template_.

[`Telemetry`][Tesla.Middleware.Telemetry]:
: Emits events using the [`:telemetry`][Telemetry] library to expose instrumentation. This middleware automatically adds the `env` to the event, but extending it with the `metadata` option is also possible. In our case, I added `%{api: "hex.pm"}`, which will help group requests by their "api". It's helpful if you have multiple Tesla clients with distinct APIs.

## Telemetry Configuration

Another necessary step is to add the proper configuration to your app's [Telemetry] configuration:

```elixir
defmodule MyApp.Telemetry do
  use Supervisor
  import Telemetry.Metrics

  # ...
  @impl true
  def init(_arg) do
    # starting ConsoleReporter to see metrics in IEX console
    children = [{Telemetry.Metrics.ConsoleReporter, metrics: metrics()}]
    Supervisor.init(children, strategy: :one_for_one)
  end

  def metrics do
    [
      # your other metrics...
      summary("tesla.request.stop.duration",
        unit: {:native, :millisecond},
        tags: [:api, :method, :req_url],
        tag_values: &tesla_tag_values/1
      )
    ]
  end

  # extract the tags from the env + middleware metadata
  defp tesla_tag_values(meta) do
    %{api: meta.api, method: meta.env.method, req_url: meta.env.opts[:req_url]})
  end
end
```

This configuration enables a few things:

`tags`
: Specify which tags to extract from the `metadata`. We're specifying `req_url` here, but it's unavailable at the event's root. We need to extract it using `tag_values`.

`tag_values`
: A function that processes the event `metadata`. Because our `req_url` is deep inside the `meta.env.opts`, we need to extract it here. The map returned will be available to the `tags` function.

Running the example in the `iex` console:

```elixir
iex> MyApp.HexPm.get_package("phoenix")
[Telemetry.Metrics.ConsoleReporter] Got new event!

Event name: tesla.request.stop
All measurements: %{duration: 188736000}
All metadata: %{api: "hex.pm", env: ...}

Metric measurement: #Function<.../1 in Telemetry.Metrics.maybe_convert_measurement/2> (summary)
With value: 188.736 millisecond
Tag values: %{api: "hex.pm", method: :get, req_url: "https://hex.pm/api/packages/:name"}
```

An important note: reporters are responsible for extracting tags and tag_values, and each reporter may implement it differently. Check the reporter implementation to ensure it applies the proper tags transformation!

### Links

- [hex.pm]
- [Telemetry.Metrics][Telemetry]
- [Tesla]
- [Tesla.Middleware.Telemetry]
- [Tesla.Middleware.PathParams]
- [Tesla.Middleware.KeepRequest]
- [How to setup "URL event scoping"][URLScoping]

[hex.pm]: https://hex.pm
[Telemetry]: https://hexdocs.pm/telemetry_metrics/Telemetry.Metrics.html
[Tesla]: https://hexdocs.pm/tesla/readme.html
[Tesla.Middleware.Telemetry]: https://hexdocs.pm/tesla/Tesla.Middleware.Telemetry.html
[Tesla.Middleware.PathParams]: https://hexdocs.pm/tesla/Tesla.Middleware.PathParams.html
[Tesla.Middleware.KeepRequest]: https://hexdocs.pm/tesla/Tesla.Middleware.KeepRequest.html
[URLScoping]: <https://hexdocs.pm/tesla/Tesla.Middleware.Telemetry.html#module-url-event-scoping-with-tesla-middleware-pathparams-and-tesla-middleware-keeprequest>
