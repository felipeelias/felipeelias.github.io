---
layout: post
title:  "A new scheduler for Eventboss"
date:   2019-11-04
description: "Eventboss is a Pub/Sub built on top of SNS and SQS. This is the rationale behind the new scheduler design."
tags: [ruby, aws, sqs, eventboss, refactoring, threadpool, scheduler]
categories: ruby
comments: true
---

> Eventboss is a Pub/Sub built on top of SNS and SQS. It's one of the main components used in AirHelp to handle service-to-service communication. In this post, I'll go through the internals of the subscriber component, which handles consuming messages from SQS.

Shortly before we open-sourced the Eventboss gem, I decided to invest some time improving the subscriber internals and making it more stable.

The old Eventboss subscriber was simple. It iterated over a list of queues, took all messages from the queue, and for each message, called the appropriate listener class. The listener class is defined per queue and holds the business logic of the service.

However, the gem processed messages unevenly because it was reading all messages from a single queue before moving to the next one. In the far past, this was not a problem. Still, as AirHelp grew, the situation was more noticeable, especially in complicated services when multiple queues were involved.

## The new scheduler

To achieve better fairness, I decided to rewrite the consumer part and introduce a new scheduler. Instead of having a shared thread pool for fetching and processing the messages, I decided to separate fetch and process on dedicated thread pools and connect those via an internal queue.

![eventboss-design](/assets/images/eventboss-fetcher-and-workers.png)

Fetcher's responsibility is to retrieve messages from SQS and push to the internal queue. Each request fetches at most 10 messages at once (which is the SQS limit).

```ruby
class Fetcher
  # ...
  def fetch(queue, limit)
    @client.
      receive_message(queue_url: queue.url, max_number_of_messages: max_no_of_messages(limit)).
      messages
  end
end
```

Each fetcher runs in a separated thread and handles a single SQS queue, using long polling. That allowed us to have fetchers working in parallel, avoiding the first problem I encountered, which was only a single queue fetched at a time.

Before pushing a message to the internal queue, the fetcher wraps the message in a "unit of work". The unit of work is responsible for initializing listeners, logging, monitoring, and acknowledging or postponing the message. In case of failure to acknowledge the message, SQS makes the message available to fetchers once again.

```ruby
class LongPoller
  # ...
  def fetch_and_dispatch
    fetch_messages.each do |message|
      @bus << UnitOfWork.new(queue, listener, message)
    end
  end
end
```

Workers run in a simple thread pool and do not have any knowledge of what kind of work they're doing. This approach simplified the design a lot, with the benefit of letting us scale the number of workers as we needed.

```ruby
class Worker
  def run
    while (work = @bus.pop)
      work.run
    end
  rescue ...
    # handle shutdown and errors
  end
end
```

The internal queue (here called `@bus`) is a bounded one, which gives backpressure for free. When the queue is full, pushing work to the queue is blocked until the queue has free space.

```ruby
@bus = SizedQueue.new(@queues.size * 10)
```

The queue size has a factor of 10 to let each fetcher to enqueue all its messages at once.

## Guarantees

I was able to simplify the code considerably with this design. By having separated thread pools for fetching and processing, I got a system that is much more predictable. Setting the `concurrency` option now tells exactly how many messages Eventboss consumes concurrently.

SQS provides good guarantees, which are worth mentioning:

1. Messages can be delivered at least once.
2. Messages have to be acknowledged in up to 15 minutes. This limitation helps with handling failures but also means that workers can't spend over 15 minutes handling a message.

These limitations forced us to process messages fast. Any message that requires a long time for processing is usually pushed to another system, like Sidekiq.

## Conclusion

This approach is straightforward and very performant. Every message has the same priority, and the work is evenly distributed across all worker threads.

By going this way, I also removed the dependency on concurrent-ruby (great gem, but overkill for approach). Now Eventboss only depends on AWS SDK, which is a massive win in maintainability.

And you, what do you use for service-to-service communication? Let me know in the comments.
