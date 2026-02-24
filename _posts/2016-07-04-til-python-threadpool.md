---
layout: post
title:  "TIL: Python Threadpool"
date:   2016-07-04 21:00:00 +0100
description: "TIL how to use a threadpool in Python"
permalink: /til-python-threadpool
tags: [til, python]
comments: true
archived: true
---

I'm not very proficient in [Python][python] yet, but while playing with it, I felt the need to do a bunch of API calls concurrently.

In Ruby, you can achieve that with gems like [concurrent-ruby][concurrent-ruby], but turns out [Python][python] has an implementation of a `threadpool` in its standard library, and you can use it like this:

```python
from multiprocessing.pool import ThreadPool

# Work that will be done concurrently
def work(index):
    # some long process
    return index

pool = ThreadPool(processes=5)
results = []

for index in range(0, 10):
    # Post `work` in the pool
    result = pool.apply_async(work, (index,))
    results.append(result)

for result in results:
    # wait for each result
    result.get()
```

[concurrent-ruby]: https://github.com/ruby-concurrency/concurrent-ruby
[python]: https://www.python.org/
