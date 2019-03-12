# Chapter 2 From State to Monad Transformers

data61 has an amazing Haskell course [fp-course](https://github.com/data61/fp-course). Brian McKenna has a serial of wonderful [videos](https://www.youtube.com/playlist?list=PLly9WMAVMrayYo2c-1E_rIRwBXG_FbLBW) walk through the course.

This post aims to cover <em>State</em> course which is skipped in videos.

## Basics

First, let's look what is <code>State</code>.

<code>
newtype State s a = State { runState :: s-> (a, s)}
</code>

In case you are not familiar with [<code>newtype</code>](https://wiki.haskell.org/Newtype). It is just like `data` but it can only *has exactly one constructor with exactly one field in it.* In this case `State` is the constructor and `runState` is the single field. The type of `State` constructor is `(s ->  a, s)) -> State s a`, since `State` uses record syntax, we have a filed accessor `runState` and its type is State s a -> s -> (a, s).
<blockquote>A State is a function from a state value <em><strong>s</strong></em> to (a produced value <strong><em>a</em></strong>, and a resulting state <strong><em>s</em></strong>).</blockquote>
An importance takeaway which might not so oblivious to Haskell beginner is that: `State` is merely a function with type `s -> (a,s)`, (a function wrapped inside constructor `State`, to be exact, and we can unwrap it using `runState`).

Secondly, `State` function takes a `s`representing a state, and produces a tuple contains value `a` and new state `s`.

### exec

The first exercise is to implement `exec` function, it takes a `State` function and initial `state` value, returns the new state.

It is pretty straightforward, we just need to apply `State` function with `state` value, and takes the second  element from the tuple.

A simple solution can be

```haskell
exec :: State s a -> s -> s
exec f initial = (\ (_, y) -> y) (runState f initial)
```

Since the state value `initial` appears on both sides of `=`, we rewrite it a little more point-free. `runState f` is a partial applied function takes `s` returns `(a,s)`

```haskell
exec f = (\(_, y) -> y) . runState f
```

In `Prelude`, there is a [`snd`](http://hackage.haskell.org/package/base-4.11.1.0/docs/Prelude.html#v:snd) function does exactly the same thing of `\(_, y) -> y)`.

```haskell
exec f = P.snd . runState f
```

We could also write `exec`

`exec (State f) = P.snd . f`


### Basic Usage of State

```haskell
module Dice where

import Control.Applicative
import Control.Monad.Trans.State
import System.Random

rollDiceIO :: IO (Int, Int)
rollDiceIO = liftA2 (,) (randomRIO (1, 6)) (randomRIO (1,6))

rollNDiceIO :: Int -> IO [Int]
rollNDiceIO 0 = pure []
rollNDiceIO count = liftA2 (:) (randomRIO (1, 6)) (rollNDiceIO (count - 1))

clumsyRollDice :: (Int, Int)
clumsyRollDice = (n, m)
    where
        (n, g) = randomR (1, 6) (mkStdGen 0)
        (m, _) = randomR (1, 6) g

-- rollDice :: StdGen -> ((Int, Int), StdGen)
-- rollDice g = ((n, m), g'')
--     where
--         (n, g') = randomR (1, 6) g
--         (m, g'') = randomR (1, 6) g'

-- use state to construct
rollDie :: State StdGen Int
rollDie = state $ randomR (1, 6)

-- use State as Monad
rollDieM :: State StdGen Int
rollDieM = do generator <- get
              let (value, generator') = randomR (1, 6) generator
              put generator'
              return value

rollDice :: State StdGen (Int, Int)
rollDice = liftA2 (,) rollDieM rollDieM

rollNDice :: Int -> State StdGen [Int]
rollNDice 0 = state (\s -> ([], s))
rollNDice count = liftA2 (:) rollDieM (rollNDice (count - 1))
```

How about draw card from a deck

```haskell
module Deck where

import Control.Applicative
import Control.Monad.Trans.State
import Data.List
import System.Random

data Rank = One | Two | Three | Four | Five | Six | Seven | Eight | Nine | Ten | Jack | Queue | King deriving (Bounded, Enum, Show, Eq, Ord)

data Suit = Diamonds | Clubs | Hearts | Spades deriving (Bounded, Enum, Show, Eq, Ord)

data Card = Card Suit Rank deriving (Show, Eq, Ord)

type Deck = [Card]

fullDeck :: Deck
fullDeck = [Card suit rank  | suit <- enumFrom minBound,
                              rank <- enumFrom minBound]

removeCard :: Deck -> Int -> Deck
removeCard [] _ = []
removeCard deck index = deck' ++ deck''
    where (deck', remain) = splitAt (index + 1) deck
          deck''    = drop 1 remain


drawCard :: State (StdGen, Deck) Card
drawCard = do (generator, deck) <- get
              let (index, generator') = randomR (0, length deck ) generator
              put (generator', removeCard deck index)
              return $ deck !! index

drawNCard :: Int -> State (StdGen, Deck) [Card]
drawNCard 0 = state (\s -> ([], s))
drawNCard count = liftA2 (:) drawCard (drawNCard $ count - 1)
```

How about folding a list using `State`
https://github.com/yuanw/applied-haskell/blob/2018/monad-transformers.md#how-about-state

```haskell
foldState :: (b -> a -> b) -> b -> [a] -> b
foldState f accum0 list0 =
    execState (mapM_ go list0) accum0
  where
    go x = modify' (\accum -> f accum x)
```


Why we cannot compose any two monad

```haskell
import Control.Applicative

newtype Compose f g a = Compose (f (g a))

instance (Functor f, Functor g) => Functor (Compose f g) where
    fmap f (Compose h) = Compose ((fmap . fmap) f h)


instance (Applicative f, Applicative g) => Applicative (Compose f g) where
    pure = Compose . pure . pure
    (Compose h)  <*> (Compose j) = Compose $ pure (<*>) <*> h <*> j

instance (Monad f, Monad g) => Monad (Compose f g) where
     (Compose fga) >>= m = Compose $ fga >>= \ ga -> let gfgb = ga >>= (return . m) in undefined
```

https://stackoverflow.com/questions/7040844/applicatives-compose-monads-dont


## Build a composed monad by hand

```haskell
newtype StateEither s e a = StateEither
    { runStateEither :: s -> (s, Either e a)
    }
```

Let's implement the functor instance of this typeP


## mtl style typeclassess

[MonadState](http://hackage.haskell.org/package/mtl-2.2.2/docs/Control-Monad-State-Lazy.html)

```haskell
class Monad m => MonadState s m | m -> s where
    get :: m s
    put :: s -> m ()
```

## Exercise 1 Write ReaderT

```haskell
module Exercise1 where

import Control.Monad.Trans.Class
import Control.Monad.IO.Class
import Data.Functor.Identity

newtype ReaderT r m a = ReaderT {runReaderT :: r -> m a}

instance Functor m => Functor (ReaderT r m) where
    fmap f (ReaderT g) = ReaderT ( fmap f . g)

instance Applicative f => Applicative (ReaderT r f) where
    pure a = ReaderT (\ r -> pure a)
    ReaderT g <*> ReaderT h = ReaderT (\ r -> g r <*> (h r))

instance Monad m => Monad (ReaderT f m) where
    ReaderT g >>= f = ReaderT (\ r -> g r >>= ( \ a -> runReaderT (f a) r))

instance MonadTrans (ReaderT r) where
    lift action = ReaderT $ const action

instance (MonadIO m) => MonadIO (ReaderT r m) where
    liftIO = lift . liftIO

type Reader r = ReaderT r Identity

runReader :: Reader r a -> r -> a
runReader r = runIdentity . runReaderT r

ask :: Monad m => ReaderT r m r
ask = ReaderT pure

-- main :: IO ()
-- main = runReaderT main' "Hello World"

main' :: ReaderT String IO ()
main' = do
    lift $ putStrLn "I'm going to tell you a message"
    lift $ putStrLn "The message is:"
    message <- ask
    lift $ putStrLn message
```

https://ocharles.org.uk/posts/2014-12-14-functional-dependencies.html


http://hackage.haskell.org/package/unliftio

## Using ExceptT

http://hackage.haskell.org/package/mtl-2.2.2/docs/Control-Monad-Except.html#g:3


## References
1. https://en.wikibooks.org/wiki/Haskell/Understanding_monads/State
2. https://haskell.fpcomplete.com/library/rio
