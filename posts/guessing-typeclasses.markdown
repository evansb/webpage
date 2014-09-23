---
title: Guessing Implementation of Type Classes
date: 2014-07-25 14:09:00
author: Evan Sebastian
---

Implementing Haskell type classes can be arduous, it is one of the many aspects that made this language
notoriously hard for beginners.
Yet, it is difficult to get a full grasp and appreciate the beauty of type classes without actually
deriving its instance for some basic data types (or at least having seen the implementation).

In this blog post, I would like to illustrate an example of thinking process that can be used
when implementing type classes whose functionality is not so obvious to determine.

Before we start, there are at least two hints that is given for free before you are trying to
derive a new instance of a type classes.

1. The type signature given in the type class declaration.
2. The laws given in the documentation.

Sometimes you don't even need to understand what its instances should do. You provide an implementation
that follows the type signature and satisfy the laws. After everything is done, you will begin to
understand its behavior, which most likely is the correct one.

To illustrate how we can harness those two hints, I will re-implement the function monad `(->) r`.

**Step one : Look at its typeclass definition and properly substitute their parameters.**

In this case, we need to substitute `m` with `(-> t)`

    class Monad m where
      (>>=) :: m a -> (a -> m b) -> m b
      return :: a -> m a

Therefore, our instance implementation should have this form.

    instance Monad ((->) t) where
      (>>=) :: ((->) t a) -> (a -> ((->) t b)) -> ((->) t b)
      return :: a -> ((->) t a)

**Start matching the types without caring about its meaning.**

Let us start with `(>>=)` since it seems to be more difficult. I would like to
change the postfix form of `(->)` into infix form to improve readibility. Now we have.

    (>>=) :: (t -> a) -> (a -> (t -> b)) -> (t -> b)

So the first parameter should be a function, the second one is a function that returns
a function. Let us pick ugly but descriptive names for these parameters, `ta` (t to a) and `atb`
(a to (t to b)).

We would like our `>>=` to return a function that takes type `t` as parameter. Hmm, let us write it
in a lambda form first.

    ta >>= atb = \t -> -- what should goes here?

The result of `\t` must be of type `b`. The only way to get `b` is via `atb`. But `atb` accepts `a`
instead of `t`. This leaves us only to the first argument, `ta`. Hmm, let us plug in `ta t`.

    ta >>= atb = \t -> (ta t)

Now our lambda function has type `(t -> a)`. Not good, since we are expecting `(t -> b)`.
But we still have the second argument! Let us plug it in before `(ta t)`.

    ta >>= atb = \t -> atb (ta t)

Since `atb _` has type `(t -> b)` we have `(t -> (t -> b))`. How do we reduce it into `t -> b`?
Well by applying it into `t`, of course.

    ta >>= atb = \t -> atb (ta t) t

Now that we have a syntatically correct expression, let us change back the parameter names.

    f >>= g = \x -> g (f x) x

Our `>>=` is done, now we proceed to `return`. It has the following type.

    return :: a -> (t -> a)

How do you create any function from any value that always return something of the same type as
the value?

Remember that we can think of type as a set of valid values.
Now, since we are on generic context `a`, we should make sure that our `return` works
for all sets (or types, if you like).
Hence, we *cannot* do the following.

1. Use any function other than type `a -> .... -> a`.
2. Include any particular member of the set in the computation.

At this point, you should realise that `const` is a possible candidate of `return`.
You may double check it by searching for `a -> t -> a` in `hoogle`.

Here is our finalised function monad implementation.

    instance Monad ((->) t) where
          return = const
          f >>= g = \x -> g (f x) x

**Step three : Proving that your implementation satisfy the law.**

These are the monad laws that I shamelessly copied from the [Haskell wiki](http://www.haskell.org/haskellwiki/Monad_laws)
with some modification in the parameter name. The proof is mine and is not formally checked (google is not friendly on this one), so let me know if I made any mistakes.

    Left identity:      return a >>= g ≡ g a
    Proof:
        return a >>= g ≡ const a >>= g
                       ≡ \x -> g (const a x) x
                       ≡ \x -> g a x
                       ≡ g a (QED)

    Right identity:       m >>= return ≡ m
    Proof:
        m >>= return   ≡ \x -> return (m x) x
                       ≡ \x -> const (m x) x
                       ≡ \x -> m x
                       ≡ m (QED)

    Associativity:    (m >>= f) >>= g  ≡ m >>= (\x -> f x >>= g)
    Proof:
        m >>= (\x -> f x >>= g) ≡ m >>= (\x -> (\q -> g (f x q) q))
                                ≡ \y -> (\x -> (\q -> g (f x q) q)) (m y) y
                                ≡ \y -> (\q -> g (f (m y) q) q) y
                                ≡ \y -> g (f (m y) y) y
                                ≡ \y -> g ((m >>= f) y) y
                                ≡ (m >>= f) >>= g (QED)

Honestly, I almost give up and left out the last one with a funny "left as exercise".
The last one was hard. Let me know if there are any errors in my proof.

**Last step : Start growing your reasoning by using examples**

Now that we have correct implementation of function Monad, let us see some examples.

    exampleOne = do
            a <- (*4)
            b <- (*3)
            c <- (*2)
            return (a + b)

This will return a function of `Integer -> Integer` (by default), say `x`.
When `exampleOne` applied to `x`, a is bound to the result of `(*4) x`, similarly to b and c.
Remember that `return (a + b)` is just `const (a + b)`.

Another trivial example with list...

    exampleTwo = do
            a <- map (+1)
            b <- map (+2)
            return (zip a b)

or FizzBuzz...

    fizzBuzz = do
        a <- \x -> if x `mod` 3 == 0 then "Fizz" else ""
        b <- \y -> if y `mod` 5 == 0 then "Buzz" else ""
        return (a ++ b)


We just implemented `(->) r` Monad without first knowing about its functionality.
This technique can be useful when we are trying to implement something with unknown behaviour,
such as when we try to create an instance of a type class for an unfamiliar type.