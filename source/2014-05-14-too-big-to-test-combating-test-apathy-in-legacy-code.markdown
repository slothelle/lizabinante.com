---
title: "Too big to test: combating test apathy in legacy code"
category: blog
tags: blog, technical, testing
---

*Originally written for the [Instructure techblog](https://instructure.github.io/blog/2014/05/14/too-big-to-test-combating-test-apathy-in-legacy-code/).*

Writing tests for large Rails apps with lots of dependencies and complicated modeling is, without question, a complete nightmare. We often spend more time wrestling with tests than we do writing code. The end result of writing tests for legacy code is unfortunately predictable: a test suite full of holes, poor coverage, and tests that aren't actually testing the thing you think they are. Bugs begin to pile up, technical debt is avoided like the plague, and quick-fix bandaids are applied instead of addressing the problems head on.

READMORE

We are then faced with a dilemma: our app has become too big and too complicated to test, and we no longer want to write tests for it because it is so painful. This is unavoidable. No matter how many conference talks we attend that promise to teach us how to writing clean, maintainable code, we're still drowning in a bog of bad. The reality isn't that we're bad developers, it's that we don't dictate our workload, deadlines, or priorities. We have to make sacrifices in our code, which isn't a bad thing until it is. So how do we fix it? How do we take away the pain?

The short answer: it's not easy to fix, and it takes time.

<!--more-->

### Prioritize technical test debt

There is no sense in prioritizing technical debt if you have technical test debt. No one wants to write tests for a messy test suite, and you can't fix problems in your code if the tests are contributing to them. Start chipping away at test apathy by improving the test writing and running experience. There are a few ways to quickly improve your testing environment.

**Clean up your factories and spec helpers**

Remove redundant factories or duplicate spec helpers. Helpers with miles of conditionals don't actually *help* anyone. Consider refactoring any method that takes a hash of options to be more semantically named. Search for factory settings or attributes, or helper methods arguments, you often find yourself using and make them the defaults instead of a passed argument.

If you're using a homegrown factory system instead of a gem like [FactoryGirl](https://github.com/thoughtbot/factory_girl), considering refactoring your factories or completely re-writing them. Lots of bad or unused factories can accumulate when you have many developers working on the same code base.

**Using Selenium? Delete your specs**

*... and re-write them*, preferably with a Selenium wrapper, such as [Capybara](https://github.com/jnicklas/capybara). Using a wrapper will also helper clean up your cluttered helper files for these specs.

Selenium specs are a source of constant pain and heartache for developers: they take a long time to run, and they're incredibly complicated. It's a lot of test setup and clicking on stuff for *one* assertion. In lieu of comprehensive Selenium specs that handle all of your edge cases, improve your unit test coverage (on both the client- and server-side), consider writing end-to-end specs for happy paths, and invest in manual testing where people actually click on things. Simplify your Selenium specs and improve your developers' happiness by only testing your happy paths and expected error messages.

**Organize your spec files properly**

Do everyone a favor and fix your nesting. If you're using RSpec, you shouldn't have a `describe` block with a `context` that has two more sets of nested `context`s. If a `describe` or `context` block only has one test in it, consider re-organizing to incorporate it with other tests. If you're writing unit tests, use the `#my_method_name` syntax to group your tests together. Re-write tests that contain multiple assertions: why do you need to assert two things in one test? Assertions that loop through arrays or hashes are sometimes necessary, but they should only be checking one thing, not multiple attributes of your objects.

Lastly, re-organize your file so it is in roughly the same order as your code. The first method in your class shouldn't be the second-to-last thing tested. A little file organization goes a long way when it comes to debugging and adding new tests.

### Use your tools well

A hammer is not a screwdriver, nor is it a crowbar: don't make your developers be the whole toolbox.

**QA Analysts: real humans you need**

Your developers are not your users: they're your developers. While it is reasonable to expect a level of familiarity with your software, developers ultimately are not the ones using it on a day-to-day basis. They are not the experts on your software. You need experts, and they're not your developers.

Automation is all fine and dandy, but you can't automate people: we're unpredictable and kind of stupid. The huge value in manual QAs and regression tests are that you can actually test your code against the human element. Your QAs Analysts are neither unpredictable nor stupid, but they can replicate that element of humans more realistically than automated tests can.

**Regression tests are not a bandaid**

Regression tests aren't just good for finding bugs: they are a tool that can help you spot the holes in your test suite. A bug fix for a regression test shouldn't just be a fix to the code, it should also repair the holes in your tests. Dig through your spec files: is the spec for that bug missing? Is it testing the wrong thing? Is the setup wrong? Fix it.

**Don't monkey patch it**

Use the gems, libraries, and tools as they are actually intended. Your test suite will rapidly spiral out of control (again) if you find yourself ripping apart your tools.

### Maintenance mode

Fixing bad tests and cleaning up ugly factories is all well and good, but what happens long term? Maintaining a test suite is just as important as maintaining code. If you walk away from your test suite after putting in so much effort, it will be all:

![Miley Cyrus: "Don't you every say I just walked away, I will always want you."](http://25.media.tumblr.com/b60af898d2636d792de3089108562c91/tumblr_mt0qdlrZMR1siookko1_500.gif)

Don't wreck your test suite by ignoring it.

**Don't be afraid of repeating yourself**

Sometimes DRY code is bad code, especially if you're creating unnecessary objects or making redundant database calls in your test setup. Clean up the setup that runs before each test, and make sure you're only creating what you need to.

**Abstract after the fact**

In the (paraphrased) [words of Sandi Metz](https://speakerdeck.com/skmetz/all-the-little-things-rubyonales): it is better to repeat yourself than to abstract the wrong thing. You don't have the perspective to abstract the right thing at the very beginning of your refactoring and cleaning process. To ensure you're not recreating the problem you're trying to fix, periodically block off time to review spec files for duplicate code that can be extracted. It takes time and hindsight, which is why reviewing spec files periodically is just as important as refactoring legacy code.

**Remember that tests are code, too**

Consider these two quotes:

> "Ugh, I have to write code. This has robbed me of my will to live."

*No developer, anywhere, ever.*

> "Ugh, I have to write tests. This has robbed me of my will to live."

*All developers, on a daily basis.*

Why do we dread writing tests so much?

A lot of developers approach tests as an after-the-fact thing: the code is done, it's time to write tests.

But really, it should be: the code is done now that I've written my tests.

The virtue of TDD is that you *have* to write tests. You can't write code unless you've written tests. This one factor has contributed largely to its success. I'm not advocating for or against TDD, I'm lobbying for you to consider your tests as an essential part of your code.

If anything, a well-written test just proves that you're right, and who doesn't love being right?