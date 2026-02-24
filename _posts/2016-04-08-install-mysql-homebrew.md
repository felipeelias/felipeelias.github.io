---
layout: post
title:  "mysql2 gem installation hell"
date:   2016-04-08 10:00:00 +0100
description: "Install older version of MySQL (5.5) and mysql2 gem with homebrew"
tags: [mysql, ruby]
categories: mysql
comments: true
archived: true
---

The current version of MySQL on homebrew is `5.7`. If you need to install a previous version of the
server, in my case it was `5.5`, do:

```sh
brew tap homebrew/versions
brew update
brew install mysql55
```

This should be enough for having the server installed correctly, but it's not enough for installing
the ruby gems:

```sh
gem install mysql2 -v '0.3.11'
Building native extensions
This could take a while...
ERROR:  Error installing mysql2:
  ERROR: Failed to build gem native extension.
...
-----
mysql.h is missing.  please check your installation of mysql and try again.
```

To do that, you'll need to first remove existing gems

```sh
gem uninstall mysql2
```

Then, link the current MySQL server with homebrew:

```
brew link mysql55
gem install mysql2
```

This worked for me 😅
