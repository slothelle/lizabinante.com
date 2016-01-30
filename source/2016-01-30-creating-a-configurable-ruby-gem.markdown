---
title: "Creating a configurable Ruby gem"
category: blog
tags: blog, technical, ruby, gems
---

I love Ruby. I've also, unrelated to my love of Ruby, had a paralyzing fear of sharing my Ruby with anyone else. Probably because I'm so emotionally attached to it? Anyways. I got over it and started publishing my [first Ruby gem](https://github.com/ArtCraftCode/ravelry): a Ruby wrapper for the [Ravelry API](http://www.ravelry.com/api). Here's the basics of how I set up and published my first gem, as well as a step-by-step guide for making it configurable.

READMORE

If you don't want to read through the whole post and just want to get the code, a gist is available [here](https://gist.github.com/feministy/602bd01a0ce3f7b141ec).

## Bundler + RubyGems = Happy gems!

I opted to use Bundler to manage everything about my `*.gemspec` file and gem business, as well as handle local development. There's a lovely guide [here](http://bundler.io/v1.10/rubygems.html) on how everything works, but I'll go over the basics.

In the examples below, I'm using `ravelry` as my gem name. You'll replace this with your gem name.

**Bundler: Create your gem**

This command will create a new directory with a Git repository, as well as the required `*.gemspec` file.

*Before* you get super attached to your gem's name, make sure the name isn't taken by searching [RubyGems.org](https://rubygems.org).

```shell
bundle gem ravelry
```

**Bundler: Build your gem**

When you're ready to build the gem and publish it, or if you want to test a specific version of a gem in an isolated environment, **update the date and version** then run this command:

```shell
gem build ravelry.gemspec
```

This will create a file/folder/thing called `ravelry-0.0.0.gem`. The version `0.0.0` will be replaced with whatever you have in your `gemspec`'s "version" setting.

**RubyGems: Publishing the gem**

First, you need to sign up for a [RubyGems](https://rubygems.org/) account. Make sure you follow the additional instructions for setting up your credentials locally or you won't get very far.

Then, when you're ready, run this command:

```shell
gem publish ravelry-0.0.0.gem
```

This will published your gem to the RubyGems.org server, and give you a nice little page, like [this](https://rubygems.org/gems/ravelry)!

Now that we've got those basics covered, let's talk about making your gem configurable.

## Setting API keys (and more): two options

For my Ravelry gem, I needed to have a way for users to set API keys. There are two approaches to this that I considered:

1. Having the user set `ENV` variables for the required keys
2. Setting up a configuration block that you pass required keys to (sometimes these are `ENV` variables)

The former was easier, so I went with that option initially. However, as the gem envolved and I started to use it in a Rails application, I decided to clean it up and go with option two.

Using a configuration block will also allow me to add optional settings to the gem in the future as I add more API endpoints. Maybe I wanted to give users an option to receive raw JSON instead of Ruby objects? Or maybe provide different methods for pagination? Regardless of *what* options I may invent later, creating a way configure settings for a gem is the easiest way to go.

## Gem file structure

Pretty much every gem I've used utilizes a module syntax, and the Ravelry gem is no different.

My gem starting out looks like this, most of which was provided by Bundler:

```plain
/lib
  ravelry.rb
  /ravelry
    *.rb

/spec
  ravelry_spec.rb
  spec_helper.rb
  /helpers
  /ravelry
    *_spec.rb

.gitignore
.rspec
.ruby-version
.yardopts
CODE_OF_CONDUCT.md
Gemfile
Gemfile.lock
LICENSE.txt
ravelry.gemspec
README.md
VERSION
```

**What does it all do?**

The `spec` folder mirrors the `lib` folder, and will continue to do so as the structure gets more complex. The `lib` folder contains everything my Gem needs to succeed, and will be the code that runs in a user's environment when they're using the Gem.

The `lib/ravelry.rb` file is used to open the `Ravelry` module, and to require all of the necessary files (more on that later).

Most of the `dotfiles` are for settings I want to be maintained across the project, or for Gem settings (like Yard, for documentation):

```plain
.gitignore
.rspec
.ruby-version
.yardopts
```

Others are required to build the Gem and work with other Ruby gems:

```plain
Gemfile
Gemfile.lock
ravelry.gemspec
```

Some can be handled inline in my `ravelry.gemspec`, but I prefer to have them as separate files:

```plain
CODE_OF_CONDUCT.md
LICENSE.txt
README.md
```

And I forgot to delete my `VERSION` file. lol whoops.

## Configuration block: the end goal

We want the users of our gem to be able to do something like this:

```ruby
Ravelry.configure do |config|
  config.access_key = ''
  config.secret_key = ''
  config.personal_key = ''
end
```

We want:

1. Our gem users to only have to write this configuration block once per application
2. Our configuration settings to be available throughout the gem (and the application using the gem)

So how do we do this?

## Creating a configuration block

I started with my `lib/ravelry.rb` file, where I require all of my files and create the Ravelry module. It looks like this:

**File**: `lib/ravelry.rb`

```ruby
# blah blah
# require stuff...

module Ravelry; end
```

This is not super helpful. In fact, all it does is create the `Ravelry` module. So let's fix it up!

We need to do a few things when we create a configuration block:

1. Create a `Configuration` class inside of our `Ravelry` module
2. Determine what things we want to be configured and add them to our class with the appropriate `attr_accessor`s
3. On our `Ravelry` module, create an `attr_accessor` for the instance of the `Configuration` class (instance variables on a module - so cool!)
4. Create a method to return the configuration settings
5. Write some tests!
6. Write some documentation!

### Steps 1-2: A `Configuration` class with accessible settings

I need to have three things accessible to users of my gem:

1. Ravelry access key
2. Ravelry secret key
3. Ravelry personal key

These are things I can't provide, but the user can get from their Ravelry account.

So let's make the class!

**File:** `lib/ravelry/configuration.rb`

*Don't forget to require the file in `lib/ravelry.rb`!*

```ruby
module Ravelry
  class Configuration
    attr_accessor :access_key, :secret_key, :personal_key

    def initialize
      @access_key = nil
      @secret_key = nil
      @personal_key = nil
    end
  end
end
```

**Why `nil`?**

I set these values to `nil` for every new instance of the `Configuration` class because I want to be transparent about the purpose of the class to people who read the gem, and to people who want to contribute to the gem.

This indicates that these keys are *required* and are not provided by me, so they must be provided by the user.

In the future, when options are added, they may be provided with default values instead of `nil`, but for now we just need `nil`.

### Steps 3-4: Making the configuration accessible

Now that we've created our `Configuration` class, we need to set up our module to use it at the top level.

**It is very important** that you *require* your `lib/ravelry/configuration.rb` file **before** anything else, but especially before you write the code in the top level of your module. Basically, just require it first.

**File:** `lib/ravelry.rb`

```ruby
require 'ravelry/configuration'

module Ravelry
  class << self
    attr_accessor :configuration
  end

  def self.configuration
    @configuration ||= Configuration.new
  end

  def self.reset
    @configuration = Configuration.new
  end

  def self.configure
    yield(configuration)
  end
end
```

What's happening here? Let's break it down.

**Creating the `attr_accessor`**

Where it happens: 

```ruby
class << self
  attr_accessor :configuration
end
```

Modules in Ruby, just like classes, can have instance variables and all of the perks that come with them.

Because we want our users to be able to both read *and* write their configuration settings, we use `attr_accessor` here.

The fancy `class << self` bit tells our `Ravelry` module that this instance variable is on the module scope.

*Technically*, we only need an `attr_writer` here because we have a method that does the reading the instance variable `@configuration` for us. So why use `attr_accessor`? Frankly, I prefer it because it indicates we'll be reading whatever value set here later, even if we are *actually* retrieving it using a different method.

**Passing a block to our class** 

Where it happens:

```ruby
def self.configure
  yield(configuration)
end
```

There is no other way to put it: this shit is just magic. I really love Ruby at times like this.

When we call `Ravelry.configure`, we pass it a block that *actually creates a new instance* of the `Ravelry::Configuration` class using whatever we've set inside of the block.

So this:

```ruby
Ravelry.configure do |config|
  config.access_key = ''
  config.secret_key = ''
  config.personal_key = ''
end

Ravelry.configuration.access_key # => ''
```

Assuming the rest of your configuration set up is the name, is roughly equivalent to this:

```ruby
config = Ravelry::Configuration.new
config.access_key = ''
config.secret_key = ''
config.personal_key = ''

# two ways to access here here:
config.access_key # => ''
Ravelry.configuration.access_key # => ''
```

*So why bother `yield`ing a block? Why not just do it the second way?*

In short, the block is the more Ruby way of doing this. It's cleaner, and that way your code isn't just floating out in space in your application, it's nestled nicely inside of your namespaced block.

**Writing and reading the configuration**

Where it happens:

```ruby
def self.configuration
  @configuration ||= Configuration.new
end
```

The writing actually happens thanks to the `attr_accessor` we set up earlier, but this is where reading comes in.

When we call `Ravelry.configuration`, we're either going to return:

- The instance variable `@configuration` that we setup with our `configure` block

**or**

- A new instance of `Ravelry::Configuration` where everything is set to `nil`

*Why not just return `nil` if they haven't set it up yet?*

Returning `nil` isn't helpful to the users of our gem.

Instead, returning an instance of `Ravelry::Configuration` will show them what settings need to be configured if they haven't read the documentation.

Also, I think it's just makes you a nicer person.

**Resetting**

Where it happens:

```ruby
def self.reset
  @configuration = Configuration.new
end
```

I can't imagine a scenario (outside of testing) where I would want to reset my configuration, but someone out there might need it! This will reset the `@configuration` settings to `nil`.

### Step 5: Tests!

So it turns out I didn't actually write any tests for my configuration block... and I am only realizing that now as I am writing this blog post. I'm a bad role model.

Most of my tests user fixtures instead of making API calls. But! I have one test where I make *real* API call. 

I *did*, however, add the configuration settings into my global RSpec config:

**File:** `spec/spec_helper.rb`

```ruby
RSpec.configure do |config|
  config.before(:all) do
    Ravelry.configure do |config|
      config.access_key = ENV['RAV_ACCESS']
      config.secret_key = ENV['RAV_SECRET']
      config.personal_key = ENV['RAV_PERSONAL']
    end
  end
end
```

Without explicitly writing tests for my configuration setup, I know it works because my one API call where I use my `Ravelry.configuration` settings succeeds.

But what about my failure case? Well I better figure that out and write some tests...

### Step 6: Documentation

If you don't write documentation for your configuration setup, **people won't know how to use your gem** unless they are a wizard and can figure it out from your code.

This is mean, don't make them do this.

It takes about 30 seconds to write the 4 lines in your `README.md` explaining how to pass options to your configuration block. In fact, you can copy [this](https://gist.github.com/feministy/602bd01a0ce3f7b141ec#file-documentation-md)!

## You did it!

Ok, more accurately, I did it and you read about it. But! That means you can do it now!