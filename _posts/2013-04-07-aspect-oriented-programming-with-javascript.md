---
layout: default
title: "Aspect Oriented Programming with JavaScript"
date: 2013-04-07
---

# Contents

[Aspect Oriented Programming](#aop)
[JavaScript Library for Aspect Oriented Programming](#js_library_aop)
["before" advice](#before_advice)
["after" advice](#after_advice)
["afterThrowing" advice](#after_throwing_advice)
["afterReturning" advice](#after_returning_advice)
["around" advice](#around_advice)
[Introducing methods](#introductions)
[Library Implementation Details](#js_library_implementation)
[Conclusion](#conclusion)

<a name="aop"></a>

## Aspect Oriented Programming

You are probably already familiar with Object Oriented Programming (OOP) and Functional Programming (FP) paradigms, Aspect Oriented Programming (AOP) is just another programming paradigm. Let's quickly recall what OOP and FP mean and then define what AOP is.

In OOP programs are constructed from objects: encapsulated data is guarded against access by other code and has a few methods attached to it that allow reading and modifying this data.

In FP the primary building block is function. Functions can be passed as arguments to other functions or returned from functions, ideally there is no shared state and each function communicates with other functions just by means of its return values (output) and parameters (inputs).

AOP in its turn is centered around aspects. Aspect is a module of code that can be specified (adviced) as executable at a certain place in the rest of the code (pointcut). A good introduction into AOP for Java is given in the Spring framework [documentation](http://static.springsource.org/spring/docs/2.5.5/reference/aop.html). Here are the definitions of the key terms in AOP:

- **Aspect** - module of code to be executed
- **Advice** - specification when an aspect should be executed
- **Pointcut** - place in code where an advice should be applied

What all those words mean in practice will be more clear from the examples below.

These paradigms are not mutually exclusive, for example, in JavaScript it is possible to use both the Object Oriented and Functional ways of doing things, and usually it is a mix of both. Some languages definitely support one paradigm better than another or do not support some paradigms at all: as an example, Java is primarily an OOP language and Haskell is a FP language.

Ultimately these are just different approaches at how one can structure their programs and they have their strong and weak points in certain situations. OOP is fine when we have a complex domain area with many entities and relations between them and we would like to reflect all those things in the code base so that we can support and evolve our application easier. FP works well when we need to do many computations and can make a good use of its ability to decompose a complex algorithm into simpler reusable parts that are easy to comprehend. AOP is well-suited for cases when we want to introduce some new additional universal behavior, such as logging, transaction or error handling. In that case we would like this additional functionality orthogonal to the core logic of our application to be put into one place and not be scattered across the whole application.

<a name="js_library_aop"></a>

## JavaScript Library for Aspect Oriented Programming

As discussed above JavaScript supports both OOP and FP. It also turns out that it is very easy to add support for AOP to JavaScript by writing a simple library. I was inspired by the article by Danne Lundqvist's [Aspect Oriented Programming and javascript](http://www.dotvoid.com/2005/06/aspect-oriented-programming-and-javascript/) and just decided to implement my own library which will support a few more advices and will provide a different API.

JavaScript allows to easily redefine methods, add properties to objects during execution time, and also functions are objects in JavaScript. As a result a full-fledged Aspect Oriented Framework in JavaScript is only about 150 lines long as you will shortly see. The topic may be a bit advanced for a beginner JavaScript programmer, so it is assumed that the reader at this point has a good handle of prototypes, closures, invocation context, and other advanced features that make programming in JavaScript so much fun. If this sounds like something completely new, please, refer to the [Eloquent JavaScript](http://eloquentjavascript.net/) book by Marijn Haverbeke.

Our library will support the following:

- **Aspect** - just functions, as almost everything in JavaScript is a function
- **Advice** - "before" before method, "after" after method, "afterThrowing" when an exception is thrown, "afterReturning" just before returning a value from a function, "around" at the moment of function execution
- **Pointcut** - "methods" all the methods of an object, "prototypeMethods" all the methods defined on the prototype of an object, "method" only a single method of an object

Let's take a closer look and start from examples.

<a name="before_advice"></a>

## "before" advice

Simple example of using the AOP library:

```javascript
    test("jsAspect.inject: 'before' advice, 'prototypeMethods' pointcut", function() {
        function Object() {
        };

        Object.prototype.method1 = function() {
            return "method1value";
        };

        Object.prototype.method2 = function() {
            return "method2value";
        };

        jsAspect.inject(Object, jsAspect.pointcuts.prototypeMethods, jsAspect.advices.before,
            function beforeCallback() {
                var args = [].slice.call(arguments, 0);

                this.beforeCallbackArgs = this.beforeCallbackArgs || [];
                this.beforeCallbackArgs.push(args);
            }
        );

        var obj = new Object();

        equal(obj.method1("arg1", "arg2"), "method1value", "method1 was called as expected and returned the correct value");
        equal(obj.method2("arg3", "arg4", "arg5"), "method2value", "method2 was called as expected and returned the correct value");
        deepEqual(obj.beforeCallbackArgs, [["arg1", "arg2"], ["arg3", "arg4", "arg5"]], "before callback was called as expected with correct 'this'");
    });
```

In order to advice application of the aspect **beforeCallback** before invocation of each method defined on the prototype of **Object** we call **jsAspect.inject** with appropriate arguments:

```javascript
    jsAspect.inject(Object, jsAspect.pointcuts.prototypeMethods, jsAspect.advices.before,
        function beforeCallback() {
            var args = [].slice.call(arguments, 0);

            this.beforeCallbackArgs = this.beforeCallbackArgs || [];
            this.beforeCallbackArgs.push(args);
        }
    );
```

As a result the function **beforeCallback** is executed before each method with that method's arguments each time it is invoked. In the callback function **this** refers to the object on which the original method was called. In this example we just check if there is an array of arguments defined on that object. If there is no array we create it and then record all the current arguments there. The test above does not actually verify that the execution happens before the method but it is also easy to check, this can be a simple exercise for the reader.

<a name="after_advice"></a>

## "after" advice

"after" advice is quite similar to "before", the difference is only in one argument to **jsAspect.inject** and the fact that the aspect will actually be executed after the original method not before it:

```javascript
    jsAspect.inject(Object, jsAspect.pointcuts.prototypeMethods, jsAspect.advices.after,
        function afterCallback() {
            var args = [].slice.call(arguments, 0);

            this.afterCallbackArgs = this.afterCallbackArgs || [];
            this.afterCallbackArgs.push(args);
        }
    );
```

<a name="after_throwing_advice"></a>

## "afterThrowing" advice

This advice is executed when an exception occurs in the original method. Then this exception is first passed to the aspect as an argument and is still rethrown later in the original method. Example:

```javascript
    test("jsAspect.inject: 'afterThrowing' several aspects", function() {
        function Object() {
        };

        Object.prototype.method1 = function() {
            throw new Error("method1exception");
        };

        Object.prototype.method2 = function() {
            throw new Error("method2exception");
        };

        jsAspect.inject(Object, jsAspect.pointcuts.prototypeMethods, jsAspect.advices.afterThrowing,
            function afterThrowingCallback(exception) {
                exception.message = exception.message + "_aspect1"
            }
        );
        jsAspect.inject(Object, jsAspect.pointcuts.prototypeMethods, jsAspect.advices.afterThrowing,
            function afterThrowingCallback(exception) {
                exception.message = exception.message + "_aspect2"
            }
        );

        var obj = new Object();
        var thrownExceptions = [];

        ["method1", "method2"].forEach(function (methodName) {
            try {
                obj[methodName]();
            } catch (exception) {
                thrownExceptions.push(exception.message);
            }
        });

        deepEqual(thrownExceptions, ["method1exception_aspect2_aspect1", "method2exception_aspect2_aspect1"], "Multiple aspects are applied");
    });
```

Also we see here that a few aspects can be applied for the same advice. In fact this is true not only for "afterThrowing" but for types of supported advices. In the aspect we just append a suffix to the exception message and then verify that the exceptions are still rethrown from the original methods with the modified as expected messages.

<a name="after_returning_advice"></a>

## "afterReturning" advice

This advice is applied when the original function is about to return its value. Then this value is passed as an argument to the aspect and the actual return value will be whatever the aspect decides to return. A few aspects can be applied at the same time as well:

```javascript
    test("jsAspect.inject: several 'afterReturning' aspects", function() {
        function Object() {
        };

        Object.prototype.identity = function(value) {
            return value;
        };

        ["aspect1", "aspect2", "aspect3"].forEach(function (aspectName) {
            jsAspect.inject(Object, jsAspect.pointcuts.prototypeMethods, jsAspect.advices.afterReturning,
                function afterReturningCallback(retValue) {
                    return retValue + "_" + aspectName;
                }
            );
        });

        equal(new Object().identity("value"), "value_aspect3_aspect2_aspect1", "'afterReturning' several aspects applied in the reverse order");
    });
```

In this example we create several named aspects and in each of them append the name of the aspect to the return value.

<a name="around_advice"></a>

## "around" advice

The most interesting advice is "around". Here aspect receives several arguments: the original function to which the aspect was applied (can be actually just another aspect, but this is entirely hidden from the current aspect) and the arguments of the original function. Then the return value of the aspect is returned from the original function:

```javascript
    test("jsAspect.inject: 'around' advice, 'prototypeMethods' pointcut", function() {
        function Object() {
        };

        Object.prototype.identity = function(x) {
            return x;
        };

        Object.prototype.minusOne = function(x) {
            return x - 1;
        };

        jsAspect.inject(Object, jsAspect.pointcuts.prototypeMethods, jsAspect.advices.around,
            function aroundCallback(func, x) {
                return 2 * func(x);
            }
        );

        var obj = new Object();

        equal(obj.identity(3), 6, "'around' advice has been applied to 'identity'");
        equal(obj.minusOne(3), 4, "'around' advice has been applied to 'minusOne'");
    });
```

<a name="introductions"></a>

## Introducing methods

We can also easily add methods to existing objects like this:

```javascript
    test("jsAspect.introduce: 'methods' pointcut", function() {
        function Object() {
            this.field1 = "field1value";
            this.field2 = "field2value";
        };

        Object.prototype.method1 = function () {
            return "valuefrommethod1";
        };

        jsAspect.introduce(Object, jsAspect.pointcuts.methods, {
            field3: "field3value",
            staticmethod1: function () {
                return "valuefromstaticmethod1";
            }
        });

        equal(Object.field3, "field3value", "Object.prototype.field3");
        equal(Object.staticmethod1 ? Object.staticmethod1() : "", "valuefromstaticmethod1", "Object.staticmethod1");
    });
```

Another pointcut **jsAspect.pointcuts.methods** is used here so that we introduce methods and fields not to the object's prototype but to the object directly.

<a name="js_library_implementation"></a>

## Library Implementation Details

Now it is time for the implementation details. The basic idea is very simple: we will redefine each method that is matched by the provided pointcut. The redefined method will call the original method after executing the advised aspects at appropriate points in time. For the end user there will be no difference with the original method in how the method should be called.

But let's first start with something a bit simpler, the **introduce** method that adds new methods and fields to an object:

```javascript
    jsAspect.introduce = function (target, pointcut, introduction) {
        target = (jsAspect.pointcuts.prototypeMethods == pointcut) ? target.prototype : target;
        for (var property in introduction) {
            if (introduction.hasOwnProperty(property)) {
                target[property] = introduction[property];
            }
        }
    };
```

First an appropriate **target** is determined and then we copy each own property in **introduction** to this **target**. It is that easy in JavaScript, no proxies, no additional objects, just plainly defining methods on the original object.

The implementation of the main method we have seen in the examples so far **jsAspect.inject**:

```javascript
    jsAspect.inject = function (target, pointcut, adviceName, advice, methodName) {
         if (jsAspect.pointcuts.method == pointcut) {
             injectAdvice(target, methodName, advice, adviceName);
         } else {
             target = (jsAspect.pointcuts.prototypeMethods == pointcut) ? target.prototype : target;
             for (var method in target) {
                 if (target.hasOwnProperty(method)) {
                     injectAdvice(target, method, advice, adviceName);
                 }
             };
         };
    };
```

Again, we compute the correct target and then for each method in the target we inject a new advice as specified by the provided **adviceName**. Next the function **injectAdvice**:

```javascript
    function injectAdvice(target, methodName, advice, adviceName) {
        if (isFunction(target[methodName])) {
            if (jsAspect.advices.around == adviceName) {
                 advice = wrapAroundAdvice(advice);
            };
            if (!target[methodName][adviceEnhancedFlagName]) {
                enhanceWithAdvices(target, methodName);
                target[methodName][adviceEnhancedFlagName] = true;
            };
            target[methodName][adviceName].unshift(advice);
        }
    };
```

If the target method in the object for some reason is not actually a function (just a usual value field instead), then we do nothing. Otherwise we check whether the method has already been enhanced to support advices. If it has not yet been enhanced, we do enhance it, otherwise we simply add a new advice to an additional system field created on the original object by our framework. This field contains an array of all the advices that should be applied for a given **adviceName**. Also we can see that the "around" advice requires some additional handling, here we just wrap it with **wrapAroundAdvice** before actually adding it onto the system field on the original method. It will become a bit clearer why we do that after we consider the implementations of the functions **enhanceWithAdvices** and **wrapAroundAdvice**.

Next, the function **enhanceWithAdvices**:

```javascript
    function enhanceWithAdvices(target, methodName) {
        var originalMethod = target[methodName];

        target[methodName] = function() {
            var self = this,
                method = target[methodName],
                args = [].slice.call(arguments, 0),
                returnValue = undefined;

            applyBeforeAdvices(self, method, args);
            try {
                returnValue = applyAroundAdvices(self, method, args);
            } catch (exception) {
                applyAfterThrowingAdvices(self, method, exception);
                throw exception;
            };
            applyAfterAdvices(self, method, args);
            return applyAfterReturningAdvices(self, method, returnValue);
        };
        allAdvices.forEach(function (advice) {
            target[methodName][jsAspect.advices[advice]] = [];
        });
        target[methodName][jsAspect.advices.around].unshift(wrapAroundAdvice(originalMethod));
    };
```

This function is really the core of the whole framework and implements the main idea behind it. Let's look into the details of what happens here. First we store the original method in the variable **originalMethod** and then redefine it. In the redefined method we first apply the "before" advises before the original method, then "around" advises at the time of execution of the original method, then try to catch an exception and if we do catch an exception, then we re-throw it after calling advises registered to handle exceptions with "afterThrowing" advice. Then "after" advises are applied and then "afterReturing" advises. Pretty transparent and easy, just what one would expect to find in the implementation.

Then we create additional system fields on the original method to store the arrays of advises for each of the possible advice names. Also, at the end we push a wrapped original method as the first advice to the array of "around" advises in order not to deal with border conditions in other functions that will follow below.

Now let's look at the advice application methods:

```javascript
    function applyBeforeAdvices(context, method, args) {
        var beforeAdvices = method[jsAspect.advices.before];

        beforeAdvices.forEach(function (advice) {
            advice.apply(context, args);
        });
    };

    function applyAroundAdvices(context, method, args) {
        var aroundAdvices = method[jsAspect.advices.around]
                .slice(0, method[jsAspect.advices.around].length),
            firstAroundAdvice = aroundAdvices.shift(),
            argsForAroundAdvicesChain = args.slice();

        argsForAroundAdvicesChain.unshift(aroundAdvices);
        return firstAroundAdvice.apply(context, argsForAroundAdvicesChain);
    };

    function applyAfterThrowingAdvices(context, method, exception) {
        var afterThrowingAdvices = method[jsAspect.advices.afterThrowing];

        afterThrowingAdvices.forEach(function (advice) {
            advice.call(context, exception);
        });
    };

    function applyAfterAdvices(context, method, args) {
        var afterAdvices = method[jsAspect.advices.after];

        afterAdvices.forEach(function (advice) {
            advice.apply(context, args);
        });
    };

    function applyAfterReturningAdvices(context, method, returnValue) {
        var afterReturningAdvices = method[jsAspect.advices.afterReturning];

        return afterReturningAdvices.reduce(function (acc, current) {
            return current(acc);
        }, returnValue);
    };
```

The implementation is again pretty straightforward. We use **forEach** to iterate over the "before", "after" and "afterThrowing advices", and **reduce** to apply the "afterReturing" advices in a sequence.

"around" advices are handled a bit differently: we take the first advice (which as you remember was "wrapped" earlier) and pass to it the arguments of the original method together with an array of all the "around" advices as the first argument. For what happens next when executing the first "wrapped" advice we have to look at the implementation of the **wrapAroundAdvice** method:

```javascript
    function wrapAroundAdvice(advice) {
        var oldAdvice = advice,
            wrappedAdvice = function (leftAroundAdvices) {
                var oThis = this,
                    nextWrappedAdvice = leftAroundAdvices.shift(),
                    args = [].slice.call(arguments, 1);

                if (nextWrappedAdvice) {
                    var nextUnwrappedAdvice = function() {
                        var argsForWrapped = [].slice.call(arguments, 0);

                        argsForWrapped.unshift(leftAroundAdvices);
                        return nextWrappedAdvice.apply(oThis, argsForWrapped);
                    };
                    args.unshift(nextUnwrappedAdvice);
                };
                return oldAdvice.apply(this, args);
            };

        //Can be useful for debugging
        wrappedAdvice.__originalAdvice = oldAdvice;
        return wrappedAdvice;
    };
```

When invoking a "wrapped" advice we check if there are any next "around" advises left (the last one is the original method if you remember). If there are no advises we deal with the original method which we just invoke by passing to it the arguments provided by user at the point of the invocation of the method enhanced with aspects. Otherwise we "unwrap" the next available "around" advice. The essence of "unwrapping" is that we remove the extra argument containing the array of all the remaining "around" advises. Then we pass this "unwrapped" advice to the current advice as the first argument
so that each "around" advice (except for the original method) has one argument more than the original method: on the first place goes the next advice to be applied or the original method. Then execution of the current advice can trigger
execution of the next available "around" advice (first argument) and we will again recursively go to the function **wrapAroundAdvice**.

Arguably **wrapAroundAdvice** is the most complicated piece of the framework. It looks like there are alternatives to emulating stack of calls in this way, for example, we can redefine the function each time a new "around" advice is added. But then we would have to copy all the advises bound to the original method to the new method and this new redefined method will have to have a structure like the method we create in **enhanceWithAdvices**, and then things get again a bit complicated when we want to add another "around" advice as we would not like to execute each "before" advice more than once, we will have to clean-up the originally added advises, etc. So the present implementation seemed like a reasonably simple although it does require some mental effort to understand.

<a name="conclusion"></a>

## Conclusion

We demonstrated that it is quite easy to implement a full-fledged AOP framework in JavaScript due to the dynamic nature of the language and standard extensibility possibilities it provides. The created framework implements some of the functionality (advises) of the Java based [Spring framework](http://static.springsource.org/spring/docs/2.5.5/reference/aop.html).

Now before you go and use AOP techniques in your code like the ones we used here I feel that a few words of caution should be added. AOP is a quite powerful tool and can decouple code modules well from each other, but the problem is that quite often this decoupling is excessive and introduces additional complexity. As a result it may become virtually impossible to trace what causes what to execute in your code and it can bring a lot of headache when later supporting this code. It really pays off to always try to solve a particular problem using the traditional programming paradigms like OOP and FP and only to resort to AOP when it is really needed. For example, it is much better to introduce logging in one place of your application with one aspect than to add repetitive logging calls all over the place, AOP is of a great use here. That is, knowing some paradigm does not mean that it is always well-suited for solving your problem. This is almost the same as with design patterns, remember the main goal is to write an application fast and easy so that it is maintainable rather than making your code look "smart". Abusing both design patterns and AOP is certainly discouraged. Following this advice (no pun intended) will save a lot of time for people who will be supporting your code in the future.

And finally, the full implementation of the library can be found here: [jsAspect: Aspect Oriented Framework for JavaScript](https://github.com/antivanov/jsAspect)

## Links

[Article "Aspect Oriented Programming and JavaScript"](http://www.dotvoid.com/2005/06/aspect-oriented-programming-and-javascript/ "Aspect Oriented Programming and JavaScript")
[Aspect Oriented Programming with Spring (Java)](http://static.springsource.org/spring/docs/2.5.5/reference/aop.html "Aspect Oriented Programming with Spring")
