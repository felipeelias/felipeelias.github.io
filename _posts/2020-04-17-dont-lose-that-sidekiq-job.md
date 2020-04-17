---
layout: post
title: "Don't lose that Sidekiq job (and your job)"
date: 2020-04-17
description: >-
  Can you rely solely on Sidekiq if you need to have reliable job scheduling?
tags: [ruby, sidekiq]
categories: ruby
comments: true
---

Many of you have already encountered during your software development career, the following task:

1. write something to the database
2. schedule a job to run asynchronously, because of the database change

A simple concrete example may be creating a user and sending a welcome email in the background. Or creating an order, and charging the user in the payment system. Or updating a record in the database and publishing a notification of the change to a broker like [SQS], [RabbitMQ] or [Kafka].

The premise is that you make a write to the database first and an additional write to a second system, that being another database or a completely separated process.

If you're coming from the Ruby world, you may have seen this pattern often, when using [Sidekiq]. For example:

```ruby
def create_user
  user = User.create!(attributes)
  WelcomeEmailJob.perform_async(user.id)
end
```

The code is self-descriptive. You create a user in the database and send a "welcome" email asynchronously, using [Sidekiq].

For the sake of the example, let's say that the email contains important instructions for the newly created user and it absolutely needs to be sent or your business will fail.

Internally, [Sidekiq] uses [Redis] to store its jobs. It may not be obvious at first, but that [Redis] may be unavailable and when it does, you'll lose the email job.

Or worse, your process may crash before it can schedule the job in [Sidekiq]. That is a tough spot to be in because you won't be able to even handle the error and write it to a log file.

```ruby
def create_user
  user = User.create!(attributes)
  # your process crashes here, the next line will never be reached
  WelcomeEmailJob.perform_async(user.id)
end
```

Wrapping that method in a transaction here does not solve the problem. It instead, introduces other problems.

Explicitly, a transaction looks like this:

```ruby
def create_user
  transaction do                            # BEGIN TRANSACTION
    user = User.create!(attributes)
    WelcomeEmailJob.perform_async(user.id)
  end                                       # COMMIT or ROLLBACK
end
```

What if the transaction is rolled back? In that case, the [Sidekiq] job will not be rolled back. It's a different database and there is no way to automagically do that. Even if you want to somehow roll back the [Sidekiq] job on your own, remember that your process may crash in the meantime.

If on top of the email, you were posting an event to another system, things can get even messier. For example, you may be charging the credit card of the owner of the account. If you do that (charge) and then roll back the transaction because of a database constraint, like uniqueness on the username, your users may start distrusting your business.

You might be thinking of introducing an "email sent" flag and handle not sent emails manually later via a [Rake] task.

It's a slightly better solution, but I'm not a huge fan of solutions that require manual intervention. You may need to fix that on Sunday evening or when you're on vacation, so for me, that's a source of headaches.

On top of everything, you're not solving the real problem which is to reliably send that email your product needs so much.

## The Solution You May Not Like

I hope that at this point it is clear that handling every possible error will lead you to an endless rabbit hole.

So how to reliability schedule that [Sidekiq] job? The idea is to split this operation into reliable phases.

Step one is to make the user creation and the "intention" to send the email an atomic operation. What is an email "intention"? It's just a record in the database that represents the email:

```ruby
def create_user
  transaction do
    user = User.create!(attributes)
    WelcomeEmail.create!(user: user)
  end
end
```

If you're using one of the popular relational databases out there, the database guarantees that the user and the welcome email are created atomically. Either both happen or none of them happen.

Step two is to actually send the email. That may be done in multiple ways, a simple one is to create a separated process that reads email intentions and sends them:

```ruby
while true
  email = WelcomeEmail.oldest_not_sent

  if email
    send_email(email.id)
  else
    sleep(wait_time)
  end
end
```

This is a simple worker that just finds emails not yet sent and sends them one by one. After sending, you may update the database row with the timestamp of the time of the send operation.

Alternatively, you may implement a [Cron] task that periodically sends a bunch of emails at once. The point is not the worker itself or how it is implemented, but the pattern used to solve this specific issue.

## Taking A Step Back

This is a trivial example and reality is much more complicated. So let's take a step back and look at the fundamentals discussed in this post. If you understand them well, chances are that you'll be able to apply the thought process to numerous situations instead of relying on technologic specific solutions.

The fundamentals are the following.

### Dual Writes

The example we've seen is a simplification of the dual-writes problem. Whenever you have to write to two (or more) systems, and both operations need to be atomic and consistent, we're entering the land of distributed systems. Even with such a simple example, we noticed that dual writes are hard to solve by simply handling all possible errors.

### Outbox pattern

We also have seen how to guarantee that the email is scheduled when the user is created by separating the creation of the email "intention" from the sending. The database guarantees that the user and email intention are written correctly via its ACID guarantees.

This is also a simplification of the Outbox Pattern, which is used in a much broader context.

### Domain Design

There is a hidden benefit of modeling the email intention this way. We're converting the email to a dedicated entity. That enables your business to track individual emails, knowing whether they "converted" or not, and even allowing multiple welcome emails to be sent.

### Simpler Architecture

[Redis] being unavailable is not a problem anymore. Well, actually I cheated. In the end, I removed [Redis] (and [Sidekiq]) from the initial flow.

By accident, we achieved a simpler design.

## Conclusion

I wanted to show in this post some concepts that I've been using and thinking on my daily work. Most importantly, I wanted to make you aware of dual writes and how to approach solving it.

There is no right or wrong way, just the most appropriate way for the circumstances you're facing. But understanding the fundamentals behind will help you adapt to future situations.

I recommend following those resources if you want to expand more on the concepts mentioned here:

* [https://thoughts-on-java.org/dual-writes/](https://thoughts-on-java.org/dual-writes/)
* [https://debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern/](https://debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern/)
* [https://en.wikipedia.org/wiki/ACID#Atomicity](https://en.wikipedia.org/wiki/ACID#Atomicity)

[SQS]: https://aws.amazon.com/sqs/
[RabbitMQ]: https://www.rabbitmq.com/
[Kafka]: https://kafka.apache.org/
[Sidekiq]: https://github.com/mperham/sidekiq
[Redis]: https://redis.io/
[Rake]: https://github.com/ruby/rake
[Cron]: https://en.wikipedia.org/wiki/Cron
