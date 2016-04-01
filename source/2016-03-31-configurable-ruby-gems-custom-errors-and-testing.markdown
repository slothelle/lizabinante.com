---
title: 'Configurable Ruby gems: Custom error messages and testing'
category: blog
tags: blog, technical, ruby, gems
---

Previously, I've written about writing a Ruby gem with options for configuration. This is a really great thing to offer your gem users for a number of reasons, but how can we make the interface more helpful to our users? Things like custom error messages for configuration errors and solid testing are two easy ways to get started!

READMORE

_This is the second post I've written on configurable Ruby gems. You can read the first post, which includes a beginner-friendly explanation of writing Ruby gems, here: [Creating a configurable Ruby gem](/blog/creating-a-configurable-ruby-gem/)._

This post uses examples taken from my [Ravelry Ruby gem](https://github.com/ArtCraftCode/ravelry), but it doesn't require you to be familiar with Ravelry or the gem.

## Quick recap: our configuration setup

Much of this is from the original post. There will be changes to this code below.

In our `lib/ravelry/configuration.rb` file, we create our `Configuration` class:

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

Then, we add the `Configuration` object into our `Ravelry` module:

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

Our end user will configure their gem like this:

```ruby
Ravelry.configure do |config|
  config.access_key = ''
  config.secret_key = ''
  config.personal_key = ''
end
```

## Creating custom errors

Thanks to inheritance in Ruby, it's super easy to create new error classes. You simply inherit from `StandardError` (or whatever preferred [exception](http://ruby-doc.org/core-2.2.0/Exception.html) fits your scenario).

Because I am planning on building this gem out some more, I wanted to create my own `Errors` module with separate classes for distinct error messages.

The simplest way of doing this is:

- create a `Ravelry::Errors` module
- create classes for my errors that inherit from `StandardError`
- raise the errors as needed

And boom! We're done.

### `raise` or `rescue`?

There are tons of different ways to handle errors in Ruby and I'm not going to go into the mechanics - partly because it doesn't matter here, but also because I am not a domain expert on this area of Ruby.

But! There is one significant thing I'd like to point out, and that's the difference between `raise` and `rescue`.

- `raise` causes an exception, and will *terminate your program*.
- `rescue` will _not_ cause an exception and your program will continue to run.

`raise` and `rescue` are actually two different things and can be used together: you don't have to pick one or the other. You can, for example, use `rescue` to handle exceptions _after_ an error is `raise`d using a `begin`/`rescue` block; this is where you may want to log something to an error reporting service or what have you.

For the purposes of my gem, I'm using `raise` because *the program cannot–and should not–continue* without proper configuration.

## Our first `Errors` class

This is pretty straightforward.

**File: `lib/errors/configuration.rb`**

```ruby
module Ravelry
  module Errors
    class Configuration < StandardError; end
  end
end
```

Because we're inheriting from `StandardError`, we don't actually have to provide any additional information here.

Say I want to validate the presence of `@secret_key`. If I want to call this, I simply do as follows in my gem (somewhere):

```ruby
raise Errors::Configuration unless @secret_key
```

This will raise the error `Ravelry::Errors::Configuration` and everything will be dandy.

## Providing custom error messages

Perhaps we want to provide our users with a little more information about their error messages (we do).

The dead simple way to do this is to pass a string to our error message call. So the above snippet becomes:

```ruby
raise Errors::Configuration, "Ravelry secret key missing!" unless @secret_key
```

Then, when our `Ravelry::Errors::Configuration` error is raised, we also get the friendly message letting us know that we're missing the secret key.

In our `Configuration` class, we now have:

**File: `lib/ravelry/configuration.rb`**

```ruby
module Ravelry
  class Configuration
    attr_writer :access_key, :secret_key, :personal_key

    def initialize
      @access_key = nil
      @secret_key = nil
      @personal_key = nil
    end

    def access_key
      raise Errors::Configuration, "Ravelry access key missing!" unless @access_key
      @access_key
    end

    def secret_key
      raise Errors::Configuration, "Ravelry secret key missing!" unless @secret_key
      @secret_key
    end

    def personal_key
      raise Errors::Configuration, "Ravelry personal key missing!" unless @personal_key
      @personal_key
    end
  end
end
```

Seems a bit repetitive, doesn't it? More on that later.

## Testing

In my last post, I lamented that I hadn't written tests for the configuration block of the gem and that I should probably do that. Well, happy news, I have!

To keep things abbreviated here, I'll be pretending there is only one configuration option (`@secret_key`) in my examples.

### Preventing my tests from exploding

Because I've now required the gem to be configured properly and raise an error if not, I need to make sure my test environment respects that.

To do this, I need to add a `config.before(:all)` block with these settings to my spec helper. That looks like this: 

**File: `spec/spec_helper.rb`**

```ruby
RSpec.configure do |config|
  # other settings...
  config.before(:all) do
    Ravelry.configure do |config|
      config.secret_key = ENV['RAV_SECRET']
    end
  end
end
```

Now none of our tests should be failing. (They're not, hooray!)

### Test cases

There are a few things we need to test about our configuration:

- the configuration block applies the correct settings
- the configuration object returns the setting we applied changes to
- the `reset` functionality clears our configuration settings
- the proper error is raised if we try and use an unconfigured gem

Easy enough!

_You'll see that I am using `context` blocks below in addition to my tests. This is because I actually have more configuration keys than just the one shown below._

**File: `spec/ravelry/configuration_spec.rb`**

```ruby
require_relative '../spec_helper'

describe Ravelry::Configuration do
  context 'with configuration block' do
    it 'returns the correct secret_key' do
      expect(Ravelry.configuration.secret_key).to eq(ENV['RAV_SECRET'])
    end
  end

  context 'without configuration block' do
    before do
      Ravelry.reset
    end

    it 'raises a configuration error for secret_key' do
      expect { Ravelry.configuration.secret_key }.to raise_error(Ravelry::Errors::Configuration)
    end
  end

  context '#reset' do
    it 'resets configured values' do
      expect(Ravelry.configuration.secret_key).to eq(ENV['RAV_SECRET'])

      Ravelry.reset
      expect { Ravelry.configuration.secret_key }.to raise_error(Ravelry::Errors::Configuration)
    end
  end
end
```

_Technically_, our test for `#reset` should be on `Ravelry`, not `Ravelry::Configuration`, since it lives on the module level. However, I like to test this in the `Configuration` specs because it makes more sense to me semantically. It also expresses to my gem contributors how the configuration works.

## What about that repetitive code?

As noted above, our error messages got pretty repetitive very quickly. It seems silly to pass nearly identical strings as error messages each time. Also, do we really need that conditional in there, or could we make that a little better, too? Fortunately, Ruby provides us with mechanisms to handle all of these scenarios.

But! That's not for this post - or [this version (0.1.0)](https://github.com/ArtCraftCode/ravelry/tree/0.1.0) of my gem. That's for the next post, and the next version!
