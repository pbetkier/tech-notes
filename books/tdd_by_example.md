# Test-Driven Development By Example - Kent Beck

## What is TDD

* TDD is a technique to achieve "clean code that works". 
* Red, green, refactor cycle. You write new code when you have a failing test for it and make it pass ASAP.
* Why having a dummy implementation, just returning a value that is expected in the test, is not considered *done* even though it makes the test pass? Because there is duplication between test data and tested code - duplication has to be eliminated.

TDD is the opposite of BDUF (big design up-front). BDUF may also lead to "clean code that works", but in that case you first make the code clean, then make it work. TDD makes it the other way round: first make it work, then make it clean. It's better, because you don't hunt for bugs at the end of the cycle. Hunting bugs takes enormous amount of time and affects your morale. On the other hand, having code that works stimulates for more.

## Strategies for getting from RED to GREEN

You want to become GREEN and start refactoring as soon as possible. When you are RED you lose the feeling of being safe. There are three main strategies for that:

* Fake it: simply return a dummy value that you expected in the test. Once you are GREEN, work on removing this duplication by replacing it with real logic.
* Obvious implementation: you may provide the implementation right away if it's obvious to you. If you feel you know what you're doing just go for it. If you start getting unexpected test failures, fallback to faking it first.
* Triangulation: generalize the problem to 2 or more examples (more than 1 test case for a single problem). Use it, if you're not sure how to refactor the code after faking it. After the code is refactored, you should remove the redundant tests.

## Refactoring

Refactoring is what you will do a lot in TDD. You will waste a lot of time if your IDE doesn't help you with that. Some ideas:

* Since you have tests, it's ok to start with ugly design, with copy&paste etc. Once you are GREEN you may refactor that. 
* If you don't have tests and want to refactor, then you should add them first. 
* "Accept brief interruptions, never interrupt interruptions".
* Don't be afraid to remove tests if they are no longer relevant after changes to code structure! Each test must be able to justify its existence ;)

## Design

* TDD will help you with designing an API, because you will implement an API that was convienient to use in the test. 
* It will help with designing relations between classes, because you will immediately see that a class is hard to test or has a lot of dependencies.
* Objects that are in the heart of the domain should know a little about the surrounding world, so that they are flexible: testable, reusable, easy to understand.
* TDD makes it possible to continously improve the code quality. The central part of the system should be rock solid, but it's ok if perypherial parts are a bit ugly as long as they are tested. You can always refactor them later.

## Other ideas

* Run your test after writing it and make sure it fails.
* Isolate the aspect that your test case tests. A test should have one business reason to fail.
* Keep a todo list. List all the requirements and corner cases that you plan to test, add refactorings that you plan to perform.
* If you don't know how to write a test, think assert-first.
* Use test data that is easy to follow and understand. Magic numbers are OK as long as they are enclosed within one test case.
* If you're not sure how an external library behaves, it's nothing wrong to write a test for it.
* Write regression tests for bugs that you're about to fix. You need to be sure the fix solved the problem. Also, bugs sometimes return.

