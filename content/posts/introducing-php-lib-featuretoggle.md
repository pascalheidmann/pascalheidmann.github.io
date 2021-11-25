---
title: "Introducing PHP Lib Featuretoggle"
date: 2021-11-25T23:15:57+01:00
draft: true
tags: ["php", "library", "feature-toggle", "laminas", "symfony"]
---
Lately I've been developing a library I call "lib-featuretoggle" in my spare time after experiencing the hassle that
implementing feature toggles can be.

# Motivation

If you are working on a larger scale project like I do at CHECK24 right now you will come to a point where you want to
develop new features and test them against your production system without releasing them to your customer base. You 
might want to test it with just a portion of your customer base, just internal users or give users the option to 
opt-in for your new fancy feature - all are valid and quite common.

These are referred as `feature toggle` and are binary decisions a user or the system on that user's behave makes if 
a feature path is available or not. 

Implementing each concrete feature toggle can be a tedious task if you have to write your infrastructure from scratch 
and will probably be not very easy to extend to support one of those other use cases I mentioned earlier.

# Solution: use `lib-featuretoggle`
So I set out to create something in my spare time that might be used in our product sometime which meant it had to 
be usable via some kind of `dependency injection` and be easily extended with some kind of adapter.
For our product we use `laminas` (previously known as Zend Framework) but for reasons both other products at CHECK24 and for my 
private projects prefer `symfony` therefore my new library has to support at least those DI solutions.

I started with a generic feature toggle class that contains a condition and a name for identification. While in my 
first naive implementation both condition and key where found on the same class I quickly realized I wanted to be 
able to create more complex strategies like "allow our internal users (by IP) to activate a feature toggle while they 
are logged in". So I moved conditions into their own, separate and furthermost important recursive structure while 
keeping the key on the original feature toggle class.

## Retiring feature toggles on launch day

The next hurdle I wanted to tackle with ease was the situation you easily come across after your feature lands in 
the wild after exhaustive testing behind a feature toggle: you have to remove it.
There are several reasons you don't want to release a feature and remove every trace of a feature toggle in the same 
process but better wait a bit to let the user test your new feature through and through. I think the most important 
one is the flexibility to rollback without reverting your code but the list also includes the chance you remove your 
feature toggle and there is still code referencing your feature toggle. Therefore, you want to have some kind of 
logic in your feature toggle to return always your new default state no matter what logic your feature toggle 
previously had.

To tackle this the library uses several layers:
1. FeatureToggleManager
2. FeatureToggleRepository
3. FeatureToggle
4. Conditions

Every layer in that hierarchy has an 1:m relationship to the next layer below it with a First-In-First-Out (FiFo) 
solution regarding the `FeatureToggleRepository`. You will have two repositories in your application: the first one 
contains a list of all recent feature toggles, but they will always return their default state. The second 
repository might use some more advanced ways (like looking up every feature toggle in a database) and to evaluate your 
feature based on that result toggle's state, but as FiFo wins the static assignment from first repository will be used.
