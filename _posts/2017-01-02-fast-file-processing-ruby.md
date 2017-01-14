---
layout: post
title:  "Optimal way of processing large files in Ruby"
date:   2017-01-02
description: "Fast processing of large files in Ruby with low memory consumption"
tags: ruby
comments: true
---

I was asked what was the optimal way to process files in [Ruby][ruby]. I had some assumptions, but they turn out to be wrong 😁, so I'm writing this post for future reference (and for anyone out there interested on it).

_Note: the "benchmarks" here are non-scientific, they are just a way to show in how many orders of magnitude the examples differ from each other_

# File.readlines/IO.readlines

This is by far the slowest. That's because this method scans the whole file, returning an array with every line in the file, which is very convenient and you might not even see problems in small files.

For my test case, I created a file with 24MB and loading it with [`readlines`][readlines] takes almost 2 seconds:

```
→ time ruby -e "File.readlines('large.txt')"

real  0m1.352s
user  0m1.044s
sys   0m0.210s
```

Also the memory consumption was quite high, on my machine it was reaching around 100MB!

# File.read/IO.read

Faster than [`#readlines`][readlines], however it returns a large string. This means that the whole file will still be loaded into memory, which is still not ideal.

```
→ time ruby -e "File.read('large.txt')"

real  0m0.392s
user  0m0.212s
sys   0m0.098s
```

Memory consumption was around 31MB on my machine. [Ruby][ruby] runtime itself has 7MB, plus 24MB of loaded strings, matches the actual memory.

# File#each/IO.foreach

This is where things get more interesting. [`File#each`][each] receives a block, passing each line as the argument of the block. This so far is the best method to process a file sequentially, because the lines are not all loaded into memory at the same time.

```
→ time ruby -e "File.open('large.txt','r').each { |line| line }"

real  0m1.410s
user  0m1.231s
sys   0m0.089s
```

The total time is pretty similar to [`#readlines`][readlines], though, looking at memory consumption at the end of the script, it was nearly the same as before loading the file, around 8MB.

Note that I'm passing a dummy block `{ |line| line }` on [`#each`][each]. That's because calling [`#each`][each] without a block returns an enumerator. Which is a good thing!

Imagine that you want to find the first 10 lines that contains the string `abcd`. You could do that with:

```ruby
IO.foreach('large.txt').grep(/abcd/).take(10)
```

That takes around 3 seconds on my machine. It could be better though, if we take advantage of [`Enumerable#lazy`][lazy]:

```ruby
IO.foreach('large.txt').lazy.grep(/abcd/).take(10).to_a
```

That takes around 1 second, because [`lazy`][lazy] makes methods like `grep`, `find`, `reject` to be evaluated only when they're needed. Pretty powerful!

# Could it be faster?

I found out that you can "advise" the system on the type of read you're going to perform with [IO#advise][advise]. For example, if you're doing a sequential read, you can call `file.advise(:sequential)`, however I didn't see that many improvements on my tests.

Do you know a better way? Let me know in the comments!

[advise]: https://ruby-doc.org/core-2.1.0/IO.html#method-i-advise
[ruby]: https://www.ruby-lang.org
[lazy]: http://ruby-doc.org/core-2.4.0/Enumerable.html#method-i-lazy
[readlines]: https://ruby-doc.org/core-2.1.0/IO.html#method-i-readlines
[each]: https://ruby-doc.org/core-2.1.0/IO.html#method-i-each
