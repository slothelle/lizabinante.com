---
title: "Deploying Rails 4.1 apps with Resque to Heroku"
category: blog
tags: blog, technical, rails, heroku, resque
---

The Heroku guides for deploying Rails apps encourage you to do so using a Procfile and Unicorn. Not being super deployment savvy, I tend to follow the instructions provided to me.

... that is, until they *completely, totally, and utterly fail me*.

READMORE

After I added Resque to a Rails app I was working on, I was having a hell of a time deploying it to Heroku: first Redis wouldn't start, and then when Redis finally started, my workers wouldn't start. Nothing in the guides for Resque, or two Redis add ons (RedisToGo and Redis Cloud), was helpful or even remotely close to correct.

Some intense Googling turned up some issues related to older versions of Resque (1.22.1, which Heroku uses in their guides) and some compatibility issues with Unicorn.

I don't think the compatibility issues have been resolved, but I was able to come up with a fix that works locally and on production without any hiccups.

I am using Redis Cloud in my configuration below, but this should work with other Redis add ons.

## Gemfile

This guide works specifically with the following versions of Rails, Unicorn, and Resque:

```ruby
gem 'rails', '4.1.1'
gem 'resque', '~> 1.24.1'
gem 'unicorn', '~> 4.6.2'
```

I can't make any promises about other versions.

## Procfile

This is pretty standard for Heroku, Resque, and Rails.

```ruby
web: bundle exec unicorn -p $PORT -c ./config/unicorn.rb
resque: env TERM_CHILD=1 QUEUE=* bundle exec rake resque:work
```

## Redis

Create a new file: `config/initializers/redis.rb` and put this inside of it:

```ruby
if ENV["REDISCLOUD_URL"]
  $redis = Resque.redis = Redis.new(:url => ENV["REDISCLOUD_URL"])
end
```

## Resque

We need access to the Rake tasks. Create a new file: `lib/tasks/resque.rake`.

```ruby
require 'resque/tasks'
 
task "resque:preload" => :environment
```

## Unicorn

The last thing we need to do is configure Unicorn, which turned out to be the most difficult part.

In your `config/unicorn.rb` file, put:

```ruby
worker_processes Integer(ENV["WEB_CONCURRENCY"] || 3)
timeout 15
preload_app true
 
 
before_fork do |server, worker|
  Signal.trap 'TERM' do
    puts 'Unicorn master intercepting TERM and sending myself QUIT instead'
    Process.kill 'QUIT', Process.pid
  end
 
  defined?(ActiveRecord::Base) and
    ActiveRecord::Base.connection.disconnect!
 
  if defined?(Resque)
    Resque.redis.quit
    Rails.logger.info('Disconnected from Redis')
  end
end
 
after_fork do |server, worker|
  Signal.trap 'TERM' do
    puts 'Unicorn worker intercepting TERM and doing nothing. Wait for master to send QUIT'
  end
 
  if defined?(Resque)
    Rails.logger.info('Connected to Redis')
  end
end
```

And you should be all set to deploy to Heroku with ease!