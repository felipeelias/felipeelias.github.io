---
layout: post
title:  "Net::HTTP and Timeouts"
date:   2017-08-20
description: "How to set sane values for Net::HTTP read and open timeout options"
tags: ruby
categories: ruby
comments: true
---

Ruby's `Timeout` [is considered dangerous][1]! That's sad because I have tons of HTTP requests wrapped in `Timeout::timeout {}` and it seems that the alternative out there is to use `open_timeout` and/or `read_timeout`. My quest is to understand how to use them because they don't work _exactly_ the same as wrapping your HTTP request into a `Timeout` block.

What are sane values for those two options, and how to choose them? TL;DR, it depends!

## The gates are open

In the context of this post, I was playing only with [`Net::HTTP`][6] and I know that [there are some surprises there][5], but I'm not going to enter into details.

The `open_timeout` option only ensures that opening a TCP socket doesn't take as long as the timeout. That seems simple enough to understand and probably no more than a few seconds is fine. I'm more interested in finding a sane value for `read_timeout`, especially because it works differently than I expected.

## What read_timeout means?

In Ruby's [`Net::HTTP`][6], the `read_timeout` says how long the code should wait for data to be ready to read in the socket.

In the usual flow, the client ([`Net::HTTP`][6]) writes the HTTP request to the socket and then waits for the response from the server. It will take some amount of time since the request for the first byte of the response to come, so the client blocks and waits for data to appear in the socket. And it is at that moment that the read timeout takes effect.

If you `strace` a simple [`Net::HTTP`][6] call that looks like this:

```ruby
Net::HTTP.start(host, port, read_timeout: 10) do
  # ...
end
```

You'll see something like this (I omitted a lot of stuff):

```
# Write the request to the file descriptor 7 (our socket)
write(7, "GET / HTTP/1.1\r\nUser-Agent"..., 167) = 167

# Immediately try to read from that socket, but nothing is ready yet
read(7, 0xFFFFFFFFF, 16384) = -1 EAGAIN (Resource temporarily unavailable)

# Blocks waiting for data on the socket
ppoll([{fd=7, events=POLLIN}], 1, {10, 0}, NULL, 8)
                                   ^^ timeout of 10
```

The `ppoll` system call blocks waiting for data on the socket. It's a good place to put a timeout because you don't want to block forever. In this example, if the wait time takes more than 10 seconds, the system call will timeout and Ruby [will throw a `Net::ReadTimeout` error][3], which is fine. The only caveat here is that there might be **many `ppoll` calls**.

## But how many?

That, depends.

If the server you are communicating with is returning the response in chunks ([with `Transfer-Encoding: chunked` header][4]) there is a higher chance of hitting the scenario when the code tries to read data from the socket, nothing is there yet, and it Ruby will block and wait on the socket for changes.

For example:

```ruby
class Server
  def call(env)
    delay = 1 # second

    body = Enumerator.new do |enum|
      5.times do |i|
        puts "Sleeping #{delay}"
        sleep delay
        enum << "#{i}\n"
      end
    end

    ['200', {'Content-Type' => 'text/plain'}, body]
  end
end

Rack::Handler::Puma.run Server.new
```

This server sleeps for 1 second, writes a number to the response and repeats 5 times. A [`Net::HTTP`][6] client making a request to this server will issue 6 `ppoll` calls, one for the headers and 5 additional for each chunk of data.

If you set `read_timeout` to 2 seconds, it does not guarantee that the whole request won't take more than 2 seconds. In this scenario, a short timeout makes more sense.

If the server you're communicating with receives the request, goes away for some time and spits out the response back to you all at once (like most), it's very likely that the first `ppoll` will wait for the whole request to be processed. Then it only depends on how fast the data is coming from the network. In this scenario, a longer timeout makes more sense.

## So, the conclusion is...?

The conclusion is that setting a `read_timeout` to `x` seconds does not guarantee that the **whole** request will timeout after `x` seconds. It depends on how many times the `ppoll` is called, or in other words, how often the code is waiting for data to be ready to read in the socket.

Is that a huge problem? I am not sure. Maybe in most of the cases, all data is available on the socket and the code will just wait once, so using a 5-second timeout is quite similar as setting a `Timeout` block to 5 seconds.

Maybe there are alternatives to `read_timeout` and to `Timeout`, but I lack the knowledge about it. If you know a better strategy, let me know in the comments.

I had a lot of fun trying to test the various scenarios in this post, and I learned a lot more about Ruby! By no means I'm an expert on the subject, if you think I said something very wrong, let me know!

[1]: http://www.mikeperham.com/2015/05/08/timeout-rubys-most-dangerous-api/
[2]: https://github.com/ruby/ruby/blob/17b3441ac4ad1f10bfc7ed1ab88b14ada9ec3e7d/lib/net/http.rb#L903-L910
[3]: https://github.com/ruby/ruby/blob/cbedbaf9d9c7452afac2dfc0f06e1f72e235adea/lib/net/protocol.rb#L181
[4]: https://en.wikipedia.org/wiki/Chunked_transfer_encoding
[5]: https://jvns.ca/blog/2016/03/04/whats-up-with-ruby-http-libraries/
[6]: https://ruby-doc.org/stdlib-2.4.1/libdoc/net/http/rdoc/Net/HTTP.html
