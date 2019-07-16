---
layout: post
title: "Haskell Monads by Example"
date: 2019-07-14 21:39:00
tags: haskell monads
---

There are [a lot](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html) 
of [great introductions](http://learnyouahaskell.com/a-fistful-of-monads)
to [monads](https://wiki.haskell.org/Monad_tutorials_timeline#), but since monads are 
notorious for being a difficult topic to understand, the world could probably benefit from
another one. This blog post will focus on making the idea of monads more concrete through
examples of some of the more ubiquitous monads. 

## Overview 

A lot of people have heard of the `IO` monad, and this leads them to think that
monads allow "side effects." However, monads are still a functionally pure concept - it's
only the IO monad specifically which allows side effects 
(although some would religiously argue that `IO` is still pure).

In general, a monad `m` is a type constructor which allows us to combine functions (using the
infix operator `>>=` or `do` notation)
that produce a result which has type `m a`. The specifics of what "combining" functions
means is what makes each monad special. Let's look at some examples 

### `Maybe` and `do` notation 

The `Maybe a` type either _has a value_ (`Just a`) or _doesn't have a value_ (`Nothing`).

Let's look at a function which returns `Maybe` (it's in the Prelude):

```haskell
lookup :: Eq a => a -> [(a, b)] -> Maybe b
```

This function takes a key and a list of key-value pairs, and tries to find the related pair.
In this example we attempt to look up the phone numbers of some well-known individuals. 

```haskell
directory = [("Alice", "123-4567"), ("Bob", "987-6543")]
-- lookup "Alice" directory => Just "123-4567"
-- lookup "Eve" directory => Nothing
```

Let's say we have another list which maps phone numbers to outstanding phone bill amounts.

```haskell
userIds = [("123-4567", 44), ("987-6543", 88)] -- Amounts in USD
```

Now we can try to write a function `getPhoneBill` to get someone's phone bill via their name using
the `lookup` function descriped earlier.
There are three outcomes for this function:

 - The name and the associated phone number are both found
 - The name isn't in `directory`
 - The phone number found in `directory` isn't in `userIds`

First let's look at it without using `do` notation:

```haskell
getPhoneBill name = lookup name directory 
                       >>= (\phoneNumber -> lookup phoneNumber userIds)

-- Can also be written more concisely and less legibly as follows:
getPhoneBill name = lookup name directory >>= flip lookup userIds
```

In the case of `Maybe`, we can think of `v >>= f` as follows:
 - If `v` evaluates to `Just x`, then `v >>= f` evaluates to `f x`.
   We've "unwrapped" `v` by "peeling off" the `Just` layer
 - If `v` evaluates to `Nothing`, then there isn't a value to to run `f` on.
   So `v >>= f` evaluates to Nothing and we've skipped out `f` entirely

In our example, `v` would be `lookup name directory` and `f` would be the
lambda function `\phoneNumber -> lookup phoneNumber userIds`.
If the first lookup succeeds, then `>>=` passes the phone number `String` to function on its right.
Otherwise, the entire thing remains `Nothing` and the function on the right is never run.

Here, the `Maybe` monad allows us to stop our computation if any of its parts returns `Nothing`.
However, the function above may seem quite ugly and still as unreadable as compared to 
using `case` statements. To solve this, Haskell provides `do` notation to make monads more pleasant.

```haskell
getPhoneBill name = do phoneNumber <- lookup name directory
                       lookup phoneNumber userIds
```

In the context of the `Maybe` monad, using `<-` indicates that we are trying to extract
a `Just` value from the right side. If we get `Nothing` instead, the value of the whole
`do` block becomes `Nothing` as well.

It's important to see why this is equivalent to the earlier function using `>>=`. In general,
when we have code like this:

```haskell
do y <- f x
   g y
```

it is the same as 

```haskell
f x >>= (\y -> g y)
```

This applies recursively, so if we have

```haskell
do y <- f x
   z <- g y 
   h y z 
```

it becomes

```haskell
f x >>= 
  (\y -> g y >>= 
    (\z -> h y z))
```

This becomes convenient, especially when our processes involve more steps.

### `Either`

Now let's look at the `Either` monad, which despite its name is actually
similar to `Maybe`. A value of type `Either a b` is either `Left a` or `Right b`.
We use `Left` to indicate that the computation has stopped ("failed") with a value, or
`Right` to indicate success. In practice, we can use `Left` to store error information.

For example, let's say we are building a simple interpreter. There are a lot of things
that can go wrong, here are a few common ones:
 - A nonexistent variable may be referenced 
 - An invalid arithmetic operation may be attempted e.g. `1 / 0`
 - An operation may receive operands of invalid types e.g. `1 - "hello"`

 So let's say we have some AST-like data types like this:
 ```haskell
 data Expression = Subtract Expression Expression 
                 | Divide Expression Expression
                 | Variable String 
                 | IntConstant Int
                 | StringConstant String 

data ExpressionResult = IntResult Int | StringResult String
```

We can create a type representing the errors we could encounter.
Our functions can return these to indicate something went wrong
```haskell
data RuntimeError = UnknownVar String | DivZero | WrongType
```

When we evaluate an expression, we either need to return a result, or an error.
But since expression often contain subexpressions, we also need to handle errors
in subexpressions. This is where the `Either` monad helps us:

```haskell
evalExpression = \case 
  Subtract lhs rhs -> do lhsResult <- evalExpression lhs
                         rhsResult <- evalExpression rhs 
                         case (lhsResult, rhsResult) of 
                           (IntResult a, IntResult b) -> Right $ IntResult (a - b)
                           _ -> Left WrongType

  -- This repetition could be avoided by 
  -- having a common ArithmeticOp constructor
  -- but that is overkill for this example
  Divide lhs rhs -> do lhsResult <- evalExpression lhs 
                       rhsResult <- evalExpression rhs 
                       case (lhsResult, rhsResult) of 
                         (IntResult _, IntResult 0) -> Left DivZero
                         (IntResult a, IntResult b) -> Right $ IntResult (a `div` b)
                         _ -> Left WrongType 

  Variable v -> ... -- code to lookup v and return Left/Right appropriately, 
  IntConstant i -> Right $ IntResult i 
  StringConstant s -> Right $ StringResult s 
```

If evaluation of an AST node suceeds, we use `Right` to indicate success.
Otherwise we use `Left` and provide data about the error. Instead of `Right` we could also use
the monad function `return :: a -> m a`, which in the case of the `Either` monad will place a value
in `Right`, and in the case of `Maybe` into `Just`.

If `evalExpression lhs` or `evalExpression rhs` returns `Left e`, then the entire `do` block is stopped
and its value will be `Left e`. So our errors will automatically propogate for us, and we don't need to 
manually handle it. It remains easy to add more complex expressions or more error types.
