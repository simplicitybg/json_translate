[![Gem Version](https://badge.fury.io/rb/json_translate.svg)](https://badge.fury.io/rb/json_translate)
[![Build Status](https://api.travis-ci.org/cfabianski/json_translate.png)](https://travis-ci.org/cfabianski/json_translate)
[![License](http://img.shields.io/badge/license-mit-brightgreen.svg)](COPYRIGHT)
[![Code Climate](https://codeclimate.com/github/cfabianski/json_translate.png)](https://codeclimate.com/github/cfabianski/json_translate)

# JSON Translate

Rails I18n library for ActiveRecord model/data translation using PostgreSQL's
JSONB datatype or MySQL's JSON datatype. It provides an interface inspired by
[Globalize3](https://github.com/svenfuchs/globalize3) but removes the need to
maintain separate translation tables.

## Requirements

* ActiveRecord >= 4.2.0
* I18n
* MySQL support requires ActiveRecord >= 5 and MySQL >= 5.7.8.

## Installation

gem install json_translate

When using bundler, put it in your Gemfile:

```ruby
source 'https://rubygems.org'

gem 'activerecord'

# PostgreSQL
gem 'pg', :platform => :ruby
gem 'activerecord-jdbcpostgresql-adapter', :platform => :jruby

# or MySQL
gem 'mysql2', :platform => :ruby
gem 'activerecord-jdbcmysql-adapter', :platform => :jruby

gem 'json_translate'
```

## Model translations

Model translations allow you to translate your models' attribute values. E.g.

```ruby
class Post < ActiveRecord::Base
  translates :title, :body
end
```

Allows you to translate the attributes :title and :body per locale:

```ruby
I18n.locale = :en
post.title # => This database rocks!

I18n.locale = :he
post.title # => אתר זה טוב
```

You also have locale-specific convenience methods from [easy_globalize3_accessors](https://github.com/paneq/easy_globalize3_accessors):

```ruby
I18n.locale = :en
post.title # => This database rocks!
post.title_he # => אתר זה טוב
```

To find records using translations without constructing JSON queries by hand:

```ruby
Post.with_title_translation("This database rocks!") # => #<ActiveRecord::Relation ...>
Post.with_title_translation("אתר זה טוב", :he) # => #<ActiveRecord::Relation ...>
```

In order to make this work, you'll need to define an JSON or JSONB column for each of
your translated attributes, using the suffix "_translations":

```ruby
class CreatePosts < ActiveRecord::Migration
  def up
    create_table :posts do |t|
      t.column :title_translations, 'jsonb' # or 'json' for MySQL
      t.column :body_translations,  'jsonb'
      t.timestamps
    end
  end
  def down
    drop_table :posts
  end
end
```

## I18n fallbacks for missing translations

It is possible to enable fallbacks for missing translations. It will depend
on the configuration setting you have set for I18n translations in your Rails
config.

You can enable them by adding the next line to `config/application.rb` (or
only `config/environments/production.rb` if you only want them in production)

```ruby
config.i18n.fallbacks = true
```

Sven Fuchs wrote a [detailed explanation of the fallback
mechanism](https://github.com/svenfuchs/i18n/wiki/Fallbacks).

## Temporarily disable fallbacks

If you've enabled fallbacks for missing translations, you probably want to disable
them in the admin interface to display which translations the user still has to
fill in.

From:

```ruby
I18n.locale = :en
post.title # => This database rocks!
post.title_nl # => This database rocks!
```

To:

```ruby
I18n.locale = :en
post.title # => This database rocks!
post.disable_fallback
post.title_nl # => nil
```

You can also call your code into a block that temporarily disable or enable fallbacks.

```ruby
I18n.locale = :en
post.title_nl # => This database rocks!

post.disable_fallback do
  post.title_nl # => nil
end

post.disable_fallback
post.enable_fallback do
  post.title_nl # => This database rocks!
end
```

## Enable blank value translations

By default, empty String values are not stored, instead the locale is deleted.

```ruby
class Post < ActiveRecord::Base
  translates :title
end

post.title_translations # => { en: 'Hello', fr: 'Bonjour' }
post.title_en = ""
post.title_translations # => { fr: 'Bonjour' }
```

Activating `allow_blank: true` enables to store empty String values.
```ruby
class Post < ActiveRecord::Base
  translates :title, allow_blank: true
end

post.title_translations # => { en: 'Hello', fr: 'Bonjour' }
post.title_en = ""
post.title_translations # => { en: '', fr: 'Bonjour' }
```

`nil` value delete the locale key/value anyway.
```ruby
post.title_en = nil
post.title_translations # => { fr: 'Bonjour' }
```

## Manually setting the available locales for translation for each model

By default the value of `I18n.available_locales` is used to create the helper methods, e.g.:

```ruby
I18n.available_locales = [:en, :de]

class Post < ActiveRecord::Base
  translates :title
end

post.title_en = 'Hello'
post.title_fr = 'Bonjour' # NoMethodError: undefined method `title_fr=' for #<Post:0x0000000000000000>
```

Sometimes your app might need to offer translation of content in locales that differ from the UI of your app. In such cases you can explicitly set the available locale accessors for each model you have by using the `locale_accessors` and passing either an array of symbols (if the same locales are to be available for each instance of the model) or a proc, when you want to have different locale accessors for each instance of the same model.

Setting the locale via array:

```ruby
class Post < ActiveRecord::Base
  translates :title, locale_accessors: [:pt, :fr]
end
```

This allows you to define completely decoupled locales from your UI locales:

```ruby
I18n.available_locales = [:en, :de]

class Blog < ActiveRecord::Base
  has_many :posts
end

class Post < ActiveRecord::Base
  belongs_to :blog
  translates :title, locale_accessors: [:pt, :fr]
end

blog = Blog.new()
post = blog.posts.build

post.title_fr = 'Bonjour'
post.title_en = 'Hallo' # NoMethodError: undefined method `title_en=' for #<Post:0x0000000000000000>
```


Setting the locale via a proc:

```ruby
class Post < ActiveRecord::Base
  translates :title, locale_accessors: -> (post) { post.website.available_locales }
end
```

This allows you to define per-instance locale helpers

```ruby
I18n.available_locales = [:en, :de]

class Blog < ActiveRecord::Base
  has_many :posts
end

class Post < ActiveRecord::Base
  belongs_to :blog
  translates :title, locale_accessors: -> (post) { post.website.available_locales }
end

blog1 = Blog.new(available_locales: [:it, :es])
post1 = blog1.posts.build

post1.title_it = 'Ciao'
post1.title_en = 'Hallo' # NoMethodError: undefined method `title_en=' for #<Post:0x0000000000000000>

blog2 = Blog.new(available_locales: [:fr, :pt])
post2 = blog2.posts.build

post2.title_fr = 'Bonjour'
post2.title_it = 'Ciao' # NoMethodError: undefined method `title_it=' for #<Post:0x0000000000000000>
```
