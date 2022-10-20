---
title: 'Modules and self in Ruby'
category: blog
tags: blog, technical, ruby
---

I love modules in Ruby: they're like little folders for your code (ok, well, sometimes they are actually representative of folders in your code). It took me awhile to get comfortable using them, particularly because the concept of `self` can be difficult to understand at first. Here's a quick overview of `extend`, `include`, and `self` as they pertain to modules in Ruby.

READMORE

_This post is written using Ruby 2.3.1_

We're going to use the following module and two classes throughout this post:

```ruby
module Animals
end

class Zoo
end

class PetStore
end
```

They'll have slightly different things inside of them as we go along!

## `extend` vs. `include` in classes

A common use case for modules is to use them to provide some shared functionality that you also want to make available in a class. You can use both `extend` and `include` to accomplish this, but there are slight differences between the two.

### `include`: instance methods

Let's start with `include`: this adds the methods from the module as **instance-level** methods. To access the methods from the module you've `include`d, you have to make a new instance of your class.

```ruby
module Animals
  def mammals
    ["sloths", "puppies", "porcupines"]
  end
end

class Zoo
  include Animals
end

zoo = Zoo.new
zoo.mammals # => ["sloths", "puppies", "porcupines"]
```

Another way to look at this is using one of my favorite Ruby tricks: subtracting `Object.methods` from your Ruby class to get a list of methods that are directly defined on your class (and the classes it inherits from).

```ruby
zoo = Zoo.new
zoo.methods - Object.methods # => mammals
```

This returns `mammals`, the only instance method we've defined on our `Zoo` class.

Let's compare this behavior to `extend`.

### `extend`: class methods

Slightly different is `extend`: this adds the methods from the module as **class-level** methods. To access the methods from the module, you don't have to instantiate a new object, you can simply call them.

```ruby
module Animals
  def mammals
    ["sloths", "puppies", "porcupines"]
  end
end

class PetStore
  extend Animals
end

PetStore.mammals # => ["sloths", "puppies", "porcupines"]
```

But what happens if I try and call `mammals` on an instance of `PetStore`?

```ruby
pet_store = PetStore.new
pet_store.mammals # => undefined method `mammals' for #<PetStore:0x007f858a919f30> (NoMethodError)
```

The method doesn't exist!

Using our same trick from before, we can verify this is what we expected by comparing both class methods _and_ the instance methods on `PetStore`.

```ruby
PetStore.methods - Object.methods # => mammals

pet_store = PetStore.new
pet_store.methods - Object.methods # =>
```

There are **no instance methods** on the `PetStore` class!

## `self` in modules

We'll start by adding the small `mammals` method to our `Animals` module:

```ruby
module Animals
  def mammals
    ["sloths", "puppies", "porcupines"]
  end
end
```

What happens when we call `Animal.mammals`?

```ruby
# undefined method `mammals' for Animals:Module (NoMethodError)
```

Using our new favorite Ruby trick, let's see what we've got:

```ruby
Animals.methods - Object.methods # =>
```

Again, it returns nothing. So, how do we call that teeny tiny `mammals` method?!

Ruby offers two ways of calling a method directly from a module:

- `extend self`
- prefixing the method name with `self` 

### `extend self`

By adding a simple, one line change to our module we can call the `mammals` method:

```ruby
module Animals
  extend self

  def mammals
    ["sloths", "puppies", "porcupines"]
  end
end

Animals.mammals # => ["sloths", "puppies", "porcupines"]
```

One important thing to note about `extend self` is that it will apply this change to *all* of the methods in your module. So when I add a new method, I automagically get access to it:

```ruby
module Animals
  extend self

  def mammals
    ["sloths", "puppies", "porcupines"]
  end

  def sea_creatures
    ["narwhal", "starfish", "dolphin"]
  end
end

Animals.mammals # => ["sloths", "puppies", "porcupines"]
Animals.sea_creatures # => ["narwhal", "starfish", "dolphin"]
```

Super handy if you're calling the methods directly on the module itself! 

While I can't think of a scenario where I would want to both `extend self` in my module *and* `include` or `extend` that module's functionality in a class, it's important to note that you **can do this** if you want to. Basically, using `extend self` doesn't change the behavior of `include` *or* `extend` in your classes:

```ruby
module Animals
  extend self 

  def mammals
    ["sloths", "puppies", "porcupines"]
  end
end

class Zoo
  include Animals
end

Zoo.mammals # => undefined method `mammals' for Zoo:Class (NoMethodError)

zoo = Zoo.new
zoo.mammals # => ["sloths", "puppies", "porcupines"]

