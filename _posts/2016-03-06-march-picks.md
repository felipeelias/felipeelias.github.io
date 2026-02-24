---
layout: post
title:  "March Picks"
date:   2016-03-06 15:00:00 +0100
description: "Interesting content from March 2016"
permalink: /march-picks
tags: picks
comments: true
archived: true
---

# [Simple Made Easy][talk]

> Rich Hickey emphasizes simplicity's virtues over easiness’, showing that while many choose easiness they may end up with complexity, and the better way is to choose easiness along the simplicity path.

In other words, simplicity is measurable: simple things have one task, or one goal. Easy, on the other hand is relative to the person's interpretation; it might be easy for you to play the piano, but not for me.

On the same topic, ["The Pursuit of Simplicity in Programming"][post] extends his talk, mentioning accidental vs essential complexity.

How many times during your work day, you choose the simplest approach instead of the easy one?

# [Blog][blog]

The most minimalist [blog][blog] I've seen 😀 Lots of great content written by @danluu, some of them that I read and recommend:

- [Files are hard](http://danluu.com/file-consistency/)
- [Everything is broken](http://danluu.com/everything-is-broken/)
- [Automated bug detection with analytics](http://danluu.com/bugalytics/)

# \#reductions

Inspired by [Clojure's][clojure] [`reductions`][reductions] function, returns all intermediate values of a reduction, or in [Ruby][ruby], [`Enumerable#inject`][inject]:

```ruby
def reductions(array, initial, &block)
  Array.new.tap do |final|
    array.inject(initial) do |memo, val|
      final.push(memo = yield(memo, val))
      memo
    end
  end
end

reductions((1..5), 0) { |memo, val| memo + val }
# => [1, 3, 6, 10, 15]
reductions((1..5), 0, &:+)
# => [1, 3, 6, 10, 15]
```

# Sensible Bash Defaults

[https://github.com/mrzool/bash-sensible](https://github.com/mrzool/bash-sensible)

> An attempt at saner Bash defaults.

There are so many good tips there, like, better auto-completion, better history handling, navigation and so on. Also recommended to take a look at this [blog post][bash-post].

# \#noEstimates debundked

It's actually an old post that I bumped into recently. ["#noEstimates debundked"][no-estimates] by @akitaonrails talks about how this idea is absurd, and make pretty interesting points.

> The problem is not the estimation, it's the execution.

[talk]: http://www.infoq.com/presentations/Simple-Made-Easy
[post]: http://blog.mediumequalsmessage.com/simplicity-in-programming
[blog]: http://danluu.com/
[reductions]: https://clojuredocs.org/clojure.core/reductions
[clojure]: https://clojure.org/
[ruby]: https://www.ruby-lang.org/en/
[inject]: http://ruby-doc.org/core-2.3.0/Enumerable.html#method-i-inject
[bash-post]: http://mrzool.cc/writing/sensible-bash/
[no-estimates]: http://www.akitaonrails.com/2013/10/07/off-topic-noestimates-debunked#.VK1ZMorF_SY
