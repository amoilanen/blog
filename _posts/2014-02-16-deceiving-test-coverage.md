---
layout: default
title: "Deceiving Test Coverage"
date: 2014-02-16
---

# Contents

[What's your test coverage?](#introduction)
[Why we test](#testingcode)
[Quality of tests](#qualityoftests)
[Test coverage](#testcoverage)
[Summary](#summary)

<a name="introduction"></a>

## What's your test coverage?

If you are a developer and write tests probably you already heard people around you talking about test coverage a lot. Usually it is something like: "we do not have this module covered yet", "we have 90% test coverage", "this new class is covered 100%". It is implied that the higher the test coverage is, the better it is: we are more confident in the code, there are fewer bugs, and the code is well-documented. And you may even get asked from time to time by other developers or wonder yourself what is the test coverage for the code you are currently working on.

Test coverage then seems like a pretty important metric that describes how well our code is tested. At the same time we use this term so often that inevitably we gloss over some important details and can even forget what we are trying to achieve with test coverage to begin with.

This article tries to revisit some of the reasoning behind the test coverage metric and provide a few simple examples that illustrate how we can use this metric and why we should better understand what it really means.

Hopefully, after reading this article you will be a bit more sceptical when somebody claims that the code is "100% tested", "everything is covered", "we should try to reach 100% coverage here", etc. Moreover, you will be able to offer some good arguments and maybe convince others to also be more practical and reasonable when it comes to this metric. And also you will be hopefully having much more meaningful discussions about the quality of the code than just:

- "What's our test coverage?"
- "It is 100%"
- "Good, it should definitely work"

<a name="testingcode"></a>

## Why we test

Probably, you were already doing some testing and the topic is quite familiar to you, but still, let's start again from the very beginning in small baby steps in order to better understand how exactly do we come up with such a metric as test coverage.

We test our code because we want to be sure, at least to some degree, that it works. Every software developer knows that a just written piece of code rarely works as expected when it is first run. Getting from the first version of the code to the working one can involve a few cycles of applying some changes and observing the results in the working code, that is testing the code. If everything is good, than the code is tested and works in the end.

Well, almost, because it turns out that due to the subjectivity of testing by the same person(s) who develop the code they can sometime miss a few errors (known as "bugs"). That's where testers come in and try to catch those bugs in the code which developers inadvertently left there.

If the testers miss bugs, then it is the turn of users to catch errors in software. Hopefully the users get well-tested and high-quality software, but, as we all know, unfortunately all too often the software we use still has quite a few bugs.

As we want our code to work correctly, ideally we would like to have no bugs in it when we make it available to the users. But this turns out to be a quite challenging task.

For some areas, the cost of error is so high and prohibiting, that finding all the errors and fixing them becomes a must. A good example of this can be software that is installed on a space ship that can cost a lot of money and requires everything to work precisely and perfectly.

For other applications, such as, for example, a video hosting service, the errors are still undesirable, but cost less.

It is all about making a trade-off: more testing means more effort, the cost of the left bugs should be less than the cost of additional testing required to find them.

Another approach is formally proving that some code works and implements specification which in its turn does not have bugs in it to begin with. Testing is quite different. **Tests give us confidence, but do not prove completely that the code works**, and this is the first important point we want to make in this article and that is often missed by some developers. There is a separate field in Computer Science that focuses on formally proving that programs are correct and some of its methods are even applied practically. We will not be talking about formally proving programs here, but for more information and additional links feel free to start with the following Wikipedia article ["Formal verification"](http://en.wikipedia.org/wiki/Formal_verification) in case you have interest.

Often tests are written in the form of executable specifications that can be run by computer which saves a lot of precious human time from doing tedious, repetitive and error-prone work and leaves it to machines that are well-suited precisely for such things.

If tests are written together with the code and before the actual code is written we are talking about [test-driven development](http://en.wikipedia.org/wiki/Test-driven_development), or test-first development. This has become quite a popular technique in the last years, but still many people write tests after the code has already been written or, unfortunately, do not write any tests at all. Guess why? Yes, "there is no time for that", although, surprisingly, it is somehow always possible to later find quite a lot of time to fix all those numerous issues that were missed because of poor testing.

Some articles and practical experience suggests that writing tests for the code, ideally doing test-driven development, in the end saves time and development effort by reducing the number of defects dramatically. Although some bugs still escape and need to be caught by QA later, their number is then not as large as it would have been without tests.

So writing tests and testing code well pays off and makes sense because of the better quality of the resulting software, less stress for developers, more satisfied users and less development effort required to develop and fix the issues.

<a name="qualityoftests"></a>

## Quality of tests

Tests are good, as we just have seen and we need them. But there are so many ways to write tests for some piece of code, then how do we know if some set of tests is better than another? Can we somehow measure the quality of our tests and compare different sets of tests with each other?

Good tests are the ones after running which we are quite sure that the code works and the major usage scenarios have been tested or we say "covered". Good tests should not be more complicated and take more effort to support than the code they test. They should be as independent as possible and allow to trace down a bug from a test failure in a matter of seconds without long debugging sessions. There should not be a lot of duplication, the test code should be well-structured. The tests should be fast and lightweight. etc.

We see that there are quite a few metrics and criteria that we can come up with that will tell us whether our tests are good enough.

<a name="testcoverage"></a>

## Test coverage

Test coverage is just one such simple metric, although probably the most popular one, that allows us to judge whether our "tests are good". Quite often, and, as we are already starting to see, a bit simplistically, people make this synonymous to "test coverage is good". Moreover, "test coverage" often means the "line coverage of code by tests" which is quite different from, for example, "scenario coverage by tests" as it can be observed in the following simple example.

The example will be in JavaScript and will use a particular testing framework [Jasmine](http://jasmine.github.io/2.0/introduction.html) but is not really JavaScript-specific. We just use JavaScript here to make our example real and executable in some environment, and you are free to port it to your favorite language and testing framework as you wish.

Let's say we want to write a simple function that will return values from some dictionary based on the set of keys that are passed to it.

```javascript
  var dictionary = {
    "key1": "value1",
    "key2": "value2",
    "key3": "value3"
  };
```

Let's write some tests first that specify the desired behavior:

```javascript
  describe("getValues - full line coverage", function() {

    it("should provide values for requested keys", function() {
      expect(getValues(["key1", "key2", "key3"]))
          .toEqual(["value1", "value2", "value3"]);
    });
  });
```

The implementation is pretty simple:

```javascript
  function getValues(keys) {
    return keys.map(function(key) {
      return dictionary[key];
    });
  }
```

We just convert a list of keys into the list of values with **map** by replacing each key with a corresponding value from **dictionary**.

As we can see, each line of our function **getValues** is executed once and we have the so longed for 100% test coverage. Good job?

Well, let's pause for a moment and look at an alternative implementation that will make our test pass and will have the same 100% test coverage.

```javascript
  function getValues(keys) {
    return ["value1", "value2", "value3"];
  }
```

The implementation is obviously not what we want, because it always returns the same values no matter which keys we pass to the function. Something obviously went wrong with our test coverage metric here because **it gave us false confidence in a broken code**. And also then our tests are still not good enough as we did not recognize the wrong implementation. Let's add more tests and be more specific for what behavior we wish.

```javascript
  describe("getValues", function() {

    it("should provide values for requested keys", function() {
      expect(getValues(["key1", "key2", "key3"]))
          .toEqual(["value1", "value2", "value3"]);
    });

    it("should provide values for single key", function() {
      expect(getValues(["key1"])).toEqual(["value1"]);
    });
  });
```

Now we have failing test (and 100% test coverage at the same time ;)) and our implementation should definitely be fixed.

But, if we are really stubborn or have no clue how to implement this required function we can fix it like this.

```javascript
  function getValues(keys) {
    return (keys.length == 1) ? ["value1"]
        : ["value1", "value2", "value3"];
  }
```

which is still wrong, although we already have two passing tests and our test coverage is still 100%.

Then we have no choice but to test all the possible combinations of the keys present in the dictionary.

```javascript
  describe("getValues", function() {

    sets(["key1", "key2", "key3"]).forEach(function(testInput) {
      var testCaseName = "should provide values for '"
          + testInput.join(",") + "'";

      it(testCaseName, function() {
        var expectedOutput = testInput.map(function(key) {
          return key.replace("key", "value");
        });
        expect(getValues(testInput)).toEqual(expectedOutput);
      });
    });
  });
```

Here we are using some function **sets** that generates all the possible sets from the given arguments. Its [implementation](https://github.com/antivanov/misc/blob/master/JavaScript/TestCoverage/generator.js) is left out of scope of this article.

It looks like our test suite is now more complicated then the code we test. It has 8 test cases which all pass and provides 100% test coverage. But now it seems that the only valid implementation to make all those tests pass is the original one.

```javascript
  function getValues(keys) {
    return keys.map(function(key) {
      return dictionary[key];
    });
  }
```

OK. So, are we finished yet?

Well, not quite. There are still many scenarios for our code that were not covered but which we may in fact expect to occur when our code is used.

What if we provide some unexpected input or a key that is not present in the dictionary? What should the behavior be? Let's add a couple more tests.

```javascript
  describe("getValues", function() {

    sets(["key1", "key2", "key3"]).forEach(function(testInput) {
      var testCaseName = "should provide values for '"
          + testInput.join(",") + "'";

      it(testCaseName, function() {
        var expectedOutput = testInput.map(function(key) {
          return key.replace("key", "value");
        });
        expect(getValues(testInput)).toEqual(expectedOutput);
      });
    });

    it("should handle invalid input properly", function() {
      expect(getValues(null)).toEqual([]);
    });

    it("should return no value for unknown key", function() {
      expect(getValues(["key1", "unknown", "key3"]))
          .toEqual(["value1", undefined, "value3"]);
    });
  });
```

Unfortunately our implementation does not pass the test for invalid input and we should correct it a bit. Let's do it.

```javascript
  function getValues(keys) {
    return keys ? keys.map(function(key) {
      return dictionary[key];
    }) : [];
  }
```

OK, now we have 10 test cases and still 100% test coverage which, by the way, it seems, we had all along in this example. Should we stop at this point?

If we look formally, there are still lots of untested scenarios, for example, what if **dictionary** contains other keys or no keys at all, will our code work? So we can get really paranoid about our code, do not believe that it works and add more and more tests. But, rather than wasting time and doing that, let's just say that now **it just works** because we tested basic scenarios well and even some invalid inputs.

Yes, that's right, as much as our tests give us confidence, without strict mathematical proof of correctness at some point we should just say "we believe that it works now". This is just how it is, tests are a tool not a means. When our tool gives us enough confidence and lets us explore the code enough in the end it is still our judgement whether the code works or not. By the way, this is precisely why our code still needs to be tested or looked at by other people afterwards. As good as we hope it is, our judgement may be flawed and we can miss some important scenarios in which our code will be used.

Given this simple example, let's move on directly to the final part of this article and formulate some of the conclusions that now should be more or less evident.

<a name="summary"></a>

## Summary

- Test coverage is just one of many metrics that can be used to judge how good our tests are
- Name "test coverage" is misleading, it should be more like "line coverage"
- Line coverage is a simplistic metric that measures not the percentage of possible scenarios that were covered, but rather that of executed lines of code
- Line coverage is being measured primarily because it is easy to measure
- Reaching 100% line coverage does not guarantee that code works
- For strictly proving that code is correct a whole different approach is needed
- Don't be deceived by nice line coverage metric values
- Spending a lot of effort on reaching 100% line coverage may not be worth it, because scenario coverage may still be well below 100%
- Consider other metrics in combination with line coverage
- Computing scenario coverage is more difficult than line coverage
- Tests do not prove that code works, only give more confidence
- Avoid excessive confidence in tests
- Quality of the code depends on the quality of tests
- Automated testing and TDD are great tools that improve the quality of software dramatically
- It is difficult or not feasible to cover all possible scenarios for code
- Some leap of faith is still needed that the code works
- There is no exact recipe when we have written enough tests, not too many not too few
- Avoid being too speculative about code and adding too many extra tests, added value per test will be less
- Thoroughly testing invalid inputs is fine for publicly exposed APIs, but may make little sense for internal library code, but, again, no exact recipe

## Links

[Code from the article](https://github.com/antivanov/misc/blob/master/JavaScript/TestCoverage)
[Formal Verification](http://en.wikipedia.org/wiki/Formal_verification)
[Test-Driven Development](http://en.wikipedia.org/wiki/Test-driven_development)
[Jasmine (JavaScript testing framework)](http://jasmine.github.io/2.0/introduction.html)
