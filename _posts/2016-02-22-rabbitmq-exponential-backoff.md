---
layout: post
title:  "RabbitMQ with Exponential Backoff"
date:   2016-02-22 20:00:00 +0100
description: "How to create a proper retry mechanism on top of RabbitMQ using exponential backoff"
tags: rabbitmq
comments: true
---

_Update: As pointed out in the comments by Bryan, you should consider whether the TTL times you'll need are not very long, since per-message TTL have some [caveats][caveats]. The messages get expired only when they hit the head of the queue. For example, if you have a message with 1h TTL on the head of the queue, messages after that will expire only after the one in the head is._

Recently I was looking for a way to handle errors when processing messages from a [RabbitMQ][rabbit] queue. My end goal was to rejecting the message somehow and process it again after some time. Basically this means that I wanted to have a queue with [exponential backoff][backoff].

The initial idea was to fetch the message, process it, handle the errors that the processing might have thrown and then [reject the message][reject-guide].

# Rejecting the message

The example below will process the message, simulate an error and handle the error by [rejecting the message][reject-guide] with [`#reject(delivery_tag, requeue = true)`][reject-api] and redeliver to the end of the queue.

```ruby
require 'bunny'

connection = Bunny.new.tap(&:start)

channel = connection.create_channel
queue   = channel.queue("some.queue")

queue.publish("some message")

queue.subscribe(block: true, manual_ack: true) do |delivery_info, properties, payload|
  begin
    puts "handling message #{payload.inspect}"
    sleep 0.5
    raise "some error when handling the message"
  rescue => e
    puts "rejecting message"
    channel.reject(delivery_info.delivery_tag, true)
  end
end
```

The output will be:

```
handling message "some message"
rejecting message
handling message "some message"
rejecting message
# ... and so on
```

This approach have a few obvious problems:

1. You might end in an infinite loop if this message cannot be processed at all.
2. If the queue is empty, you'll end up processing the message immediately after it was rejected.

# Introducing Retry Limits

Problem #1 is straightforward to solve by simply introducing some kind of counter to limit the number of attempts you try to process the message.

```ruby
queue.subscribe(block: true, manual_ack: true) do |delivery_info, properties, payload|
  begin
    # ...
  rescue => e
    headers      = properties.headers || {}
    retry_count  = headers.fetch("x-retry-count", 0)
    should_retry = retry_count <= 4

    puts "republishing message, retry count: #{retry_count}, retrying? #{should_retry}"

    channel.ack(delivery_info.delivery_tag)
    if should_retry
      queue.publish(payload, headers: { "x-retry-count": retry_count + 1 })
    end
  end
end
```

Note that we can't use [`#reject`][reject-api] here anymore. Rejecting basically puts the same unaltered message back in the queue and we wouldn't be able to change for example its header information, to inform the next process of the current count of retry attempts. Secondly, we'd have to call [`#ack`][ack-api] on the message and then publish again on the same queue.

This solves the possible infinite loop of retrying the same message forever in case your code finds it impossible to process it.

_I used `headers` here because I didn't want to mess up with the payload. Be aware that this does not include headers from the previous message. You should definitely include them when calling `publish`, but for the sake of the example, I'm skipping here._

# Exponential Backoff

Still, we want to solve the problem that we process this message right away instead of giving it some time until the next attempt. The ideal scenario would be:

1. Message #1 comes, we can't process now, so reschedule it to run in 1 second
3. After one second, message #1 still can't be processed, schedule to run in 2 seconds
3. After two seconds, message #1 still can't be processed, schedule to run in 5 seconds
4. This goes on, until the message can be processed or the number of attempts has been exhausted.

Unfortunately [RabbitMQ][rabbit] does not handle this out of the box.

# Enters TTL + Dead-Lettering

Although, it's possible to publish messages with a [TTL][ttl] (time-to-live). This pretty much means that the message will stay in the queue until that specific time _expires_:

```ruby
# message won't stay in the queue longer than 1 second
queue.publish("some message", expiration: 1000)
```

No surprises here. The message above will stay in the queue until 1 second passes. After that, the message will be either discarded or `dead-lettered`.

What we're interested in her is the [dead-lettering][dead-letter] feature.

