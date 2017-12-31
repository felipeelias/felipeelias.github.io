---
layout: post
title:  "Denormalized indexing with elasticsearch-rails"
date:   2017-12-28
description: "How to index multiple models in Elasticsearch using elasticsearch-rails gem"
tags: ruby
categories: ruby
comments: true
---

One of the biggest advantages of using [Elasticsearch][es] it is because it's fast even if you have very complex documents with many attributes coming from different models.

Achieving that requires you to [denormalize][denormalization] those models into a single `index` and it's your responsibility as a developer to keep it consistent.

That reveals some interesting challenges, let's take a look.

## Blog in 5 minutes

Imagine we have a simple has_many/belongs_to relationship between `Posts` and `Authors`. Our end goal is to be able to search by _posts from a specific author by its name_.

Assuming that you have a basic Rails application with [elasticsearch-rails][elasticsearch-rails] installed and [Elasticsearch][es] running, our models will look like this:

```ruby
# db/migrate/...
class CreateModels < ActiveRecord::Migration[5.1]
  def change
    create_table :authors do |t|
      t.string :name
      t.string :email
      t.timestamps
    end

    create_table :posts do |t|
      t.string :title
      t.text :body
      t.datetime :published_at
      t.references :author, foreign_key: true
      t.timestamps
    end
  end
end

# models/author.rb
class Author < ApplicationRecord
  has_many :posts
end

# models/post.rb
class Post < ApplicationRecord
  include Elasticsearch::Model
  include Elasticsearch::Model::Callbacks

  belongs_to :author
end
```

The `Post` model will be our entry point to manage changes in the index. I'm not going to get into details of how the [elasticsearch-rails][elasticsearch-rails] gem works, you can check its documentation on the [github repository][elasticsearch-rails].

Assuming you imported all posts and users, you can perform full-text search with:

```ruby
Post.search('example').records.all
```

That will let you search for every attribute in the `Post` model, but not by `author` names.

## Extending

Now for the fun part. Let's add `author` information in the same index as the `posts`. This will help us achieve our goal of searching by `author` name.

You'll need to define a custom mapping and a how to index it by overriding `#as_indexed_json` method:

```ruby
class Post < ApplicationRecord
  # ... snipped

  mapping dynamic: :strict do
    indexes :id,           type: :long
    indexes :title,        type: :text
    indexes :body,         type: :text
    indexes :published_at, type: :date
    indexes :created_at,   type: :date
    indexes :updated_at,   type: :date
    indexes :author do
      indexes :id,   type: :long
      indexes :name, type: :text
    end
  end

  def as_indexed_json(options = {})
    self.as_json(
      options.merge(
        only: [:id, :title, :body, :published_at, :created_at, :updated_at],
        include: { author: { only: [:id, :name] } }
      )
    )
  end
end
```

After changing this, you must **recreate** the index and **re-import** the data:

```ruby
Post.__elasticsearch__.create_index!(force: true)
Post.import
```

## Then, there is one bug 🐞

If you use your application for a while, you'll notice that if you change the `author` of a post, this change won't be reflected in Elasticsearch.

After some debugging, it turns out that [elasticsearch-rails][elasticsearch-rails] gem only indexes the attributes that changed via [ActiveModel::Dirty][dirty] module. That doesn't work for our case since `author` is not an attribute of a `post`.

Simply put, when you modify the `author` of a `post`, the attribute that changes is the `author_id`. After you save the `post`, the gem compares which attributes changed against the hash returned by `#as_indexed_json`. Since our changes are now represented as an `author` hash, the gem can't find the `author_id` there.

There are a couple of ways to solve this:

1. Drop the `Elasticsearch::Model::Callbacks` module and then handle the indexing logic yourself
2. Force a change by adding the `author` key as a change whenever the `author_id` changes
3. Ignoring all changes completely

I choose to go with solution #2, which looks like this:

```ruby
class Post < ApplicationRecord
  # ... snipped

  before_save :force_index

  def force_index
    if changes['author_id']
      attr = :@__changed_model_attributes
      old_changes = __elasticsearch__.instance_variable_get(attr)
      __elasticsearch__.instance_variable_set(attr, old_changes.merge!('author' => true))
    end
  end
end
```

It's a hack, it changes the internals of the [elasticsearch-rails][elasticsearch-rails] gem and I'm not very happy with the solution. I went this way to keep the functionality of indexing only changed attributes however, this can get pretty cumbersome to maintain.

