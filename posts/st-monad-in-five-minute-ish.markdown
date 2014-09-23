---
title: ST Monad in 5 minute-ish 
date: 2014-08-25 15:09:00
author: Evan Sebastian
---

### Motivation

IO Monad already show us how controlled side effects rocks.
Still, sometimes we are longing for impurity.

ST Monad make it possible to have impure code inside a pure function 
while still maintaining safety.

### ST is Two Way Monad
To appreciate ST, we must first understand the notion of one-way vs
two-way monad.
Two way Monad provides you mechanism to get the result out from the context or
in short, `M a -> a`. For instance, in `State` monad you can get the 
state function inside back by using `runState`.
This is not possible in one way monad.

`IO` is one way monad for good reason, *safety*. In short, a function 
cannot deny the fact that IO (i.e side effect) has been done, provided no
`unsafePerformIO` is used.

Luckily, `ST` is two way monad, we can escape ST via `runST`.

### ST and Mutable References

Mutable reference is basically variable in imperative programming language.

With ST, it is trivial to create some building blocks involving
mutable reference such as `for`.
A `for` block is simple, it is a function that takes an inital state, end
state, step function, and a block that will be executed. It will execute the
block followed by applying step function to the initial state until the
initial state is equal to the end state.

    import Control.Monad (unless)
    import Control.Monad.ST
    import Data.STRef

    for :: (Eq a)
        => STRef s a -- ^ Initial state
        -> STRef s a -- ^ End state
        -> (STRef s a -> ST s ()) -- ^ Step function
        -> ST s b -- ^ Block
        -> ST s ()
    for init_ end_ step_ block_ = do
        lhs <- readSTRef init_
        rhs <- readSTRef end_
        unless (lhs == rhs) $ do
            block_
            step_ init_
            for init_ end_ step_ block_

Now you can write a pure function in an imperative manner.
With `runST`, the impure and pure computation can be separated clearly, hence
safety is guaranteed.

    -- | Return sum from 1 until (n - 1)
    sumUntil :: Int -> Int
    sumUntil n = runST $ do
        -- Create mutable references
        i <- newSTRef (1::Int)
        j <- newSTRef n
        -- Mutate references
        let step x = modifySTRef x (+ 1)
        s <- newSTRef (0::Int)
        for i j step $ do
            i' <- readSTRef i
            modifySTRef s (+ i')
        readSTRef s

### Wrap-up

Beyond simple mutable references,
ST Monad allows the creation of more complex data  structure such as array
and hash tables.
In more sophisticated libraries, a neat thaw-freeze method is usually used.
You can `thaw` an immutable data structure temporarily, 
modify it inside ST context, and `freeze` it to turn it immutable again.
