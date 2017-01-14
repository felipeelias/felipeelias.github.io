---
layout: page
title: Projects
permalink: /projects/
description: "Open source and experimental projects"
---

Here is a list of things that I've been working on my free time.

# Doorkeeper

<a class="github-button" href="https://github.com/doorkeeper-gem/doorkeeper" data-icon="octicon-star" data-style="mega" data-count-href="/doorkeeper-gem/doorkeeper/stargazers" data-count-api="/repos/doorkeeper-gem/doorkeeper#stargazers_count" data-count-aria-label="# stargazers on GitHub" aria-label="Star doorkeeper-gem/doorkeeper on GitHub">Star</a>

> Doorkeeper is an oAuth 2 provider for Rails

I started working on doorkeeper as an internal Rails app in [Applicake][applicake] along with my friend @piotrjakubowski. We then decided that it would be a good idea to release it to the public so people could re-use it.

Since oAuth2 server is not straightforward to understand at first, I wrote extensive [documentation on the project's wiki][wiki] and wrote few [example applications][example-apps] that are easily deployable to Heroku. You can check the running server [here][server].

The project also got featured in [Ruby5][ruby5] and [Railscasts][railscasts].

# Resubject

<a class="github-button" href="https://github.com/felipeelias/resubject" data-icon="octicon-star" data-style="mega" data-count-href="/felipeelias/resubject/stargazers" data-count-api="/repos/felipeelias/resubject#stargazers_count" data-count-aria-label="# stargazers on GitHub" aria-label="Star felipeelias/resubject on GitHub">Star</a>

Resubject implements the decorator pattern and makes easy to decorate Ruby objects. It also has build-in Rails and RSpec integration.

I built it using Ruby's `SimpleDelegator` due to its simplicity to use. It works like this:

```ruby
Box = Struct.new(:name, :items)

class BoxPresenter < Resubject::Presenter
  def contents
    items.join(', ')
  end
end

box = Box.new('Awkward Package', ['platypus', 'sloth', 'anteater'])
presentable = BoxPresenter.new(box)
presentable.contents
# => platypus, sloth, anteater
```

I really like well documented projects 😎 - check out the [RDoc documentation][resubject-rdoc] and the [Readme][resubject-readme].

# Instrumentation

<a class="github-button" href="https://github.com/felipeelias/instrumentation" data-icon="octicon-star" data-style="mega" data-count-href="/felipeelias/instrumentation/stargazers" data-count-api="/repos/felipeelias/instrumentation#stargazers_count" data-count-aria-label="# stargazers on GitHub" aria-label="Star felipeelias/instrumentation on GitHub">Star</a>

This project is a simple Ruby gem that helps you see memory usage for any process in a visual way. It's easy to run, with:

```
instrument <pid>
```

Go to `http://localhost:8080` and you'll see:

![instrumentation](/assets/instrumentation.gif)

# Reader to Plus - Chrome Extension

> Share your Google reader's items on Google+

Back in the day I used Google+ and Google Reader a lot, and I wanted to automate the sharing from Google Reader's side. Here's a screenshot:

![reader-to-plus](/assets/reader-to-plus.png)

The project is still available on [Chrome Web Store][store-link] and it used to have over 2k users, but since discontinuation of Google Reader, it became deprecated.

# Game.js

Experimental Javascript framework for building games. Originally, I wanted to build the Pong game and understand the basics of game development, so building a framework also helped me understand that.

You can check the project page [here][game-js-page] and the repository [here][game-js-repo]

![game-js-screenshot](/assets/game-js-about.jpg)

# Spoty-hour

Weekend project that I built to enhance Spotify playlists and show the time that the songs will play.

[https://github.com/felipeelias/spoty-hour](https://github.com/felipeelias/spoty-hour)

[applicake]: https://web.facebook.com/applicake
[example-apps]: https://github.com/doorkeeper-gem/doorkeeper/wiki/Example-Applications
[wiki]: https://github.com/doorkeeper-gem/doorkeeper/wiki
[ruby5]: https://ruby5.codeschool.com/episodes/252-episode-248-february-21-2012/stories/2215-create-an-oauth2-provider-with-doorkeeper
[railscasts]: http://railscasts.com/episodes/353-oauth-with-doorkeeper
[server]: https://doorkeeper-provider.herokuapp.com/
[game-js-page]: http://felipeelias.github.io/game.js/
[game-js-repo]: https://github.com/felipeelias/game.js
[resubject-rdoc]: http://www.rubydoc.info/github/felipeelias/resubject/master/Resubject
[resubject-readme]: https://github.com/felipeelias/resubject/blob/master/README.md
[store-link]: https://chrome.google.com/webstore/detail/reader-to-plus/ellpglpgjfcfppiljfokjoconaheaiff