# Dead-Lettering

From [RabbitMQ][rabbit] [documentation][dead-letter]:

> Messages from a queue can be 'dead-lettered'; that is, republished to another exchange when any of the following events occur:
  - The message is rejected (basic.reject or basic.nack) with requeue=false,
  - The [TTL][ttl] for the message expires; or
  - The queue length limit is exceeded.

With all that info in mind, what we want to achieve is:

* An exchange named `work_exchange`
* An exchange named `retry_exchange` with `x-dead-letter-exchange` pointing to `work_exchange`
* A queue `work_queue` bound to `work_exchange`
* A queue `retry_queue` bound to `retry_exchange`

This means that whenever a message that is on `retry_queue` gets [dead-lettered][dead-letter] (in our case because of expiration), [RabbitMQ][rabbit] will _automatically_ publish the message to the exchange configured on `x-dead-letter-exchange`, in our case `work_exchange`.

In other words, this is what will happen:

* Message #1 is published to the `work_queue`
* We can't process the message now, so we re-publish the message to `retry_queue` with expiration set to 1 second
* After 1 second, the message expires...
* ...then [RabbitMQ][rabbit] automatically pushes the message to `work_exchange`
* Message #1 arrives in the `work_queue` again

# The Code

To achieve that, we'll have to first setup the queues:

```ruby
# Declare both exchanges
work_exchange  = channel.fanout("work_exchange",  durable: true)
retry_exchange = channel.fanout("retry_exchange", durable: true)

# This will be our work queue
work_queue = channel.queue("work_queue").bind(work_exchange)

# This will be queue where will publish messages with TTL
retry_queue = channel
  .queue("retry_queue", arguments: { "x-dead-letter-exchange": 'work_exchange' })
  .bind(retry_exchange)

```

Note that `retry_queue` have the `x-dead-letter-exchange` argument set to the `work_exchange`.

Then our worker will look like this:

```ruby
work_queue.subscribe(block: true, manual_ack: true) do |delivery_info, properties, payload|
  begin
    # ...
  rescue => e
    headers      = properties.headers || {}
    dead_headers = headers.fetch("x-death", []).last || {}

    retry_count  = headers.fetch("x-retry-count", 0)
    expiration   = dead_headers.fetch("original-expiration", 1000).to_i

    puts "failure! retry count: #{retry_count}, retrying? #{should_retry}, expiration: #{expiration}"

    # acknowledge existing message
    channel.ack(delivery_info.delivery_tag)

    if retry_count <= 4
      # Set the new expiration with an increasing factor
      new_expiration = expiration * 1.5

      # Publish to retry queue with new expiration
      retry_queue.publish(payload, expiration: new_expiration.to_i, headers: {
        "x-retry-count": retry_count + 1
      })
    end
  end
end
```

You'll notice that the message gets processed first very quickly, and the next incoming attempt gets gradually delayed more and more.

Whenever a message gets [dead-lettered][dead-letter], it will include a `x-death` header with information about why this happened. For example:

```json
{
  "x-death": [
    {
      "count": 1,
      "reason": "expired",
      "queue": "retry_queue",
      "time": "2016-02-22 22:21:32 +0100",
      "exchange": "",
      "routing-keys": ["retry_queue"],
      "original-expiration": "1500"
    }
  ]
}
```

# Conclusion

I believe this might be a good solution for cases when your workflow depends on [RabbitMQ][rabbit] and you don't want to introduce an additional background job library like Sidekiq or DelayedJob.

What are your thoughts?

[backoff]: https://en.wikipedia.org/wiki/Exponential_backoff
[reject-api]: http://reference.rubybunny.info/Bunny/Channel.html#reject-instance_method
[rabbit]: https://www.rabbitmq.com
[ttl]: https://www.rabbitmq.com/ttl.html
[dead-letter]: https://www.rabbitmq.com/dlx.html
[reject-guide]: http://rubybunny.info/articles/queues.html#rejecting_messages
[ack-api]: http://reference.rubybunny.info/Bunny/Channel.html#ack-instance_method
[caveats]: http://www.rabbitmq.com/ttl.html#per-message-ttl-caveats
