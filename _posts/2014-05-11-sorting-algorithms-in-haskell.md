---
layout: default
title: "Sorting Algorithms in Haskell"
date: 2014-05-11
---

# Contents

[Why Haskell?](#haskell)
[Quicksort](#quicksort)
[Mergesort](#mergesort)
[Bubble Sorting](#bubble_sort)

<a name="haskell"></a>

## Why Haskell?

Recently I decided to learn a bit of Haskell. Having programmed a bit in Clojure and having some familiarity with Common Lisp and Scheme I always wanted to take a closer look at Haskell.

What distinguishes Haskell is that it is a purely functional language, without state and variables. As a result statements in it are very close to mathematical expressions.

While reading [Learn You a Haskell](http://learnyouahaskell.com/) book I sat down the second day's evening behind my computer to write some simple sorting algorithms and was pleasantly surprised with the result: it was really easy and fast to implement these algorithms in Haskell, and the code reads almost like the definitions of the algorithms itself.

While I am still learning and my Haskell code may still be far from perfect I just wanted to share these first results and maybe convince you to also take a look at Haskell. If you are not comfortable with the syntax, please, refer to the first chapters of the [Learn You a Haskell](http://learnyouahaskell.com/) book. Some prior Haskell knowledge may be beneficial for reading this article although I try to explain the syntax and the language constructs a bit in the context of the provided examples.

So, let's go over some of the best known classical sorting algorithms and try to use Haskell to implement them.

<a name="quicksort"></a>

## Quicksort

Let's define the function **quicksort** that will implement the [Quicksort](http://en.wikipedia.org/wiki/Quicksort) algorithm.

Haskell is a statically typed language. When a function library is compiled, compiler tries to infer types where it can and we can also help it by specifying them explicitly.

```haskell
quicksort :: (Ord a) => [a] -> [a]
```

The function **quicksort** takes a list **[a]** of some type **a**, such that elements of **a** can be compared with each other (this is specified by using the **(Ord a)** guard). And then the function returns a list of the same type **[a]**.

The quicksort algorithm itself is really simple:

1. We pick some element x in the list

2. The rest of the elements in the list are separated into two lists: elements smaller than x and elements greater than x

3. The algorithm is applied recursively to these lists and then the list with smaller elements, the selected element and the list of greater elements are concatenated together and the sorting is done

Here is how we would implement this in Haskell:

```haskell
quicksort (x:xs) = quicksort [y | y <- xs, y <= x] ++ [x] ++ quicksort [y | y <- xs, y > x]
```

**quicksort** takes an argument **(x:xs)** which is a list consisting of the first element **x** and the rest of the list **xs**. Then we apply list comprehension **[y | y <- xs, y <= x]** to get a list of all the elements in the list **xs** that are smaller or equal than **x**. Then we concatenate the resulting list with a single element list **[x]** and the list of elements that are greater than **x**.

So the recursion on which the Quicksort algorithm is built is now defined by the function **quicksort**, but we still need to finish the recursion at some point, so we need to specify the condition when the recursion ends. This is also easily done in Haskell by augmenting the definition of the function **quicksort** with one more rule:

```haskell
quicksort [] = []
```

Thus the algorithm applied on an empty list will return an empty list.

Combining everything together we get the complete Quicksort implementation in Haskell:

```haskell
quicksort :: (Ord a) => [a] -> [a]
quicksort [] = []
quicksort (x:xs) = quicksort [y | y <- xs, y <= x] ++ [x] ++ quicksort [y | y <- xs, y > x]
```

Note, how mathematically pure this implementation looks. No variables, no order in which the steps of the algorithm should be performed, just the specification of how the algorithm should work.

<a name="mergesort"></a>

## Mergesort

Mergesort is a little more complicated to implement. The algorithm as follows:

1. List is split into two parts

2. Two parts are sorted by the algorithm

3. The sorted parts are merged by a special merging procedure for sorted lists

Let's first define how we split a list into two parts:

```haskell
mergesort'splitinhalf :: [a] -> ([a], [a])
mergesort'splitinhalf xs = (take n xs, drop n xs)
    where n = (length xs) `div` 2
```

The function **mergesort'splitinhalf** returns a pair of arrays into which the original array was split. **n** is equal to the half of the length of the array, and then we use the standard functions **take** and **drop** to get the first **n** elements of the list **take n xs** and the rest of the list after those first elements **drop n xs**.

Let's now define a function for merging two sorted arrays:

```haskell
mergesort'merge :: (Ord a) => [a] -> [a] -> [a]
mergesort'merge [] xs = xs
mergesort'merge xs [] = xs
mergesort'merge (x:xs) (y:ys)
    | (x < y) = x:mergesort'merge xs (y:ys)
    | otherwise = y:mergesort'merge (x:xs) ys
```

The function receives two arrays and produces one array of the same type. The algorithm for merging:

1. If the first list is empty **[]** then the result of the merge is the second list **xs**

2. If the second list is empty **[]** then the result of the merge is the first list **xs**

3. Otherwise we compare the first elements of the lists and append with the colon **:** function the least of them to the new list which is the result of merging the remaining two lists

Now that we have defined the functions **mergesort'splitinhalf** and **mergesort'merge** we can easily define the function **mergesort**.

```haskell
mergesort :: (Ord a) => [a] -> [a]
mergesort xs
    | (length xs) > 1 = mergesort'merge (mergesort ls) (mergesort rs)
    | otherwise = xs
    where (ls, rs) = mergesort'splitinhalf xs
```

If the length of the list is greater than 1 then we do the standard step of the algorithm. Otherwise the list with the length of 1 is already sorted (the condition for ending the recursion).

The code reads exactly the same as the textual description of the algorithm given earlier but now this is a formal and shorter specification.

The complete code for Mergesort:

```haskell
mergesort'merge :: (Ord a) => [a] -> [a] -> [a]
mergesort'merge [] xs = xs
mergesort'merge xs [] = xs
mergesort'merge (x:xs) (y:ys)
    | (x < y) = x:mergesort'merge xs (y:ys)
    | otherwise = y:mergesort'merge (x:xs) ys

mergesort'splitinhalf :: [a] -> ([a], [a])
mergesort'splitinhalf xs = (take n xs, drop n xs)
    where n = (length xs) `div` 2

mergesort :: (Ord a) => [a] -> [a]
mergesort xs
    | (length xs) > 1 = mergesort'merge (mergesort ls) (mergesort rs)
    | otherwise = xs
    where (ls, rs) = mergesort'splitinhalf xs
```

<a name="bubble_sort"></a>

## Bubble Sorting

And now the Bubble sorting algorithm: we change places in pairs of elements while we can do so, that is, while there is still a pair or elements **(x, y)** such as **x > y**.

Let's first define the function that will go through all the elements in a list and exchange pairs of elements when it sees that the sorting order is wrong.

```haskell
bubblesort'iter :: (Ord a) => [a] -> [a]
bubblesort'iter (x:y:xs)
    | x > y = y : bubblesort'iter (x:xs)
    | otherwise = x : bubblesort'iter (y:xs)
bubblesort'iter (x) = (x)
```

Then we just need to apply this function **n** times - the length of the list that should be sorted.

```haskell
bubblesort' :: (Ord a) => [a] -> Int -> [a]
bubblesort' xs i
    | i == (length xs) = xs
    | otherwise = bubblesort' (bubblesort'iter xs) (i + 1)

bubblesort :: (Ord a) => [a] -> [a]
bubblesort xs = bubblesort' xs 0
```

We do this by defining a function **bubblesort'** that takes two arguments: the current list and the number of the current iteration **i**.

Essentially what we are doing here, is transforming iteration into a recursion, so that Bubble sorting becomes a recursive algorithm.

## Links

[Source code](https://github.com/antivanov/misc/blob/master/Haskell/sorting_algorithms.hs)
[Haskell](http://www.haskell.org/haskellwiki/Haskell)
[Learn You a Haskell book](http://learnyouahaskell.com/)
[Bubble sorting](http://en.wikipedia.org/wiki/Bubble_sort)
[Mergesort](http://en.wikipedia.org/wiki/Merge_sort)
[Quicksort](http://en.wikipedia.org/wiki/Quicksort)
