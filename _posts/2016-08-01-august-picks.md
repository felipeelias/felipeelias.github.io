---
layout: post
title:  "August/September Picks"
date:   2016-09-04 18:00:00 +0100
description: "Cool stuff from August & September 2016 - Fonts with ligatures, preprocessing CSV with ruby, crystal"
permalink: /2016-august-september-picks
tags: picks
comments: true
---

# [Tool] Fonts with ligatures

Link: [https://github.com/tonsky/FiraCode][firacode]

I didn't know that fonts could have ligatures, and this is pretty epic!

![fira-code-screenshot](/assets/fira-code.jpg)

# [Post] Preprocessing CSV data in Ruby

Link: [http://kgrz.io/ruby/2016/08/05/better-csv-preprocessing-in-ruby.html][csv]

Great tip of using `:converters` option of `CSV.parse`. It lets you convert columns to specific formats during parsing. An example:

```ruby
rows = CSV.parse(file, headers: true, converters: [CSV::Converters[:float]])
rows[1]
# => 74.0,E
rows[1][0].class
# => Float
```

# [Talk] Introduction to Crystal

Link: [http://confreaks.tv/videos/goruco2016-introducing-the-crystal-programming-language][crysal]

Great overview of Crystal language. It looks pretty similar to Ruby, with the advantage of being compiled!

# [Tips] OS X Command Line Awesomeness

Link: [https://github.com/herrbischoff/awesome-osx-command-line][osx]

> A curated list of shell commands and tools specific to OS X.

Found few tips there, like:

- Disabling iTunes media key 👏🏽
- Prevent Time Machine from Prompting to Use New Hard Drives as Backup Volume
- Show Hidden Files on Finder

Have a nice week!

[firacode]: https://github.com/tonsky/FiraCode
[crysal]: http://confreaks.tv/videos/goruco2016-introducing-the-crystal-programming-language
[csv]: http://kgrz.io/ruby/2016/08/05/better-csv-preprocessing-in-ruby.html
[osx]: https://github.com/herrbischoff/awesome-osx-command-line
