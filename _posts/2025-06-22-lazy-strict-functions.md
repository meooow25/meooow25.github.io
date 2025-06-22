---
layout: post
title: Lazy and strict functions
description: On lazy and strict functions in Haskell
tags: haskell
---

If you've written some amount of Haskell you have likely found yourself looking
at two functions with the same type, where one is a strict version of the other.
We have

* [`Data.List.foldl`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-List.html#v:foldl)
  and [`Data.List.foldl'`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-List.html#v:foldl-39-)
* [`Data.List.scanl`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-List.html#v:scanl)
  and [`Data.List.scanl'`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-List.html#v:scanl-39-)
* [`Data.List.iterate`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-List.html#v:iterate)
  and [`Data.List.iterate'`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-List.html#v:iterate-39-)
* [`Control.Monad.Trans.State.Strict.modify`](https://hackage.haskell.org/package/transformers-0.6.2.0/docs/Control-Monad-Trans-State-Strict.html#v:modify)
  and [`Control.Monad.Trans.State.Strict.modify'`](https://hackage.haskell.org/package/transformers-0.6.2.0/docs/Control-Monad-Trans-State-Strict.html#v:modify-39-)
* [`Data.IORef.modifyIORef`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-IORef.html#v:modifyIORef)
  and [`Data.IORef.modifyIORef'`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-IORef.html#v:modifyIORef-39-)
* and so on...

How does a strict function differ from the lazy one? Well, we can put them into
a few categories.

1. The function repeatedly modifies some value. In the strict version, this
   value is forced to WHNF
   ([weak head normal form](https://wiki.haskell.org/Weak_head_normal_form)) at
   each step. For example, `foldl'` forces the accumulator at each step and
   `foldl` does not[^opt].
2. The function returns some structure. In the strict version, forcing the
   structure to WHNF also forces some of its contents to WHNF. For example,
   after `modify'`, if the tuple `(a,s)` is forced to WHNF, it will also force
   the state `s` in it to WHNF. With `modify` this is not the case.
3. Both of the above. `scanl'` forces the accumulator at every step, and also
   for each `(:)` generated forcing the `(:)` forces the element in it.

Both lazy and strict versions of functions are useful, depending on the
situation, but there are plenty of functions with only a lazy version. For
example, take
[`Data.List.map`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-List.html#v:map).
It is lazy, but we can easily define a strict version[^map].

```hs
-- lazy
map :: (a -> b) -> [a] -> [b]
map f = go
  where go [] = []
        go (x:xs) = f x : go xs

-- strict
map' :: (a -> b) -> [a] -> [b]
map' f = go
  where go [] = []
        go (x:xs) = let !y = f x -- note the bang on y
                    in y : go xs
```

`map'` falls into the second category of strict functions. The evaluation of `f
x` is tied to the constructor `(:)` using a
[strict binding](https://downloads.haskell.org/ghc/9.12.2/docs/users_guide/exts/strict.html#strict-bindings)
(one can also use [`seq`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Prelude.html#v:seq)).
If the `(:)` is forced to WHNF, the element in it is also forced to WHNF.

Let's perform an experiment to confirm that a strict `map` is useful. We will
sum up the elements of a list after mapping over it. One might expect that
`map'` will be more efficient because it does not need to create
[thunks](https://wiki.haskell.org/Thunk) for the elements.

<details markdown="1">

<summary>
Benchmark source
</summary>

```hs
{- cabal:
build-depends: base, tasty-bench
-}
{-# LANGUAGE BangPatterns #-}

import Prelude hiding (map)
import Test.Tasty.Bench

main :: IO ()
main = defaultMain
  [ env (pure xs_) $ \xs -> bgroup ""
    [ bench "map" $ whnf (sum . map (+1)) xs
    , bcompare "$NF == \"map\"" $
      bench "map'" $ whnf (sum . map' (+1)) xs
    ]
  ]
  where
    xs_ = [1..1000] :: [Int]

map :: (a -> b) -> [a] -> [b]
map f = go
  where go [] = []
        go (x:xs) = f x : go xs

map' :: (a -> b) -> [a] -> [b]
map' f = go
  where go [] = []
        go (x:xs) = let !y = f x
                    in y : go xs
```

</details>

Indeed, on my machine strict `map'` followed by `sum` takes 25% less time
compared to `map` followed by `sum`.

```
    map:  OK
      7.69 μs ± 744 ns
    map': OK
      5.75 μs ± 338 ns, 0.75x
```

A similar difference should be observable on other environments. Despite the
demonstrated utility, we don't have a `map'`. We also don't have `zipWith'`,
`liftA2'`, etc.

I would say it's reasonable to expect these in `base`.

Admittedly, having two versions of many higher-order functions doesn't seem
appealing. But why do we need two versions anyway? Can we have just one function
do the job of both?

## `Solo`

Yes, and it's quite simple. We use
[`Solo`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-Tuple.html#t:Solo)!

```hs
data Solo a = MkSolo a
```

`Solo` is the 1-tuple, available in `Data.Tuple` since `base-4.16`/ `GHC 9.2`.
If your `base` is older, you can define it yourself since it's ordinary `data`.

Consider this `map` variant that uses `Solo`. Defined correctly, it can work
as both lazy and strict `map`.

```hs
mapSolo :: (a -> Solo b) -> [a] -> [b]
mapSolo f = go
  where go [] = []
        go (x:xs) = case f x of MkSolo y -> y : go xs
```

```hs
-- lazy
map :: (a -> b) -> [a] -> [b]
map f = mapSolo (\x -> MkSolo (f x))

-- strict
map' :: (a -> b) -> [a] -> [b]
map' f = mapSolo (\x -> let !y = f x in MkSolo y)
-- or more succinctly
--     = mapSolo ((MkSolo $!) . f)
```

In `map'`, the evaluation of `y` is tied to the `MkSolo` constructor, which
is in turn tied to the `(:)` in `mapSolo`. So, forcing `(:)` also forces `y`,
making it a strict mapping function.  
In `map`, the `MkSolo` constructor simply wraps `f x` and forcing `MkSolo` does
not force the value in it.

Since `map` and `map'` are trivially defined in terms of `mapSolo`, we don't
really need them. We are left with just one `mapSolo`, and the function we
provide to it determines the strictness!

`mapSolo` also allows for something new: a mix of lazy and strict applications.

```hs
import qualified Data.Set as Set

data SetOrList a = Set !(Set.Set a) | List [a]

sizes :: [SetOrList a] -> [Int]
sizes = mapSolo f
  where -- Set.size is cheap, compute it right away.
        f (Set s) = MkSolo $! Set.size s
        -- length can take arbitrarily long, let's not compute it unless
        -- it's needed.
        f (List l) = MkSolo (length l)
```

Now, you may have noticed that I wrote above, "Defined correctly...". What could
go wrong with `mapSolo`?

```hs
mapSoloWrong :: (a -> Solo b) -> [a] -> [b]
mapSoloWrong f = go
  where go [] = []
        go (x:xs) =
          -- let instead of case
          let MkSolo y = f x in y : mapSolo xs

mapSoloAlsoWrong :: (a -> Solo b) -> [a] -> [b]
mapSoloAlsoWrong f = go
  where go [] = []
        go (x:xs) = (case f x of MkSolo y -> y) : mapSolo xs
```

The difference may seem subtle but it is crucial! These definitions cannot be
strict, because the evaluation of `MkSolo` is not tied to the `(:)`.

This technique works for other functions too. Here's a left fold using `Solo`,
an example of the first category of strict functions.

```hs
foldlSolo :: (b -> a -> Solo b) -> b -> [a] -> b
foldlSolo f = go
  where go z [] = z
        go z (x:xs) = case f z x of MkSolo z' -> go z' xs
```

As with `map`, `foldl` and `foldl'` can be easily defined in terms of
`foldlSolo`.

```hs
-- lazy
foldl :: (b -> a -> b) -> b -> [a] -> b
foldl f z0 = foldlSolo (\z x -> MkSolo (f z x)) z0

-- strict
foldl' :: (b -> a -> b) -> b -> [a] -> b
foldl' f !z0 = foldlSolo (\z x -> MkSolo $! f z x) z0
```

## Boxes

`Solo` is not special. In general, the idea is that the function should return
some structure, let's call it a "box", with a value *inside* it, instead of
simply returning the value. The caller of this function must match on the box.
Whether this forces the value inside the box depends on how it was constructed.
Thus the function which constructs the box has control over the strictness.

<div markdown="1" class="block tip">

**TIP:**{: .tip-text} The fact that one can force a `Solo` without forcing the
value inside is also useful in other situations, such as array-indexing, as
explained in the
[docs for `Solo`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-Tuple.html#t:Solo).

</div>

The box we work with does not need to be `Solo`. Consider
[`Data.List.unfoldr`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-List.html#v:unfoldr).

```hs
unfoldr :: (b -> Maybe (a, b)) -> b -> [a]
unfoldr f = go
  where go b = case f b of
                 Just (a, b') -> a : go b'
                 Nothing -> []
```

Here the `(:)` depends on the `Just`, which works as a box just like `Solo`. One
can be strict in the `a` or the `b` or both by forcing them when constructing
the `Just`.

```hs
-- Fibonacci sequence, strictly generated
fibonacci :: [Integer]
fibonacci = unfoldr f (0, 1)
  where f (a, b) = let !c = a + b -- strict!
                   in Just (a, (b, c))
```

There are also higher-order functions that are necessarily strict in the result
of the input function. There is no point in using boxes for these.

```hs
filter :: (a -> Bool) -> [a] -> [a]
filter p = go
  where go [] = []
        go (x:xs) = case p x of
          False -> go xs
          True -> x : go xs -- (:) already depends on p x
```

## Caveats

While we don't *need* separate lazy and strict functions in favor of just one
function using `Solo`, one could, of course, want them anyway.

It is probably more convenient to use a fully lazy or fully strict version.
Now, this means one cannot have a mix of strictness, but that amount of control
is rarely necessary. Often strictness is not the primary concern anyway, and
wrestling with `Solo`s would only obscure what one wants to express.

GHC is very good at optimizing code to remove unwanted thunks, so from a
performance perspective it usually works out well with lazy functions.
[List fusion](https://wiki.haskell.org/index.php?title=Short_cut_fusion)
helps with list functions in particular. Try using `Data.List.map` in the
`map` benchmarks above, and it should perform better than the strict `map'`!
That's because `map` fuses with `sum` and GHC is able to both eliminate the
intermediate list and calculate the sum strictly.

Using `Solo` can also potentially have poorer performance because `Solo`, being
`data`, is heap-allocated. I say "potentially" because GHC will be able to
optimize away the allocation in many cases. To guarantee good performance we
have the option of using the
[unboxed 1-tuple](https://downloads.haskell.org/~ghc/9.12.2/docs/users_guide/exts/primitives.html#unboxed-tuples)
instead of `Solo`. However, this is a low-level construct which usually isn't
seen in public interfaces.

## Bonus examples

That concludes the primary point of this post, but there are a couple more
examples that I find interesting to examine.

### `foldlM`

This is the lazy *monadic* left fold
[`Data.Foldable.foldlM`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-Foldable.html#v:foldlM),
specialized to lists for simplicity.

```hs
foldlM :: Monad m => (b -> a -> m b) -> b -> [a] -> m b
foldlM f = go
  where go z [] = pure z
        go z (x:xs) = f z x >>= \z' -> go z' xs
```

<div markdown="1" class="block note">

**NOTE:**{: .note-text} If this function looks oddly familiar, that's because if
we specialize `m` to `Solo`, we get `foldlSolo`!

```hs
foldlSolo :: (b -> a -> Solo b) -> b -> [a] -> b
foldlSolo f z0 xs = case foldlM f z0 xs of MkSolo z -> z
```

This uses the `Monad` instance for `Solo`.

```hs
instance Applicative Solo where
  pure = MkSolo
  liftA2 f (MkSolo x) (MkSolo y) = MkSolo (f x y)

instance Monad Solo where
  MkSolo x >>= f = f x
```

</div>

How do we control the strictness when it comes to `foldlM`? In `foldlSolo` we
are able to do so because

* `f` constructs a box, the `Solo`
* `(>>=)` matches on it at every step

If we use another monad and these remain true, we have the same control.

```hs
sumWithLimit :: Integer -> [Integer] -> Maybe Integer
sumWithLimit !limit = foldlM f 0
  where
    f acc x
      | acc' <= limit = Just acc'
      | otherwise = Nothing
      where acc' = acc + x
```

Here we use `foldlM` with `Maybe` as the monad. `acc'` is evaluated by the
comparison before it is put in `Just`, the box, and `Maybe`'s `(>>=)` matches on
it at every step. Thus, both conditions are satisfied and this is a strict fold.

<div markdown="1" class="block tip">

**TIP:**{: .tip-text} This is a tangent but I cannot resist pointing out how the
choice of monad gives `foldlM` different properties.

* `foldlM` specialized to `Identity` is simply the lazy fold `foldl`.
* `foldlM` specialized to `Solo` lets us control the strictness.
* `foldlM` specialized to `Maybe` lets us control the strictness, but also lets
  us return early!
</div>

Now there are monads where the two properties above do not hold. For
[`Identity`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-Functor-Identity.html#t:Identity),
there is no box to speak of, since it is only a newtype, so `f` can never
construct one and `(>>=)` can never match on it. For
[lazy `Writer`](https://hackage.haskell.org/package/transformers-0.6.2.0/docs/Control-Monad-Trans-Writer-Lazy.html#t:Writer)
or [lazy `State`](https://hackage.haskell.org/package/transformers-0.6.2.0/docs/Control-Monad-Trans-State-Lazy.html#t:State)
there are boxes, but `(>>=)` does not match on them[^lazy].

To control the strictness when using such monads, we can introduce `foldlMSolo`.

```hs
foldlMSolo :: Monad m => (b -> a -> m (Solo b)) -> b -> [a] -> m b
foldlMSolo f = go
  where go z [] = pure z
        go z (x:xs) = f z x >>= \(MkSolo z') -> go z' xs
```

The key is to match on the `MkSolo` in the argument to `(>>=)`. But we can get
this behavior right in the `(>>=)` with the help of a *monad transformer*.

```hs
newtype SoloT m a = SoloT { runSoloT :: m (Solo a) }

instance Monad m => Monad (SoloT m) where
  m >>= f = SoloT (runSoloT m >>= \(MkSolo x) -> runSoloT (f x))
                                -- ^ match on the Solo

instance Monad m => Applicative (SoloT m) where
  pure = SoloT . pure . MkSolo
  liftA2 = liftM2

instance Monad m => Functor (SoloT m) where
  fmap = liftM
```

So we can use `foldlM` with `SoloT`, without needing `foldlMSolo`. We can even
define `foldlMSolo` using `foldlM`.

```hs
foldlMSolo :: Monad m => (b -> a -> m (Solo b)) -> b -> [a] -> m b
foldlMSolo f z0 xs =
  runSoloT (foldlM (\z x -> SoloT (f z x)) z0 xs) >>= \(MkSolo z) -> pure z
```

Lastly, it is worth mentioning that there are monads which have no values, such
as
[`Proxy`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-Proxy.html#t:Proxy).
A `Proxy a` cannot produce any `a`. Since there are no values, there is no point
in considering how one can be strict in them with `foldlM`.

### `fmap`

It would be nice to have a generalized version of `mapSolo`, which gives us
control over strictness when mapping over arbitrary `Functor`s.

```hs
fmapSolo :: Functor f => (a -> Solo b) -> f a -> f b
fmapSolo = ???
```

We've just seen how `foldl` is lazy but `foldlM`, its monadic version, lets
us control the strictness. Can we do the same for `fmap`? What's the monadic
version of `fmap` anyway?

Interestingly, there are *two* answers to that in `base`.

One is the appropriately named
[`mapM`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-Traversable.html#v:mapM),
or its less appropriately named cousin
[`traverse`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-Traversable.html#v:traverse).

```hs
fmap     :: Functor f                      => (a ->   b) -> f a ->    f b
mapM     :: (Traversable t, Monad m)       => (a -> m b) -> t a -> m (t b)
traverse :: (Traversable t, Applicative m) => (a -> m b) -> t a -> m (t b)
```

```hs
fmapSolo_viaTraversable :: Traversable t => (a -> Solo b) -> t a -> t b
fmapSolo_viaTraversable f t = case traverse f t of MkSolo t' -> t'
```

This does let us control the strictness, but it requires that the `Functor`
additionally be `Traversable`.

Note that it uses the instance `Applicative Solo`, where `liftA2` is strict in
both of its `Solo` arguments. Because of this, it will fully evaluate the spine
of the `Traversable`. This makes it different from the `mapSolo` we've seen,
and may not be what you want for lists or other structures with lazy spines.
`fmapSolo_viaTraversable` on an infinite list will never yield a value.

The other answer is our good friend
[`(>>=)`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Control-Monad.html#v:-62--62--61-).

```hs
fmap       :: Functor f => (a ->   b) -> f a -> f b
flip (>>=) :: Monad m   => (a -> m b) -> m a -> m b
```

Here we don't need to involve `Solo`. Instead we require that the `Functor` be
a `Monad` and `pure` construct the structure of the monad. So a strict `fmap`
looks like

```hs
fmap'_viaMonad :: Monad m => (a -> b) -> m a -> m b
fmap'_viaMonad f m = m >>= \x -> pure $! f x
```

In fact, `base` offers this as
[`Control.Monad.<$!>`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Control-Monad.html#v:-60--36--33--62-).

Recall that `map'` was in the second category of strict functions, where forcing
the structure forces the value inside. This makes sense for `Functors` like
lists, but if the `Functor` has no structure, one cannot be any more strict than
`fmap`. So using `fmapSolo_viaTraversable` or `fmap'_viaMonad` on
[`Identity`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-Functor-Identity.html#t:Identity)
makes no difference. The same is true if there is no value at all, as with
[`Const`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-Functor-Const.html#t:Const)
or
[`Proxy`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-Proxy.html#t:Proxy).

Both the `Traversable` and `Monad` ways require additional constraints, so
unfortunately we haven't found a general way to define `fmapSolo` for all
`Functor`s. `fmapSolo` could reasonably be a method of the `Functor` class, or
a subclass, since each type needs a suitable definition.

As an example, here's a `Functor` which cannot be `Traversable` and cannot
be `Monad` (without additional constraints), but we can nevertheless define an
`fmapSolo` for it.

```hs
newtype M i o a = M (i -> (o, a))

instance Functor (M i o) where
  fmap f (M g) = M (fmap (fmap f) g)

fmapSolo :: (a -> Solo b) -> M i o a -> M i o b
fmapSolo f (M g) = M $
  \i -> case g i of
    (o, x) -> case f x of
      MkSolo y -> (o, y)
```

## The end

TL;DR:

* We don't need lazy and strict versions the same function.
* Boxes are the key to controlling strictness with just one function.
* `Solo` can be the box, if there isn't one already.
* In certain cases, with polymorphic functions, one can plug in `Solo` as the
  `Applicative` or `Monad`.

Will we see lazy and strict functions in `base` replaced by functions using
`Solo`? Probably not, and as much as I like the idea, I don't see a compelling
need for such a change. But I wouldn't mind seeing libraries in the wild putting
`Solo` to use for strictness purposes.

While I'm not the first to think of using `Solo` to control strictness, I've
never seen it described in detail, so that's what this has been. Thank you to
the Haddocks for
[`Solo`](https://hackage.haskell.org/package/base-4.21.0.0/docs/Data-Tuple.html#t:Solo),
and to [`treeowl`](https://github.com/treeowl) for demonstrating `SoloT` in
[this comment](https://github.com/haskell/core-libraries-committee/issues/255#issuecomment-1945081537).

---

[^opt]: However, GHC is free to *optimize* the lazy function to force the value
    at each step.

[^map]: `map`'s definition here is not identical to the definition in `base`,
    which is designed to work well with list fusion. Since this is not relevant
    here, I will continue to use simpler definitions for `map` and
    other list functions.

[^lazy]: In fact, this is what distinguishes the lazy version of `Writer` or
    `State` from the strict version.
