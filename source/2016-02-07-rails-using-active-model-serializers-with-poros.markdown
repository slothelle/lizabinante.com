---
title: 'Rails: Using Active Model Serializer with POROs (without Active Record or Active Model)'
category: blog
tags: technical, rails, active model, serializers, blog
---

Recently I was working on a Rails application with non-persisted models (aka plain old Ruby objects). We didn't need a database, which means we didn't need Active Record either. We opted not to use Active Model, either. We didn't need most of the stuff it provides because our application was mostly responsible for querying a few APIs and services and then performing some additional logic on the data we received. Fancy shit, right? Sometimes I feel really cool typing sentences like that.

Anyways! The application was originally written using JBuilder to serialize the data, and it was a mess to reason about. The serializers were long and didn't share code between the two, even though our two endpoints would sometimes return some of the same data.

We opted to switch from JBuilder to Active Model Serializer because:

1. We'd used it in the past and were familiar with it so we could implement it faster

... and that's it.

We briefly considered using [Rabl](https://github.com/ccocchi/rabl-rails), but didn't want to take the time during this project to learn a new syntax.

The documentation for Active Model Serializer is hit or miss, and isn't always right depending on the Rails version you're using. Active Model Serializer is currently in flux, with a large refactor happening for the upcoming version 0.10. Additionally, the implementation for POROs varies widely between version 0.9 and 0.8. This made our work a little complicated.

## 0.9 or 0.8?

We opted to use version 0.8 because 0.9 required a lot more code in our models. And, frankly, the code that 0.9 required us to write inside of our models didn't make sense to me: you had to create a method that would allow you to read attributes on the model. It was a really silly utility method that only required a few lines of code, but you either had to **(a)** extract it so it was shared and include it in *all* of your models, or **(b)** copy the code across all of your models. Neither of these seemed to be good options to me... isn't that was `attr_reader` is for anyways?! (*Answer: yes.*)

On top of that, version 0.10 (forthcoming) is being built on the 0.8 code. While 0.10 won't be backwards compatible with 0.8, the fact that it has the same code as the base made us optimistic that we would be able to upgrade easier. 

We don't have any specific plans to upgrade the serializers or make lots of changes to the application, but we wanted to keep our options open in case we needed to do security upgrades or there were unforseen changes needed down the line.

So, we opted for 0.8.

## Versions

Our application is using the following versions of Rails and Active Model Serializers:

```ruby
gem 'rails', '~> 4.2.5'
gem 'active_model_serializers', '~> 0.8.3'
```

I can't guarantee this will work with any other version of Active Model Serializers, but you're likely ok with most Rails 4 versions.

Just to reiterate here, we are *not* using Active Model in any capacity.

## Our application!

Let's say our application is returning information about dogs. Why dogs? Dogs are better than people, that's why. We should all be more like dogs.

**Model**

So here's our model. Look how simple it is!

```ruby
class Dog
  include ActiveModel::SerializerSupport
  attr_accessor :name, :breed, :gender, :cuteness_level
end
```

The most important thing here is that second line, the include:

```ruby
include ActiveModel::SerializerSupport
```

This brings in the *very important things* our serializers need, which we would have had if we were using Active Record (and Active Model). Full disclosure, I don't know what those things are, but I do know that you can't use Active Model Serializer for POROs without this.

**Controller**

In our controller, we need to get our dogs and render the dogs as json:

```ruby
def index
  @dogs = list_of_dogs.map { |thing| Dog.new(thing) }
  render json: @dogs, each_serializer: DogSerializer
end
```

Because `@dogs` is an array and not an Active Record object, we have to use `each_serializer`. Ignore what the documentation says here. It will blow up if you try to use anything else. `each_serializer` tells Active Model Serializer that we're sending it an Array object, not an Active Record object. 

I haven't investigated performance and pagination differences between `each_serializer` and the standard `serializer` option, so take that with a grain of salt and maybe do some performance testing if your application is returning a lot of data.

**Serializer**

Your serializers should go in the `app/serializers` folder.

Do yourself a favor and follow the Rails/Active Record naming patterns, it will make your life a lot easier as your architecture becomes more complex.

Here is our very simple serializer:

```ruby
class DogSerializer < ActiveModel::Serializer
  attributes :name, :breed, :gender, :cuteness_level
end 
```

And it will return the following json from our controller:

```javascript
{
  dogs: [
    {
      name: 'Kiwi Bird Fruit Dog',
      breed: 'Mini American Shepherd',
      gender: 'little lady',
      cuteness_level: 100000 
    }
  ]
}
```

## Relationships

This is all fine and dandy, but as we all know, most data has relationships. 

My dog, Kiwi, has lots of toys. She also has lots of treats. Now I want our application to return information about her toys and treats, too.

**Models**

```ruby
class Treat
  include ActiveModel::SerializerSupport
  attr_accessor :flavor, :size
end

class Toy
  include ActiveModel::SerializerSupport
  attr_accessor :color, :destroyed, :name
end
```

**Controller**

There are two ways to do this.

*Option one:* your models don't know anything about each other and you how create them, but you want to return them all in the same serializer. I can see how you might want this but it's kinda weird, so I guess you do you.

```ruby
def index
  @dogs = list_of_dogs.map { |thing| Dog.new(thing) }
  @treats = list_of_tasty_treats.map { |thing| Treat.new(thing) }
  @toys = list_of_destroyed_toys.map { |thing| Treat.new(thing) }
  data = { dogs: @dogs, treats: @treats, toys: @toys }
  render json: data, each_serializer: DogSerializer
end
```

*Option two:* you mimic the Active Record relationships and assume your `Toy` and `Treat` objects are available on your dog as `dog.treats`, just like a `has_many` relationship. In this case, your controller would look something like this:

```ruby
def index
  @dogs = list_of_dogs_treats_and_toys.map { |thing| Dog.new(thing) }
  render json: @dogs, each_serializer: DogSerializer
end
``` 

This most resembles the relationships Active Model gives you when you're using an Active Record backed application. Because of this, I tend to use this method. It makes for slightly messier models now that you have to mimic the Active Record relationship setup and create your `Dog`, `Treat`, and `Toy` all at the same time.

*But* because your application doesn't have a database it's already probably simpler than most so you could extract away some of that annoyance out of your model without feeling too much pain or code complexity. If you're asking me (and you are, you're reading this), you should go with option two.

