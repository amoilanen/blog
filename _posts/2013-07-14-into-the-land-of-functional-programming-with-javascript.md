---
layout: default
title: "Into the Land of Functional Programming with JavaScript"
date: 2013-07-14
---

# Contents

[Enter JavaScript...](#enter_javascript)
[Running examples](#running_examples)
[Functions are "first-class citizens"](#functions_first_class_citizens)
[Closures and scopes](#closures_and_scopes)
[Partial Application](#partial_application)
[Memoization](#memoization)
[Lazy evaluation](#lazy_evaluation)
[Not a functional language but](#not_a_functional_language)

<a name="enter_javascript"></a>

## Enter JavaScript...

JavaScript is an interesting language. Syntactically it resembles C, C++ and Java, but it was also inspired by a functional language: Scheme, which is a dialect of Lisp. That JavaScript has a C-like syntax is largely due to the hype around Java at the time JavaScript was introduced. The language was rushed to the market, leaving in it a few bad design decisions like global variables, 'with' construct, etc. The name of the language itself is a bit misleading: it has nothing to do with Java except for a slight syntactic similarity.

JavaScript still has somewhat bad reputation with some people who do not know the language well, but encountered a few problems when programming for the browser or heard some unfavorable opinions from the people who did so. The language itself, however, had nothing to do with the inconsistent client-side API implementations in different browsers. By the way, there are plenty of client-side libraries such as JQuery that hide the browser differences behind their API and the developer usually does not have to deal with browser specific issues.

Contrary to the popular misconception JavaScript is not used only in browser, it became quite popular recently on the server-side. Why does the language continue to evolve and be successful both on the client and server side? Why more and more people choose JavaScript as the primary language in which they develop software? The examples below will provide a few of the answers to these questions. There are a few very nice design concepts in the language that have been there from the very beginning. JavaScript is flexible and powerful when it comes to working with functions, and this is what we would like to explore here.

<a name="running_examples"></a>

## Running examples

To execute examples you can use Firefox and the JavaScript console of the [Firebug](http://getfirebug.com) extension for Firefox The code makes use of "console.log" but there is nothing browser-specific in the examples and with minor modifications they can be run on every JavaScript implementation. You may take your time to set up the development environment so that you can play with examples in this article which is the best way to learn.

<a name="functions_first_class_citizens"></a>

## Functions are "first-class citizens"

In JavaScript functions are "first-class citizens": they can be passed as arguments and returned from other functions. You do not have to wrap your function in an anonymous class like in Java to do so. To illustrate this, let's add a few useful methods for working with arrays.

```javascript
function forEach(arr, callback) {
    for (var i = 0; i < arr.length; i++) {
        callback(arr[i], i);
    };
};

function map(arr, callback) {
    var result = [];
    for (var i = 0; i < arr.length; i++) {
        result.push(callback(arr[i]));
    };
    return result;
};

function reduce(arr, initial, callback) {
    var accumulated = initial;
    for (var i = 0; i < arr.length; i++) {
        accumulated = callback(accumulated, arr[i]);
    };
    return accumulated;
};

//Examples
var x = [1, 2, 3, 4, 5];

console.log("x = ");
forEach(x, function (el) {
    console.log(el);
});

console.log("squares of x = ");
forEach(map(x, function (el) {
    return el * el;
}), function (el) {
    console.log(el);
});

console.log("sum of elements of x = ");
console.log(reduce(x, 0, function (sum, el) {
    return sum + el;
}));

console.log("product of elements of x = ");
console.log(reduce(x, 1, function (sum, el) {
    return sum * el;
}));
```

**forEach** performs an action for each element, **map** transforms each element and **reduce** computes an aggregate value for a given array. The action, transformation or aggregation are specified by the **callback** function.

Even in this simple example it is already worth noting how we can combine two different callbacks with **reduce** and get completely different results. The code present in **reduce** is easy to reuse and we reused it twice: the action performed on the element is abstracted away from the way we iterate over the elements.

But this still looks a bit ugly: we have to always pass an array as an argument. In JavaScript it is easy to fix that by adding the methods **forEach**, **map** and **reduce** onto the Array class. To do that we will add functions to the **prototype** property of the Array. The **prototype** here is just a special kind of object, properties of which will be available in every created **Array** instance. For more details look at this [explanation](http://javascript.crockford.com/prototypal.html) of prototypal inheritance in JavaScript but in this post you can also view this just like a small magic trick.

```javascript
Array.prototype.forEach = function(callback) {
    for (var i = 0, length = this.length; i < length; i++) {
        callback(this[i], i);
    };
};

Array.prototype.map = function(callback) {
    var result = [];
    for (var i = 0, length = this.length; i < length; i++) {
        result.push(callback(this[i]));
    };
    return result;
};

Array.prototype.reduce = function(initial, callback) {
    var accumulated = initial;
    for (var i = 0, length = this.length; i < length; i++) {
        accumulated = callback(accumulated, this[i]);
    };
    return accumulated;
};

//Examples
var x = [1, 2, 3, 4, 5];

console.log("x = ");
x.forEach(function (el) {
    console.log(el);
});

console.log("squares of x = ");
x.map(function (el) {
    return el * el;
}).forEach(function (el) {
    console.log(el);
});

console.log("sum of elements of x = ");
console.log(x.reduce(0, function (sum, el) {
    return sum + el;
}));

console.log("product of elements of x = ");
console.log(x.reduce(1, function (sum, el) {
    return sum * el;
}));
```

The latest version of JavaScript already includes the methods **forEach**, **map** and **reduce** for arrays similar to what we just implemented and it would be wise not to override these methods. We will only define them on **Array.prototype** in case they are not yet there (maybe, for some older browser versions).

```javascript
if (!Array.prototype.forEach) {
    Array.prototype.forEach = function(callback) {
        ...
    };
};
if (!Array.prototype.map) {
    Array.prototype.map = function(callback) {
        ...
    };
};
if (!Array.prototype.reduce) {
    Array.prototype.reduce = function(initial, callback) {
        ...
    };
};
```

This shows that we can treat functions as values and use them in conditional statements. Besides this simple example passing functions as arguments is also widely used in client-side JavaScript programming for registering event listeners. We can, for example, add a click listener to body of the current document.

```javascript
document.body.addEventListener("click", function (event) {
    console.log("Click handled", event);
}, false);
```

Not only can we pass functions as arguments to other functions it is also possible to return a function from another function.

```javascript
function op(str) {
    switch (str) {
        case '+': return function(x, y) {
            return x + y;
        };
        case '+': return function(x, y) {
            return x + y;
        };
        case '-': return function(x, y) {
            return x - y;
        };
        case '*': return function(x, y) {
            return x * y;
        };
        case '/': return function(x, y) {
            return x / y;
        };
    };
};

console.log("op('+')(1, 2) = ", op('+')(1, 2));
console.log("op('-')(5, 3) = ", op('-')(5, 3));
console.log("op('*')(4, 5) = ", op('*')(4, 5));
console.log("op('/')(12, 3) = ", op('/')(12, 3));
```

This is a bit artificial example, but it illustrates well the general idea that a function can be considered a value.

<a name="closures_and_scopes"></a>

## Closures and scopes

There is a scope associated with each function invocation. In fact in JavaScript until the latest versions there were no other scopes, that is, a pair of brackets **{}** did not define a scope like in other languages such as Java or C. In the latest version it is possible to use **let** to define a scope, but this will not be covered in the present article.

At the time of its invocation each function captures the variables in the enclosing scope in which this function has been invoked. We say that the function "closes over" the values of the variables in the enclosing scope. This is called "closure." The following counter example demonstrates how the **counter** variable is "living" in a closure:

```javascript
function getCounter() {
    var counter = 0;
    return {
        increment: function() {
            return counter++;
        },
        reset: function() {
            counter = 0;
        }
    };
};

//Getting the counter object
var counter = getCounter();

//Executing its methods
console.log(counter.increment());
console.log(counter.increment());
console.log(counter.increment());
counter.reset();
console.log(counter.increment());
console.log(counter.increment());
```

We return an object from the **getCounter** method and the variable **counter** remains accessible for the functions defined in this object.

If we have a few function invocations then we can talk about a chain of scopes formed by a chain of function invocations much like in Scheme.

<a name="partial_application"></a>

## Partial Application

Partial application is converting a function of multiple arguments into a function with a fewer number of arguments. Simple example with multiplication:

```javascript
function multiply(x, y) {
    return x * y;
};

function twice(x) {
    return multiply(x, 2);
};

console.log("multiply(2, 3) = ", multiply(2, 3));
console.log("twice(3) = ", twice(3));
```

In **twice** the second argument **2** is captured and we get a function of one variable **x** rather than two variables **x** and **y**. It is easy to build a generic solution for partially applying a function.

```javascript
if (!Function.prototype.partial) {
    Function.prototype.partial = function(argTransformer) {
        //The current function that we partially apply
        var f = this;
        return function() {
            //Need to convert the function arguments into an array
            var args = Array.prototype.slice.call(arguments, 0);

            /*
             * Transforming the arguments and calling the initial function
             * with the transformed arguments. 'this' here is determined by the context
             * of invocation of the partially applied function and is not 'f'
             */
            return f.apply(this, argTransformer(args));
        };
    };
};

var multiply = function(x, y) {
    return x * y;
};
var double = multiply.partial(function (args) {
    args.push(2);
    return args;
});
var triple = multiply.partial(function (args) {
    args.push(3);
    return args;
});

console.log("multiply(2, 5) = ", multiply(2, 5));
console.log("double(3) = ", double(3));
console.log("triple(4) = ", triple(4));
```

Initially we keep the reference to **this** which references the function for which **partial** has been invoked. We use this reference later in the anonymous function that we return from **partial** to execute the original function together with the captured arguments and the arguments passed to the anonymous function at the point of its invocation. **argsTransformer**, which was the argument to the original **partial** invocation, combines the passed and captured arguments inside the returned anonymous function. Also, note that **this** in the anonymous function returned from **partial** is different from what is stored in the **f** variable, it is now the object on which the anonymous function was invoked.

<a name="memoization"></a>

## Memoization

Memoization is an optimization technique for avoiding expensive calculations in repeated function calls. Simple example:

```javascript
function expensiveComputation(x) {
    return x * x;
};

var cache = {};

function memoizedExpensiveComputation(x) {
    var result = cache[x];

    if (!result) {
        result = expensiveComputation(x);
        cache[x] = result;
    };
    return result;
};

console.log("expensiveComputation(5)", expensiveComputation(5));
console.log("memoizedExpensiveComputation(5)", memoizedExpensiveComputation(5));
console.log("memoizedExpensiveComputation(5)", memoizedExpensiveComputation(5));
```

**memoizedExpensiveComputation** caches the values that were already computed for particular arguments and returns these values directly from the cache avoiding calling **expensiveComputation**.

With JavaScript it is easy to build a generic solution for function memoization.

```javascript
function memoize(func, host, hash) {
    //By default memoize a function on the window object
    var host = host || window,
        hash = hash || {},
        original = host[func];
    //Only functions can be memoized
    if (!host[func] || !(host[func] instanceof Function)) {
        throw "Can memoize only a function or function is not defined in host";
    };
    //Redefine the function on the host object
    host[func] = function() {
        //The key in the cash is a JSON representation of arguments
        var jsonArguments = JSON.stringify(arguments);
        //If the value has not yet been computed
        if (!hash[jsonArguments]) {
            //Calling the original function with the arguments provided to host[func],
            //'this' in the original function will also be the same as in the redefined function in order to handle
            //host[func].call and host[func].apply
            hash[jsonArguments] = original.apply(this, Array.prototype.slice.call(arguments, 0));
        };
        return hash[jsonArguments];
    };
};

function fib(num) {
    return (num < 2) ? num : this.fib(num - 1) + this.fib(num - 2);
};
memoize("fib");

console.log("fib(5) =", fib(5));
console.log("fib(10) =", fib(10));
console.log("fib(11) =", fib(11));
```

First we check that **host** contains the function, name of which was passed as the first argument to **memoize**. If it does, we keep the reference to this function and then redefine the function **host[func]** just like in the simple example before. If the computed value can be looked up in the cache, than we return it without actually calling the original function, otherwise we call the original function and store the result in the cache. The key in the cache is the JSON representation of the arguments passed to the redefined **host** function. We have to call **Array.prototype.slice** on the arguments to convert them into an array object (a flaw in the design of JavaScript - **arguments** passed to the function is an array-like object, not an array) and then we call the original function **original** with **apply** on **this** which is defined by the invocation context of the redefined **host[func]**. We pass to the original function all the arguments that were passed to the redefined version of it. For user of the API using the redefined function is as transparent as using the original one. The call to **memoize** just redefines the function so that it starts returning already computed values from a cache.

<a name="lazy_evaluation"></a>

## Lazy evaluation

With JavaScript it is also relatively easy to implement lazy evaluation and lazy streams. Each stream consists of two parts: the head and the tail. The tail can be an actual element or a promise to compute the element. Execution of this promise to compute an element can be omitted at the point of defining the stream and can be done later. In JavaScript such a promise can be implemented as a function.

```javascript
function node(head, tail) {
    return [head, tail];
};

function head(stream) {
    return stream[0];
};

function tail(stream) {
    var tail = stream[stream.length - 1];
    return (tail instanceof Function) ? tail() : tail;
};

function drop(stream) {
    var h = head(stream);
    var t = tail(stream);
    stream[0] = t ? t[0] : null;
    stream[1] = t ? t[1] : null;
    return h;
};

function iterate(stream, callback, limit) {
    while (head(stream) && ((undefined == limit) || (limit > 0))) {
        limit && limit--;
        callback(drop(stream));
    };
};

function show(stream, limit) {
    iterate(stream, function (x) {
        console.log(x);
    }, limit);
};

//Examples
function upto(from, to) {
    return (from > to) ? null : node(from, function() {
        return upto(from + 1, to);
    });
};
function upfrom(start) {
    return node(start, function() {
        return upfrom(start + 1);
    });
};

console.log("upto:");
show(upto(3, 6));

console.log("upfrom:");
show(upfrom(7), 10);
```

The key part in this code is the **tail** function where we check whether **tail** is a promise rather than an actual element (that is, a function) and execute this promise if needed. Then defining each particular lazy stream is as easy as recursively defining a promise for the next element.

It is further possible to "objectify" the lazy stream code so that each stream represents an object and add the ability to filter, transform and unite streams. The full implementation is available [here](https://github.com/antivanov/Higher-Order-JavaScript/blob/master/src/streams/lazyStream.js)

<a name="not_a_functional_language"></a>

## Not a functional language but

Despite all the flexibility and ease of working with functions JavaScript is not a functional language. It has mutable shared state and the notion of the time of execution: if one statement precedes another it is executed earlier. Also there is no tail call recursion optimization in JavaScript, so the following code will not be optimized:

```javascript
function factorial(number) {
    if (0 == number) {
        return 1;
    };
    return number * factorial(number - 1);
};

console.log("factorial(10) = ", factorial(10));
```

While JavaScript is still not functional, the excellent support for functions helps a lot by adding the necessary flexibility and power to the language and in part explains the popularity of JavaScript both on the client and the server side.

More working examples like in this post can be found on github.com at ["Higher-Order JavaScript"](https://github.com/antivanov/Higher-Order-JavaScript) The code for this project was inspired by the blog series about ["Higher-Order Ruby"](http://blog.grayproductions.net/categories/higherorder_ruby) and the ["Higher-Order Perl"](http://hop.perl.plover.com/) book. If you also like Ruby, please, visit the blog [http://blog.grayproductions.net](http://blog.grayproductions.net) and consider buying the book [http://hop.perl.plover.com/](http://hop.perl.plover.com/) if you want to learn some good Perl.

## Links

[1. Scheme programming language](http://en.wikipedia.org/wiki/Scheme_(programming_language))
[2. Firebug](http://getfirebug.com)
[3. JavaScript prototypal inheritance](http://javascript.crockford.com/prototypal.html)
[4. addEventListener](https://developer.mozilla.org/en-US/docs/DOM/element.addEventListener)
[5. Partial Application](http://en.wikipedia.org/wiki/Partial_application)
[6. Memoization](http://en.wikipedia.org/wiki/Memoization)
[7. Tail call](http://en.wikipedia.org/wiki/Tail_call)
[8. Lazy evaluation](http://en.wikipedia.org/wiki/Lazy_evaluation)
[9. "Higher-Order JavaScript"](https://github.com/antivanov/Higher-Order-JavaScript)
[10. "Higher-Order Ruby" blog series](http://blog.grayproductions.net/categories/higherorder_ruby)
[11. "Higher-Order Perl" book](http://hop.perl.plover.com/)