class PetStore
  extend Animals
end

PetStore.mammals # => ["sloths", "puppies", "porcupines"]

pet_store = PetStore.new
pet_store.mammals # => undefined method `mammals' for #<PetStore:0x007f858a919f30> (NoMethodError)
```

But what if I only want to access *some* of the methods directly using my module? `self` to the rescue!

### `self` method name prefix

Using `self` as a method name prefix *only extends the methods you tell it to*. It works like this:

```ruby
module Animals
  def self.mammals
    ["sloths", "puppies", "porcupines"]
  end
end

Animals.mammals # => ["sloths", "puppies", "porcupines"]
```

Essentially the same functionality as using the `extend self` trick we learned about earlier. Let's make our module a little bigger and see how that changes:

```ruby
module Animals
  def self.mammals
    ["sloths", "puppies", "porcupines"]
  end

  def sea_creatures
    ["narwhal", "starfish", "dolphin"]
  end
end

Animals.mammals # => ["sloths", "puppies", "porcupines"]
Animals.sea_creatures # => undefined method `sea_creatures' for Animals:Module
```

Using `self` this way in a module makes it behave similarly to using `self` to create a class-level method in your class.

How does this impact `include` and `extend`? We'll break it down one-by-one since it's a little different.

#### `self` and `include`

Using `include` in our class, we see some behaviors we expected and others that might be a little surprising:

```ruby
module Animals
  def self.mammals
    ["sloths", "puppies", "porcupines"]
  end

  def sea_creatures
    ["narwhal", "starfish", "dolphin"]
  end
end

class Zoo
  include Animals
end

Zoo.mammals # => undefined method `mammals' for Zoo:Class (NoMethodError)
Zoo.sea_creatures # => undefined method `sea_creatures' for Zoo:Class (NoMethodError)

zoo = Zoo.new
zoo.mammals # => undefined method `mammals' for #<Zoo:0x007faca4020168> (NoMethodError)
zoo.sea_creatures # => ["narwhal", "starfish", "dolphin"]
```

We can't call `mammals` on the `Zoo` class *or* an instance of `Zoo`. We can only call the `mammals` method **directly on the `Animals` module**. 

_waves goodbye to `mammals` forever..._

The `sea_creatures` method behaves exactly as we would have expected based on our previous experiences with `include`. 

To summarize, using `include` produces the following behavior:

1. `mammals`, which is defined in our `Animals` module prefixed with `self`:
  -  is not accessible by the `Zoo` class, or instances of the `Zoo` class
  - is accessible as a method on the `Animals` module
2. `sea_creatures`, which is defined in the module as a regular method:
  - is accessible to *instances of our `Zoo` class*
  - is not accessible directly through our `Animals` module

Now let's look at how this behavior changes with `extend`. 

#### `self` and `extend`

We're still using the same setup as above, this time with `extend` instead.

```ruby
module Animals
  def self.mammals
    ["sloths", "puppies", "porcupines"]
  end

  def sea_creatures
    ["narwhal", "starfish", "dolphin"]
  end
end

class PetStore
  extend Animals
end

PetStore.mammals # => undefined method `mammals' for PetStore:Class (NoMethodError)
PetStore.sea_creatures # => ["narwhal", "starfish", "dolphin"]

pet_store = PetStore.new
pet_store.mammals # => undefined method `mammals' for #<PetStore:0x007f858a919f30> (NoMethodError)
pet_store.sea_creatures # => undefined method `sea_creatures' for #<PetStore:0x007faad380a9a0> (NoMethodError)
```

As with `include`, the `sea_creatures` method behaves exactly as it did when we used `extend` last time and `mammals` is locked away and only accessible on the `Animals` module.

Here's our summary:

1. `mammals`, which is defined in our `Animals` module prefixed with `self`:
  -  is not accessible by the `PetStore` class, or instances of the `PetStore` class
  - is accessible as a method on the `Animals` module
2. `sea_creatures`, which is defined in the module as a regular method:
  - is accessible to the `PetStore` class as a class method
  - is not accessible directly through our `Animals` module

## Our newfound sense of `self`

So that's it! Not so scary afterall, right?

With modules, you can:

- define methods only accessible directly on the module
- `include` methods defined in your module as methods on instances of your classes
- `extend` methods defined in your module on your classes as class methods
- use `extend self` to make all of your methods in your module directly accessible to the module
- create a mixture of methods that are accessible *only* on the module and methods *only* on instances of your class
- create a mixture of methods that are accessible *only* on the module and methods *only* as class methods on your class

Lots and lots of options to go forth and code!




