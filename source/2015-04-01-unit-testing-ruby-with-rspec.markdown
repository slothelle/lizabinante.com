---
title: "Unit Testing: Ruby with RSpec"
date:   2015-04-01
category: tutorial
tags: ruby, rspec, testing, tutorial
---

This tutorial assumes that you:

* know how to read, write, and run Ruby
* have familiarity with Ruby Gems
* know what unit testing is
* have downloaded the calculator (see below)
* know how to switch branches using a Git repository ([see here](/tutorial/using-git-branches/))

If any of that is beyond the scope of your abilities, you can likely still follow along.

## Download calculator

A basic tutorial on how to use the repos to navigate alongside the code is available here.

* [View repo](https://github.com/feministy/rspec_calculator)
* `git clone https://github.com/feministy/rspec_calculator.git`

## Our test object: a calculator

Here we have a simple Ruby calculator that has addition, subtraction, and multiplication calculating functionalities. It also allows you to clear or remove the last number inserted. The purpose of this calculator is to mimic a simple real-world object that many people are familiar with so you can learn RSpec by testing something you already know how to use:

```ruby
class Calculator
  attr_reader :nums

  def push(n)
    @nums ||= []
    @nums << n
  end

  def multiply
    @nums.inject(&:*)
  end

  def add
    @nums.inject(&:+)
  end

  def subtract
    @nums.inject(&:-)
  end

  def clear
    @nums = []
  end

  def remove_last
    @nums.pop(1)
    @nums
  end
end
```

### Using the calculator

If you want to play with the calculator interactively: load up irb (` $ irb `). Inside of `irb`:

```ruby
load 'calculator.rb'
```

And now you have access to your calculator file in `irb` where you can play.

First you create a Calculator object:

```ruby
calc = Calculator.new
```

Then, you add numbers to your calculator with `push`:

```ruby
calc.push(46)
calc.push(2)
```

You can add as many or as few numbers as you want to your calculator.

After you've added the numbers you want into your calculator, you can do math:

```ruby
calc.add # => 48
calc.multiply # => 92
calc.subtract # => 44
```

If you want to remove the last number you added:

```ruby
calc.remove_last # => [46]
```

Or, when you're ready to start over, you can:

```ruby
calc.clear # => []
```

At any point, you can check the contents of your calculator with:

```ruby
calc.nums # => [46]
```

**This calculator is not destructive**. So if you do:

```ruby
calc = Calculator.new
calc.push(3)
calc.push(5)
calc.nums # => [3, 5]
calc.add # => 8
calc.nums # => [3, 5]
```

You can see that `calc.nums` does not change. The contents are protected.

## Running existing tests

**Checkout branch: `part-01`**

Before you start modifying the code or writing your own tests, you should become familiar with the existing tests.

First, you will need to install RSpec. This requires RubyGems. Install with the command `gem install rspec`.

Second, you will need to have Ruby 1.9.3 or greater. The tests currently pass with Ruby 2.0.0 and 1.9.3; I cannot guarantee functionality for any other version of Ruby.

To run the tests, type: `rspec calculator_spec.rb`

This will return some pretty formatted information for you.

```
Calculator
  #multiply
    works
  #push
    adds new numbers
    accumulates numbers
    requires an argument
    does not accept more than 1 argument
  #nums
    should be nil on init

Finished in 0.00516 seconds
6 examples, 0 failures

Randomized with seed 21811
```

The test run order is randomized to help you spot any potential problems with your code or your test suite.

## Anatomy of a test

**Checkout branch: `part-02`**

Starting with an empty file, here is a simple test for addition:

```ruby
require 'rspec'
require_relative 'calculator.rb'

describe 'Calculator' do
  context '#add' do
    it 'returns the sum of all numbers' do
      calc = Calculator.new
      calc.push(2)
      calc.push(5)
      calc.push(3)
      total = calc.add
      expect(total).to be(10)
    end
  end
end
```

Now lets break this down.

We are requiring the `rspec` gem, and our `calculator.rb` file:

```ruby
require 'rspec'
require_relative 'calculator.rb'
```

Then we need to tell ourselves what it is we're testing. We use `describe` in a larger sense here to indicate that it is the object we're testing. This is usually a class or a module. `describe` opens a block for us.

We use `context` to open another block that indicates what method we're testing. In this case, `#add`. The convention of using a `#` before the method name is something helpful you can do to indicate that your test is a unit test that covers only that method.

```ruby
describe 'Calculator' do
  context '#add' do
```

This is the description of what our test is actually testing for.

```ruby
    it 'returns the sum of all numbers' do
```

We have to make a new Calculator object:

```ruby
      calc = Calculator.new
```

And put some numbers into our Calculator:

```ruby
      calc.push(2)
      calc.push(5)
      calc.push(3)
```

Now that our calculator has numbers, we can add them together to get a total using the add method.

```ruby
      total = calc.add
```

This is our assertion. Read in plain English: `we expect the total to be 10`. Our test will pass if the total is 10. Our test will fail is the total is not 10 (the Integer).

```ruby
      expect(total).to be(10)
    end
  end
end
```

## Additional RSpec resources

* [Better Specs](http://betterspecs.org/) - comparison of good specs and bad specs
* [RSpec Expectations](http://rubydoc.info/gems/rspec-expectations/frames) - guides for using RSpec