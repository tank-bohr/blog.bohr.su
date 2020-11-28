---
layout: post
title:  "How to write tests"
date:   2020-11-28 21:35:00 +0300
---

Hello! Captain Obvious is in touch, and today I will tell you how to write tests.

I believe that nobody should write software without tests. Writing without tests is crazy! Not writing tests is like writing programs without version control. Today you will not meet such madmen. But there are still quite a few developers who don't write tests. So let's figure it out. Why doesn't everyone write tests yet? Personally, I see two reasons, and now I will tell you more about them.

## Reason 1. Writing tests takes time and slows down development

No, it doesn't slow down! Tests speed up development. Imagine you are developing a feature that requires a few changes. And after every change, you need to check the entire application to make sure you haven't broken anything. After **every** change, you need to check the **entire** application. But if you have tests, then check the entire application will take seconds! This more than covers the time spent developing the tests themselves. The test is written once, but it is useful as long as your application's feature lives.

## Reason 2. We do not know how to write tests because we never did it

In my opinion, this is the main reason. And now I will try to help such developers and tell you how easy it is to write tests. Writing tests is very simple, and now I will tell you a universal test writing algorithm.

- We take the function that we want to test
- Call this function
- Checking the result and/or side effect
- PROFIT!!

## Test structure

The test has three parts. Consists of three A's - [AAA](http://wiki.c2.com/?ArrangeActAssert)

- *Arrange* - preparation stage. Here we prepare the required variables for the function call. and set up the environment in which the function under test will be called
- *Act* - here, we call the function under test or, in general, we perform any action that we want to test.
- *Assert* - checking the result. here we make sure that the result is the same as expected

## Testing for side effects

There are two cases.

- *The function under test is pure.* This is the most straightforward case to test. It is necessary to endeavor for most functions to be like this. Testing them is incredibly easy. We just compare the result with the expected one.
- *The function being tested is impure.* This function is much more difficult to test. As a rule, in such cases, side effects are checked.

The most common side effect is I/O. Such effects can be tested in different ways. For example, a function writes to a database. You can create an empty database and, after the function has been completed, make a request to the database and check that the write was successful

Another example, a function that makes an http-call to some http-api. We can run a [web server](github.com/tank-bohr/bookish_spork) with predefined responses and check which requests our system produced.

This approach is called integration testing because we test the integration of several components (application-database, application-external web service). If a test tests several subsystems interaction within one application, such a test can also be considered an integration test.

But there is also another approach. For example, a [mock-object](https://en.wikipedia.org/wiki/Mock_object) is used instead of a database. Such a test is much easier to write. It will be much faster and will test the component in isolation from others. This test is called a Unit Test. This approach, combined with the [DI](https://en.wikipedia.org/wiki/Dependency_injection) technique, allows you to check for any side effects. For OOP languages, it looks something like this

- we create a special mock-object and in tests
- we inject it into the tested object (outside the tests, we will inject the real object).
- calling the testee method
- check that a specific method has been called on the mock-object.

A mock-object that can be checked for called methods is called Spy. Spy is a special mock object that tracks all the calls on itself, and thanks to this allows you to check later which methods were called on this object.


## Integration tests vs. Unit tests

There are two crucial characteristics of tests: sensitivity and fragility. Sensitivity shows how well a test is able to catch a program error. Fragility determines the degree of test false-positives when you change the code in one place, and 60 tests break at once. And you have to fix them all, even though there was only one change in the code. Tests should be sensitive but not fragile. The problem is, the more sensitive the tests are, the more fragile they become. And vice versa. Therefore, we must seek a delicate balance.
Kent Beck says, "[Programmer tests should be sensitive to behavior changes and insensitive to structure changes.](https://medium.com/@kentbeck_7670/programmer-test-principles-d01c064d7934#70f8)". The interesting thing is it's just a characteristic of integration tests. They are insensitive to changes in structure but sensitive to changes in behavior. While unit tests are the opposite. As a consequence, unit tests are pretty bad at solving the problem of finding regressions.

Nevertheless, unit tests still have several advantages.

- they are much easier to write (since the Arrange stage is much easier)
- they are faster because there are no heavy dependencies
- [design pressure](https://www.amazon.com/Growing-Object-Oriented-Software-Guided-Tests/dp/0321503627`). Tests are the code that calls your code. The better the code is written, the easier it is to test it, and the worse your code is, the harder it is to write unit tests. And the complexity of writing unit tests is a clear - symptom that your software has design problems.
- allow you to avoid the combinatorial explosion of states that integration tests suffer.

Suppose you have two components, each of which can be in 4 states. To test all states with unit tests, you need 8 (4 + 4) tests. But to test all states with integration tests, you need 4 x 4 = 16 tests. And the more the number of states the components have, the more significant this difference will be. This is what I call the combinatorial explosion.

The classic approach is a [test pyramid](https://martinfowler.com/bliki/TestPyramid.html). A lot of cheap unit tests and a little expensive integration tests. But recently, another approach has been more popular: unit tests are not needed at all since they have very little practical use. The most famous proponent of this approach is [DHH](https://dhh.dk/2014/tdd-is-dead-long-live-testing.html). I bring to your attention a [hangouts discussion](https://www.youtube.com/watch?v=z9quxZsLcfo) between Kent Beck and DHH on the topic.

## Bonus. Advanced types of automated testing

**Property-based testing.** It is a technique that focuses on checking property but not a single example. It's not easy to write such tests; in some cases, it is even impossible. I encourage you to read the book [Property-Based Testing with PropEr, Erlang, and Elixir](https://propertesting.com/). It describes the main approaches to property-based testing.They are also relevant for other languages

**Mutation testing**. If you are lucky and work in the [java](http://pitest.org/) or [ruby](https://github.com/mbj/mutant) ​​ecosystem, you have access to mature frameworks for mutation testing. Mutation tests are tests for your tests. They change the code randomly (introducing so called mutations), and tests should fail from this. Surviving mutants show where your tests are not sensitive enough.

**[Fuzzing](https://en.wikipedia.org/wiki/Fuzzing)** - The technique is straightforward. A random (or not) stream of garbage data is fed to the program input until the program crashes. It helps to increase the stability of your software dramatically.


### Related links

- video [The Magic Tricks of Testing](https://www.youtube.com/watch?v=URSWYvyc42M) by Sandi Metz
- book [Growing Object-Oriented Software, Guided by Tests](https://www.amazon.com/Growing-Object-Oriented-Software-Guided-Tests/dp/0321503627) by Steve Freeman and Nat Pryce
- book [Test Driven Development: By Example](https://www.amazon.com/Test-Driven-Development-Kent-Beck/dp/0321146530) by Kent Beck
- book [PropEr Testing](https://propertesting.com/) by Fred Hebert
