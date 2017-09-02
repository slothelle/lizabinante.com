---
title: 'Cryptozoologist as password generator: is this really a good idea...?'
category: blog
tags: blog, technical, ruby
---

Honestly I never knew I needed to generate so much random text until I had a random text generator all my own. Ever since I wrote [Cryptozoologist](https://github.com/feministy/cryptozoologist), I seem to have a lot more reasons to lorem ipsum it up. `pygmy-puff-jumper-polar-drift`? I mean, obviously. But it got me thinking: could you use Cryptozoologist as a random password generator? And if you could, how ... terrible of an idea was it?

READMORE

It's probably a pretty terrible idea. There's a reason I don't work in infosec. I'm sure someone in infosec can tell you if this is _actually_ a good idea. 

(It's certainly a cute idea though)

Cryptozoologist has 4 main dictionaries that can be used by the `random` function if you configure the optional quantity dictionary: animals (a whopping 606 words), clothing items (less impressive at 276 words), colors (a disappointing 195 words), and quantity (only 85 words).

Cryptozoologist's `random` functionality is order dependent, meaning that the dictionaries will always be used in the same order. There's no random order arrangement of words, they always appear as `animal-clothing-color-quantity`. You can change what delimiter you're using, but it will always be the same delimiter throughout your generated string.

It was pretty straightforward to calculate the number of available unique passwords: 606 &times; 276 &times; 195 &times; 85 = 2,772,268,200

Let me write that out for you in the big fancy font because it is _pretty cool_:

## 2,772,268,200

That's a lot.

Now, let's say I add a new feature to Cryptozoologist that lets you generate the string in random order: it doesn't depend on the order of the dictionaries, just one word from each dictionary. How many unique possibilities would there be then? It would be `4!` (aka 24, or 4 factorial in fancy math speak) &times; 2,772,268,200. This is a lot.

Like... a lot a lot.

## 66,534,436,800

Who needs that many passwords? (Me, I do)

What happens if I add a **single new word** to one of my dictionary sets? Let's math it up: 60**7** &times; 276 &times; 195 &times; 85 = 2,776,842,900

## 4,574,700 _more_ unique possibilities

And from that unique list, what if the order is random and doesn't matter? Let's `4!` that: 2,776,842,900 &times; 24 = 66,644,229,600

## 109,792,800 _more_ unique possibilities

You can create an extra **109 million** unique strings by adding _a single word_ to a list. Combinatorial and factorial math are just... mind blowing.

Ok, now... someone in infosec tell me: is it a good idea to use Cryptozoologist as a random password generator?

_A very big thank you to my pal [Lucas Willett](https://twitter.com/ltw_) for walking me back through factorial math and, because I am a woman and math is hard let's go shopping, for double checking my math. Wanna talk factorials on a Friday night? Lucas is your guy._