---
title: "Introducing PHP Lib Featuretoggle"
date: 2021-11-25T23:15:57+01:00
draft: false
tags: ["php", "library", "feature-toggle", "laminas", "symfony"]
---
Lately I've been developing a library I call [`lib-featuretoggle`](https://github.com/pascalheidmann/lib-featuretoggle)
in my spare time after experiencing the hassle that implementing feature toggles can be.

# Thoughts on `Feature Toggles`

If you are working on a larger scale project like I do at CHECK24 right now (
btw. [we are constantly hiring](https://jobs.check24.de/hamburg/it)) you will come to a point where you want to develop
new features and test them against your production system without releasing them to your customer base. You might want
to test it with just a portion of your customer base, just internal users or give users the option to opt-in for your
new fancy feature - all are valid and quite common.

These are referred as `feature toggle` and are binary decisions a user or the system on that user's behave makes if a
feature path is available or not. They are not meant to control the execution flow in a permanent manner or decide
between multiple paths!

Implementing each concrete feature toggle can be a tedious task if you have to write your infrastructure from scratch
and will probably be not very easy to extend to support one of those other use cases I mentioned earlier.

# Designing `lib-featuretoggle`

So I set out to create something in my spare time that might even be used in our product sometime which meant it had to
be usable via some kind of `dependency injection` and be easily extended with some kind of adapter. For our product we
use [`laminas`](https://getlaminas.org/) (previously known as Zend Framework) but for reasons both other products at
CHECK24 and for my private projects prefer [`symfony`](https://symfony.com/) therefore my new library has to be 
compatible with least those DI solutions.

I started with a generic feature toggle class that contains a condition and a name for identification. While in my first
naive implementation both condition and key where found on the same class I quickly realized I wanted to be able to
create more complex strategies like "allow our internal users (by IP) to activate a feature toggle while they are logged
in". So I moved conditions into their own, separate and furthermost important recursive structure while keeping the key
on the original feature toggle class.

## Retiring feature toggles on launch day

The next hurdle I wanted to tackle with ease was the situation you easily come across after your feature lands in the
wild after exhaustive testing behind a feature toggle: you have to remove it.

There are several reasons you don't want to release a feature and remove every trace of a feature toggle in the same
process but better wait a bit to let the user test your new feature through and through. I think the most important one
is the flexibility to rollback without reverting your code but the list also includes the chance you remove your feature
toggle and there is still code referencing your feature toggle. Therefore, you want to have some kind of logic in your
feature toggle to return always your new default state no matter what logic your feature toggle previously had.

To tackle this the library uses several layers:

1. FeatureToggleManager
2. FeatureToggleRepository
3. FeatureToggle
4. Conditions

Every layer in that hierarchy has an 1:m relationship to the next layer below it with a First-In-First-Out (FiFo)
solution regarding the `FeatureToggleRepository`. You will have two repositories in your application: the first one
contains a list of all recent feature toggles, but they will always return their default state. The second repository
might use some more advanced ways (like looking up every feature toggle in a database) and to evaluate your feature
based on that result toggle's state, but as FiFo wins the static assignment from first repository will be used.

## Conclusion
I want to stress that feature toggles while sounding like a silver bullet to many problems have their own downsides as
most experienced devs might know:

As they make it easy to flip between different version with a flick (depending on your implementation of course,
but `lib-featuretoggle` makes this a no-brainer with a database repository) devs and PMs tend to build them once but
then forget about them, so they will eventually become a static part of the application. So **always** plan 
sufficient time within your next sprints to remove them in time after releasing your feature as they otherwise 
become kind of feature creep you definitely want to avoid.

Also, they aren't meant by their nature to be used as a way to control the flow of your application. You should always
use testable rules for this kind as feature toggles are meant to be removed and will break your application when misused
(or at least your tests if they are included in them).

Nonetheless, I think they deserve a place in most projects and can ease development by a lot if used correctly.

# Check out `lib-featuretoggle`

If you are using PHP, go and check
out [`lib-featuretoggle` at github](https://github.com/pascalheidmann/lib-featuretoggle).

If you find some kind of bug or use it in your project
and want to enhance it, feel free to open a [pull request](https://github.com/pascalheidmann/lib-featuretoggle/pulls)
or [issue](https://github.com/pascalheidmann/lib-featuretoggle/issues).

# Future outlook
I want to add support for both symfony DI via bundles and laminas via modules / ConfigProvider so this might be the 
next steps. Also, I have some plans for private projects where I will use my lib for sure to get some more real life 
usage which will in turn have influence over some design ideas with `lib-featuretoggle`.

For now the library is written for PHP 7.4+ which are the currently supported version but with the fresh release
of [PHP 8.1](https://stitcher.io/blog/new-in-php-81) this might be pushed to 8.0+ in the future. Its API is not yet
fully finalized (therefore no 1.x release) but I guess there will not many big changes to it anymore. I might experience
more with lazy creation of feature toggles via arrays (see `LazyArrayFeatureToggleRepository`) which right now can 
only support single level conditions instead of real recursion.

In future posts I might present some real usage and enhancements, so feel free to contact me for your usecase of
feature toggles, both in general and especially if you use my lib :)
