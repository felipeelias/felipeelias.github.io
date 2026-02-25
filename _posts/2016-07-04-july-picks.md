---
layout: post
title:  "July Picks"
date:   2016-07-04 18:00:00 +0100
description: "Cool stuff from July 2016, Docopt, Git highlight, Reverse engineering a UDP stream, How to keep your best programmers - July Picks"
permalink: /2016-july-picks
tags: picks
comments: true
archived: true
---

# [Tool] Zero code option parser

Link: [http://docopt.org/][docopt]

[docopt][docopt] is an option parser that is based on conventions that have been used for decades in help messages and man pages for describing a program's interface. An interface description in docopt is such a help message, but formalized.

# [Tool] Git 2.9 diff improvements

Git 2.9 [has been released][release] and with it, few nice additions, my favourites goes to diff improvements:

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Using <a href="https://twitter.com/hashtag/git?src=hash">#git</a> 2.9+? You should immediately enable `--compaction-heuristic` for improved diffs. Comparison below. <a href="https://t.co/vXmDmYg5l3">pic.twitter.com/vXmDmYg5l3</a></p>&mdash; John Feminella ✈ BDL (@jxxf) <a href="https://twitter.com/jxxf/status/742799938501890048">June 14, 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

Another addition was a script that puts an extra emphasis on the changed part of line:

![diff-highlight-screenshot](/assets/diff-highlight-git-2-9.webp)

In order to install that, make sure that `diff-highlight` is on your path, since the script is under Git's contrib files. After that, add this to your `~/.gitconfig`

```
[pager]
  log = diff-highlight | less
  show = diff-highlight | less
  diff = diff-highlight | less
```

# [Post] Hotel Music

Link: [http://wiki.gkbrk.com/Hotel_Music.html][hotel]

Great post by Gökberk on how he reversed engineering a mysterious UDP stream in the hotel where he was.

# [Post] How to keep your best programmers

Link: [http://www.daedtech.com/how-to-keep-your-best-programmers][best]

Long post, but worth reading. While it does not explicitly tells how to keep your best programmers, it touches good points. For example:

> [...] programmers are only happy in jobs that provide value to them and jobs to which they provide increasing value. The best and brightest not only want to grow but also to feel that they are increasingly useful and valuable–indicative, I believe, of pride in one’s work.

Have a nice week!

[docopt]: http://docopt.org/
[release]: https://github.com/blog/2188-git-2-9-has-been-released
[hotel]: http://wiki.gkbrk.com/Hotel_Music.html
[best]: http://www.daedtech.com/how-to-keep-your-best-programmers
