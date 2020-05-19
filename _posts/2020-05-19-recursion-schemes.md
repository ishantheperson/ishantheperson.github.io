---
layout: post
title: "Using recursion schemes"
date: 2020-05-17 15:19:00
tags: haskell
---

The Haskell compiler GHC has implemented [over 115 language extensions](https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/glasgow_exts.html). A lot of these let you do some really interesting things,
such as [statically checking array indices](https://github.com/ishantheperson/ModelChecking/blob/master/src/Vector.hs) via dependant types, or even [write programs completely within the type system](https://aphyr.com/posts/342-typing-the-technical-interview). Unfortunately a lot of these are not "practical" in the sense that
the effort involved in using these features outweighs the benefits. But I recently read about a way to put
some of that machinery to good use called "recursion schemes". These help us write code which is more obviously
correct, and more concise. 

I found the following series about recursion schemes by Patrick Thomson, 
and I really wish I'd encountered it earlier. I would highly recommend
anyone with any interest in functional programming read this series, or at least
the first two articles. These are not Haskell-specific, the concepts can be implemented
in many other languages (although probably not as easily). This post only shows applications
of the material described below, so reading them isn't necessary to see why they are useful. 

 - [Introduction to Recursion Schemes](https://blog.sumtypeofway.com/posts/introduction-to-recursion-schemes.html)
 - [Recursion Schemes - Part II](https://blog.sumtypeofway.com/posts/recursion-schemes-part-2.html)

This article describes the specific implementation of these abstractions in Haskell 
(specifically the `recursion-schemes` package):
 - [Recursion Schemes - Part 4.5](https://blog.sumtypeofway.com/posts/recursion-schemes-part-4-point-5.html)

In this post I will show how I refactored some code in a recent project I was working on. This project was a compiler
for a functional language, and so I had defined a syntax tree data type like this:

```hs
data F0Expression symbol typeInfo = 
    F0Lambda symbol (typeInfo F0Type) (F0Expression symbol typeInfo) -- ^ fn x (: t) => etc 
  | F0App (F0Expression symbol typeInfo) (F0Expression symbol typeInfo) -- ^ e1 e2 
  | F0Let (F0Declaration symbol typeInfo) (F0Expression symbol typeInfo) -- ^ let decl in e end. Multiple decls are desugared to nested lets by the parser
  | F0If (F0Expression symbol typeInfo) (F0Expression symbol typeInfo) (F0Expression symbol typeInfo) -- ^ if e1 then e2 else e3 
  | F0Literal F0Literal 
  | F0TagValue String Int (F0Expression symbol typeInfo) -- ^ introduce sum type
  | F0Case (F0Expression symbol typeInfo) [(symbol, (symbol, F0Expression symbol typeInfo))] -- ^ rules are <constructor> (<bound var> <e>)
  | F0Identifier symbol 
  | F0Tuple [F0Expression symbol typeInfo] -- ^ Construct a tuple
  | F0TupleAccess Int Int (F0Expression symbol typeInfo) -- ^ Access element i out of n in e 
  | F0TypeAssertion (F0Expression symbol typeInfo) F0Type 
  | F0OpExp F0Operator [F0Expression symbol typeInfo] -- ^ arithmetic ops, comparison ops, etc. 
  | F0ExpPos SourcePos (F0Expression symbol typeInfo) SourcePos -- ^ Start, Expression, End 
```

Although this language is fairly small (it only has the basics of a ML-like language),
there are already 12 cases. 

Now suppose we wanted find the free variables of a given expression. 
Free variables are variables which appear in an expression 
without being declared (e.g. the free variables of `fn x => x + y + z` are `y` and `z`)
There are really only 3 ways to declare a variable in our language.
 - In a `let` expression (`F0Let`)
 - In a lambda argument e.g. `fn x => ..` (`F0Lambda`)
 - As part of a `case` expression (`F0Case`)

And there is of course only one way to reference a variable (`F0Identifier`). 
So the problem is, there are only 4 interesting cases in `F0Expression`, but 
right now it seems we would have to write 12 different rules.
```hs
freeVariables :: Ord s => F0Expression s f -> Set s 
freeVariables = \case
  F0Lambda name _ e -> name `Set.delete` freeVariables e
  F0App e1 e2 -> freeVariables e1 `Set.union` freeVariables e2 
  F0Identifier x -> Set.singleton x 
  F0Literal _ -> Set.empty 
  F0OpExp _ es -> Set.unions (map freeVariables es)
  F0ExpPos _ e _ -> freeVariables e 
  F0If e1 e2 e3 -> Set.unions (map freeVariables [e1, e2, e3])
  F0Tuple es -> Set.unions (map freeVariables es)
  F0TupleAccess _ _ e -> freeVariables e 
  F0TagValue _ _ e -> freeVariables e
  F0Case obj arms -> freeVariables obj `Set.union` Set.unions (map (\(_, (x, e)) -> x `Set.delete` freeVariables e) arms)
  F0TypeAssertion e _ -> freeVariables e
  F0Let d e -> freeVariablesDecl d `Set.union` freeVariables e 
  where freeVariablesDecl = \case 
          F0Value _ _ e -> freeVariables e 
          F0Fun _ _ _ e -> freeVariables e 
          F0Data {} -> Set.empty
```
This function has many undesirable features:
 - The majority of the code does nothing but recurse on nested `F0Expression`s
   and then combine the results with `Set.union` 
 - It's not easy to tell if we made a mistake when writing this function. 
   We could forget to recurse on an argument, or forget to combine some of the results

By using a recursion scheme we can actually almost completely automate this process. 
```hs
freeVariables :: Ord symbol => F0Expression symbol typeInfo -> Set symbol
freeVariables = cata go
  where go (F0IdentifierF x) = Set.singleton x 
        go (F0LambdaF x _ e) = Set.delete x e
        go (F0LetF d e) = Set.union (freeVariablesDecl d) e
        go (F0CaseF obj rules) = fold $ obj : map (\(_, (x, s)) -> Set.delete x s) rules
        go other = fold other

        freeVariablesDecl = \case 
          F0Value _ _ e -> freeVariables e 
          F0Fun _ _ _ e -> freeVariables e 
          F0Data {} -> Set.empty
```

This is significantly more concise than the previous declaration. We only explicitly write out
the rules which are significant. The catamorphism function `cata` from the `recursion-schemes` package 
automatically handles the recursion, and the `fold` function from the Haskell standard library will 
automatically combine the recursive results. 

It's also a lot easier to tell if this function is correct. It's a lot easier to check 5 equations compared to 12. And if we add more features to our language (e.g. record expressions, namespaces, etc.) we actually don't
have to modify this function at all. 

The best part about all of this is that I didn't need to modify the original declaration of `F0Expression`
at all. The library provides a way to use Template Haskell to automatically generate all code which makes this work.
I only had to add this:
```hs
makeBaseFunctor ''F0Expression
```

Here are some other examples of functions I rewrote using `cata`. If I didn't use recursion schemes here, I would
have to write 12 lines of tedious recursion boilerplate for each function:
```hs
-- Performs a given type substitution "s" on an expression
subst :: F0Expression Symbol Identity -> F0Expression Symbol Identity
subst s = cata go 
  where go (F0LambdaF x (Identity t) e) = F0Lambda x (Identity $ subst s t) e 
        go e = embed e  

-- Gives the free type variables of an expression
freeTypeVariables :: F0Expression Symbol Identity -> Set TypeVariable
freeTypeVariables = cata go 
  where go (F0LambdaF _ (Identity t) e) = freeTypeVariables t <> e 
        go other = fold other 

-- Source position information is attached to nodes during parsing
-- in order to give better error messages. However when writing test cases
-- or inspecting the generated tree it can become tedious. This function
-- removes the position tags from the tree.
removePositionInfo :: F0Expression symbol typeInfo -> F0Expression symbol typeInfo
removePositionInfo = cata go
  where go (F0ExpPosF _ e _) = e
        go e = embed e 
```