**Serializers**

Now, onto the updated serializers. Because we've used the standard naming pattern Active Model expects, we only need to add two lines to our `DogSerializer`:

```ruby
class DogSerializer < ActiveModel::Serializer
  attributes :name, :breed, :gender, :cuteness_level

  has_many :treats
  has_many :toys
end 
```

If this throws an error for you, try this:

```ruby
has_many :treats, serializer: TreatSerializer
has_many :toys, serializer: ToySerializer
```

And then make some new `Treat` and `Toy` serializers:

```ruby
class TreatSerializer < ActiveModel::Serializer
  attributes :flavor, :size
end 

class ToySerializer < ActiveModel::Serializer
  attributes :color, :destroyed, :name
end 
```

And now we have this json:

```javascript
{
dogs: [
    {
      name: 'Kiwi Bird Fruit Dog',
      breed: 'Mini American Shepherd',
      gender: 'little lady',
      cuteness_level: 100000,
      treats: [
        {
          flavor: 'Bacon and cheddar',
          size: 'large'
        }
      ],
      toys: [
        {
          color: 'yellow',
          destroyed: true,
          name: 'Not-so-big bird'
        }
      ]
    }
  ]
}
```

And we're done!

## Testing...?

There was nothing in the documentation for testing, so we tested our controller responses. This has the nice added benefit of testing our controllers, even if it does make our test code a bit longer.

