---
title: 'Writing a lightweight authentication server with Sinatra'
category: blog
tags: technical, sinatra, omniauth, oauth, ravelry, blog
---

I work on a lot of side projects that use the [Ravelry API](http://www.ravelry.com/api). They can range from complex full-stack applications to small client-side apps where I only need to fetch API information periodically or simply authenticate to user. I did a little research on handling OAuth requests purely from the client side and wasn't happy with the results I was finding, especially in regards to security, so I decided to write a server-side solution. For these projects, I've taken to using a small, lightweight server written with Sinatra to process all of my authentication requests. This example uses the Ravelry Omniauth stratgey specifically, but can easily be applied to other Omniauth strategies or reworked to use another OAuth strategy.

READMORE

_This is part one of a two part series on OAuth. Part two will cover signing OAuth requests sent from our server after the initial authentication._

There are two ways to set up our little application:

1. A full stack application that has a server responsible for authentication, session storage, and rendering views
2. A server-side only application that handles your authentication and sessions, without any concern for views

In this post, I am only going to cover option two. There's a long list of pros and cons for both options, and you might settle for a different one depending on your needs.

Why option two, then? I want this server to be shared between multiple, independent client side apps, so that means one shared service. My shared service can be monitored more effectively if it's only doing the one thing that way I only have to worry about uptime for one backend thing (other than my Ravelry dependency, of course).

**Covered in part two:** With either setup, I can add additional authentication-required endpoints as needed and handle all of my authenticated requests through the my authentication server, or create an endpoint that serves as a proxy and simply passes the request straight through to the API regardless of the endpoint. 

## Setting up the application

For everything below, I'm using:

- Ruby 2.1.2
- [`sinatra`](http://www.sinatrarb.com/) 1.4.6
- [`omniauth-ravelry`](https://github.com/duien/omniauth-ravelry) 0.0.3

Everything else included below doesn't matter because it's dependent on the versions of either Sinatra of my Omniauth strategy.

**Why Sinatra?**

I like to use Sinatra for my smaller applications in lieu of Rails, especially when I don't need a database. 

Sinatra actually the first full-stack web development framework that I worked with, so I'm very familiar with it. You also don't need to write a lot of code to get up and running - the code is only as big as your application, really. Rails is totally overkill in this scenario. Sinatra provides me with out-of-the-box sessions and CORS policies, which I can customize easily simply by changing a few settings for the the Rack gems that Sinatra is _already_ using.

While I could have considered something like Node for this, there wasn't an existing Omniauth (or OAuth) strategy for Ravelry and the whole point of this exercise was for me to continue avoiding learning anything more about OAuth (lol just wait for part two, where I actually learn about OAuth _for reals_).

**Gemfile**

My Gemfile looks something like this:

```ruby
source 'https://rubygems.org'

ruby '2.1.2'

gem 'sinatra'
gem 'omniauth-ravelry'
gem 'sinatra-contrib'

group :development do
  gem 'pry'
  gem 'pry-nav'
end
```

You'll notice that I haven't pinned any versions here - that's because this application is so small that I am not super worried about upgrading. I'm using the most current verisons of my two biggest dependencies: `sinatra` and `omniauth-ravelry`. Additionally, as mentioned earlier, things like `sinatra-contrib` are actually tied to my version of `sinatra`, so no need to specify further there.

## The application

The application itself is pretty simple. I have two main files:

- `app.rb` the application! Voila. The end. You did it!
- `helpers.rb` aptly named, this contains my helper methods that I use mostly to manipulate and access my sessions. I could leave these in my `app.rb` file, but it's going to get a bit bigger later (see part two).

So here is what they look like:

File: `helpers.rb`

```ruby
helpers do
  def current_user?
    !session[:uid].nil?
  end

  def set_user
    info = session[:info]
    @user = { username: info['nickname'], first_name: info['first_name'], logged_in: current_user? }
  end

  def clear_session
    session[:info] = nil
    session[:uid] = nil
    @user = nil
  end

  def set_session
    session[:uid] = request.env['omniauth.auth']['uid']
    session[:info] = request.env['omniauth.auth']['info']
  end
end
```

Hilariously, I have more lines of configuration and `require`s than I do of actual code.

File: `app.rb`

```ruby
require 'sinatra'
require 'sinatra/json'
require 'omniauth-ravelry'
require 'json'
require './helpers'

# dev gems
require 'pry' if development?
require 'pry-nav' if development?
require 'sinatra/reloader' if development?

configure do
  set :sessions, true
  set :session_secret, ENV['SESSION_SECRET']
  use OmniAuth::Builder do
    provider :ravelry, ENV['RAV_ACCESS'], ENV['RAV_SECRET']
  end
end

get '/' do
  redirect to('/auth/ravelry') unless current_user?
  set_user unless @user
  json @user
end

get '/logout' do
  clear_session
end

get '/auth/ravelry/callback' do
  set_session
  set_user
  json @user
end
```

**How it works**

Omniauth provides you with a lot of stuff out of the box. Essentially doing this:

```ruby
use OmniAuth::Builder do
  provider :ravelry, ENV['RAV_ACCESS'], ENV['RAV_SECRET']
end
```

Is the bulk of the code I have to write. Omniauth handles a lot of the innerworkings for you, all you have to do is write what happens at the callback.

For an unauthenticated new user, the flow looks like this:

1. Send a get request or simply point your browser to `/` (whatever the url root is for my service)
2. Get redirected to `/auth/ravelry` (an Omniauth provided route), which takes you to Ravelry
3. Go through the steps of granting my app access to your Ravelry account
4. Return to the callback url `/auth/ravelry/callback`
5. User is now authenticated!

The only steps I had to write any code for are 1 and 4. Step 1 was actually a convenience thing that I provided for myself because I wanted to be able to hit the same endpoint if a user was logged in... which is to say you can replace the above with:

1. Send a get request or simply point your browser to `/auth/ravelry` (an Omniauth provided route for my service), which takes you to Ravelry
2. Go through the steps of granting my app access to your Ravelry account
3. Return to the callback url `/auth/ravelry/callback`
4. User is now authenticated!

The bulk of the work is handled in our callback, which you have to write for your authentication to be successful:

```ruby
get '/auth/ravelry/callback' do
  set_session
  set_user
  json @user
end
```

This calls my helper methods to store the session data that I need and create my instance of `@user`. At this point, my server only stores the user's ID and some basic profile information in a session. I'm not storing anything in a database, I'm only using my server's session storage.

In the next post, I'll cover how to send authenticated requests through to the Ravelry API by creating OAuth signatures.

## What about the Ravelry gem?

While I've been working to get the [Ravelry gem](https://github.com/ArtCraftCode/ravelry) up to par with the current API, I haven't yet had a chance to get authentication implemented. I'll be the first to admit that OAuth has always stumped me a little, and I haven't done much work with OAuth 1, so the gem is currently missing this critical piece. Perhaps after all of this work I'll be able to get that added to the gem!

