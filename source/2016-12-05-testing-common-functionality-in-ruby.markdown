---
title: 'Testing common functionality across Ruby modules (and classes)'
category: blog
tags: blog, technical, ruby, testing
---

One of my favorite things to do when writing tests is find little ways to _write fewer tests_ but still have the same test coverage. While working on [Cryptozoologist](https://github.com/feministy/cryptozoologist), a Rubygem I use to teach gem writing workshops, I utilized a few simple tricks to do this.

READMORE

First, a quick overview of Cryptozoologist: Cryptozoologist generates random strings from animal, clothing item, and color pairings. You could get something like "orange-clownfish-turtleneck" or "magenta-three-toed-sloth-shoe-horn". It's an entirely pointless and relatively simple gem that only requires you understand arrays and modules in Ruby. This simplicity makes it an excellent teaching tool, even if the gem itself is relatively useless.

Cryptozoologist offers a simple API so that without any configuration you call `Cryptozoologist.generate` and blammo! `'tomato-zebra-sweatshirt'`

The gem additionally offers [configuration options](https://github.com/feministy/cryptozoologist#configuration) that allow you to include or exclude specific types of word lists from your dictionaries, change your delimiter, customize the order, or add a quantity word to your string. The included word list types by default are:

- Colors: paint names, web safe colors
- Animals: common, mythical (_hello, Harry Potter obsession..._)
- Clothing items

These correspond to several files that contain dictionaries. 

The file tree looks something like this:

```
dictionaries/
  animals/
    common.rb
    mythical.rb
  colors/
    paint.rb
    web_safe.rb
  clothing.rb
  quantity.rb
```

Let's look at one of the dictionaries with subtypes, `Cryptozoologist::Dictionaries::Animals::Mythical` (oh yes, look at that sweet, sweet nested module):

```ruby
module Cryptozoologist
  module Dictionaries
    module Animals
      module Mythical
        def self.list
          [
            "abraxan",
            "aethonan",
            "alicorn",
            "banshee",
            "basilisk",
            "bigfoot",
            "blast ended skrewt",

            # removed for brevity...

            "will o the wisp",
            "werewolf",
            "wraith",
            "zombie"
          ]
        end
      end
    end
  end
end
```

I wanted to test *each* dictionary and make sure it behaved correctly and didn't mix itself up with the contents from other dictionaries. Because this gem is so simple, it's kind of a silly test. How would one module suddenly return content from another module? It probably wouldn't unless I did something very intentional and very odd, but because this is a gem I use to teach people things this is a nice and simple case to do some fun stuff in the tests. (Fun stuff in tests? A thing no one has ever said, ever)

I decided that I wanted to test a few things:

- dictonaries without subdictionaries (clothing, quantity): it should have a word list, shouldn't include other word lists items, and shouldn't be a valid exclusion for the configuration block
- dictionaries with submodules (animals, colors): it should have a word list, should act as a valid exclusion for configuration, and shouldn't include words from other lists

I ended up having 6 different dictionaries to test. 

Four with subtypes:

```ruby
Cryptozoologist::Dictionaries::Animals::Common
Cryptozoologist::Dictionaries::Animals::Mythical
Cryptozoologist::Dictionaries::Colors::Paint
Cryptozoologist::Dictionaries::Colors::WebSafe
```

And two without:

```ruby
Cryptozoologist::Dictionaries::Clothing
Cryptozoologist::Dictionaries::Quantity
```

The whole test file is only 94 lines long and could honestly probably be slimmed down a bit as you'll see that some of those tests cases actually _overlap_.

First, I created variables containing the information about my dictionaries. Note that the keys listed in each `subtypes` array are the method names used within the gem to return the word lists.

```ruby
subdictionaries = {
  "animals": {
    subtypes: [:common, :mythical],
    common: Cryptozoologist::Dictionaries::Animals::Common,
    mythical: Cryptozoologist::Dictionaries::Animals::Mythical
  },
  "colors": {
    subtypes: [:paint, :web],
    paint: Cryptozoologist::Dictionaries::Colors::Paint,
    web: Cryptozoologist::Dictionaries::Colors::WebSafe
  }
}

dictionaries = { 
  "clothing": Cryptozoologist::Dictionaries::Clothing,
  "quantity": Cryptozoologist::Dictionaries::Quantity
}
```

Armed with my hashes containing this valuable data, it was a simple matter of doing some quick iteration. The dictionaries without subtypes were the easiest to test. You'll see that I leveraged `send` here to do some magic, converting the string I use in my test descriptions into a symbol:

```ruby
dictionaries = { 
  "clothing": Cryptozoologist::Dictionaries::Clothing,
  "quantity": Cryptozoologist::Dictionaries::Quantity
}

dictionaries.each do |type, dictionary|
  context "##{type}" do
    it "has a #{type} list" do
      expect(Cryptozoologist::Dictionaries.send(type.to_sym).length).to be > 1
    end

    it "contains #{type} words" do
      expect(Cryptozoologist::Dictionaries.send(type.to_sym).include?(dictionary.list.sample)).to be true
    end

    # ... more tests
  end
end
```

You may be wondering _"why convert the string to a symbol when you could just use a symbol and interpolate it?"_. Yes, I could do that, but I'd prefer to read English instead of code when reading my test descriptions - that little colon can increase the mental overhead require to parse the results, and it's a small thing that makes me happy. Let me have my string!!

Writing tests for the subdictionaries was similar, here's an excerpt:

```ruby
subdictionaries = {
  "animals": {
    subtypes: [:common, :mythical],
    common: Cryptozoologist::Dictionaries::Animals::Common,
    mythical: Cryptozoologist::Dictionaries::Animals::Mythical
  },
  # other subdictionary...
}

subdictionaries.each do |type, subdictionary|
  context "##{type}" do
    subdictionary[:subtypes].each do |subtype|
      it "contains #{subtype} #{type}" do
        sublist = subdictionary[subtype].list
        expect(Cryptozoologist::Dictionaries.send(type.to_sym).include?(sublist.sample)).to be true
      end

      it 'filters out exclusions' do
        Cryptozoologist.reset

        Cryptozoologist.configure do |config|
          config.exclude = [subtype]
        end

        sublist = subdictionary[subtype].list
        expect(Cryptozoologist::Dictionaries.send(type.to_sym).include?(sublist.sample)).to be false
      end
    end

    # ... more tests for everything else
  end
end
```

Iterating over the `subtypes` list allowed me to test *all* of the configuration options available to my gem, thus enabling me to sleep soundly at night knowing that someone won't be stuck with a string that's as sad as `'orange-bluebell-black'`!

While this gem is clearly very simple and the test cases a bit overkill, this kind of thought process is not restricted to simple strings and arrays. This pattern can be leveraged to test things like inheritance, allowing us to enforce things like what type of object is returned by a child class or to make sure that methods from the parent class aren't overridden. 

As long as you keep your test declaration clear as to *what* you're testing, this is a handy little tool to keep in your toolbox to minimize repetitive test writing.