If you don't care about this optimization, you can go with #3 and **always** force the index by clearing the `@__changed_model_attributes` instance variable:

```ruby
def force_index
  __elasticsearch__.instance_variable_set(:@__changed_model_attributes, nil)
end
```

With either approach, if you change the `author` the changes will be reflected in Elasticsearch.

## And then, there are two bugs 🐞🐞

After the hint on the previous bug, one would notice that changing the author's name won't reflect on [Elasticsearch][es] either! That's because `Author` model doesn't know anything about indexing itself the `Post` index.

This is where keeping the consistency on the index gets tricky. There are numerous ways of solving this, each of them with its own drawbacks. To keep things simple I'm going to suggest one solution that works well and doesn't use any other dependency other than [Elasticsearch][es] itself 🎉🎉🎉.

We'll use `#update_by_query` feature which as the name suggests, lets you update various documents that match a query. It has some cool features like being able to work _asynchronously_, updating documents at its own pace without _overloading the cluster_ and handling _conflicts_. Check out the [documentation here][update_by_query].

Let's take advantage of that to update all posts that belong to a specific `author` in the background:

```ruby
# models/author.rb
class Author < ApplicationRecord
  after_commit :update_relations

  private

  def update_relations
    Post.update_authors(self)
  end
end

# models/post.rb
class Post < ApplicationRecord
  # ... snipped

  def self.update_authors(author, options = {})
    options[:index] ||= index_name
    options[:type]  ||= document_type
    options[:wait_for_completion] ||= false

    options[:body] = {
      conflicts: :proceed,
      query: {
        match: {
          'author.id': author.id
        }
      },
      script: {
        lang:   :painless,
        source: "ctx._source.author.name = params.author.name",
        params: { author: { name: author.name } }
      },
    }

    __elasticsearch__.client.update_by_query(options)
  end
end
```

The code is quite self-explanatory. Any changes in the `Author` model will trigger an `#update_by_query` which performs an update for all posts that match the query:

```ruby
query: {
  match: {
    'author.id': author.id
  }
}
```

For each match, it will execute the _scripted update_ defined, which simply sets the `author` name to the one specified in the `params`:

```ruby
script: {
  lang:   :painless,
  source: "ctx._source.author.name = params.author.name",
  params: { author: { name: author.name } }
}
```

You may want to optimize the `#update_relations` method to only call `#update_authors` when necessary. Using `params` let you easily include more attributes in the future and also avoids potential security issues brought by concatenating strings in the `source`.

Setting `wait_for_completion` to `false` will tell [Elasticsearch][es] to perform the update asynchronously. This is good if there is a potential case of an `author` having tons of posts.

## Thinking about conflicts

You may have noticed that I set `conflicts: :proceed` in the updated body. This is to handle a couple of scenarios:

### The post is updated

Imagine the case where we update an author's name that has one bazillion posts. That will take a while... There is a chance that any of the author's posts will be updated by somebody else in the meantime.

Before running an `#update_by_query`, [Elasticsearch][es] takes a snapshot of the index and uses the internal versioning scheme to identify such conflicts. If a `post` is updated after the time when update was "queued" and before it was "run", the `post` will have a new version, so the `#update_by_query` will fail for that `post`. In this scenario, we'd like to **skip** such conflicts and proceed.

This means that the _last update wins_ and we have the guarantee that the post will have the latest value for the author's name.

### Multiple updates to the same author

If somebody updates the `author` once, then immediately regrets this decision and updates it again to something else, there is a chance that the first update will still be running (considering bazillion of posts). If that's the case, the first update will encounter conflicts, ignore them and move on.

In theory, the second update will always win because it will come after the first one.

## Conclusion

[Denormalizing][denormalization] data can help you take advantage of [Elasticsearch][es] fast querying features, but it has a cost of having to handle concurrent updates to multiple models, which reveals some pretty hard to debug issues and inconsistency.

Note that is a very simple scenario and probably you won't need [Elasticsearch][es] if you don't have anything other than that. However, the biggest advantage comes when you have to index many different models in the same document and when doing joins in the database becomes prohibitive.

[elasticsearch-rails]: https://github.com/elastic/elasticsearch-rails
[dirty]: http://api.rubyonrails.org/classes/ActiveModel/Dirty.html
[update_by_query]: https://www.elastic.co/guide/en/elasticsearch/reference/5.6/docs-update-by-query.htm
[denormalization]: https://en.wikipedia.org/wiki/Denormalization
[es]: https://www.elastic.co/products/elasticsearch
