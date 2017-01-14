---
layout: post
title:  "June Picks"
date:   2016-06-10 18:00:00 +0100
description: "Interesting content from June 2016, Rails has won, Laws of software development - June Picks"
permalink: /2016-june-picks
tags: picks
comments: true
---

# [Article] Rails has won

Link: [http://www.akitaonrails.com/2016/05/23/rails-has-won-the-elephant-in-the-room][rails-has-won]

This is a response to [My time with Rails is up][flaws]. I recommend reading both articles. While Rails is bloated and there are technically superior solutions out there, we have to accept the reality:

> People think that because something is "technically superior" everybody else should blindly adopt. But this is not how the market works.

# [Article] How To Sound Smart At Your Next Team Meeting

Link: [https://www.exceptionnotfound.net/fundamental-laws-of-software-development/][laws]

> Never attribute to malice what can be adequately explained by stupidity.
> 80% of the effects stem from 20% of the causes.
> Any code of your own that you haven't looked at for six or more months might as well have been written by someone else.

# [Article] Do You Take Yourself Seriously?

Link: [https://medium.com/@sarahcpr/do-you-take-yourself-seriously-704418a5f614#.p9md1f671][srsly]

This is a periodic reminder for myself.

# [Code] `ab` telling you wrong things

I was running some benchmarks and I noticed that `ab` was reporting few "Failed Requests" on every benchmark. It turns out this is **not** related to HTTP status codes, but it is because the tool compares the length of subsequent responses with the first one. If they differ, then it considers a failed request. For dynamic contents, they will be *always* different!

Using the `-l` option "fixes" that:

> -l: Do not report errors if the length of the responses is not constant. This can be useful for dynamic pages. Available in 2.4.7 and later.

Example:

```
ab -c 10 -n 100 -l -r -T 'application/json' -p data.json 'http://example.com'
```

# [Article] Am I really a developer or just a good googler?

Link: [http://www.stilldrinking.org/how-to-worry-less-about-being-a-bad-programmer][worry-less]

> [...] programming is now a job like any other, because everything you need to do to satisfy your BizDev team can be learned without reverse-engineering the prototype for Thag's Move Things Better Octagon.

Thanks for reading! 😬

[worry-less]: http://www.stilldrinking.org/how-to-worry-less-about-being-a-bad-programmer
[laws]: https://www.exceptionnotfound.net/fundamental-laws-of-software-development/
[srsly]: https://medium.com/@sarahcpr/do-you-take-yourself-seriously-704418a5f614#.p9md1f671
[rails-has-won]: http://www.akitaonrails.com/2016/05/23/rails-has-won-the-elephant-in-the-room
[flaws]: http://solnic.eu/2016/05/22/my-time-with-rails-is-up.html
