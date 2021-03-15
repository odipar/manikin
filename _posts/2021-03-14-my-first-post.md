---
layout: post
title: "An introduction to Manikin"
---

### Scala
I like strong typing and purely functional programming. I also like to model my business domain with objects that have identity. 
That's why I prefer programming in Scala, a programming language that elegantly marries both FP and OO.

### Place Oriented Programming
However, there is one thing I dislike, and that's *shared* mutable state:
most OO programs mutate state - in place! 

It is a practice that Rich Hickey (from Clojure fame) calls [PLace Oriented Programming](http://www.infoq.com/presentations/Value-Values) (PLOP).
Clojure's solution to managing mutable state is via Software Transactional Memory (STM).

*Manikin*, the system I'm going to present in this post, employs a different approach that is more similar to version control like GIT.

### Pure Objects
My definition of an ordinary object is that it has (mutable) state and (mutable) identity. 
Given the same identity or id, an object can be at different states at different points in time.
Consequently, I define a *Pure* Object to be an object that has *immutable* state and *immutable* identity. 

Manikin relates an Id with state O with the following trait:

```scala
trait Id[O] {
  def init: O
}
```

Note that an Id is also factory for the pristine or initial state O.

### Transitions
So if Pure Objects are immutable, how can we do something useful with them?
The key is that we always create *new* immutable states.
We just create a method that implements a state *transition* from old to new.

Here is an example:

```scala
case class Account(balance: Double, interest: Double)

def deposit (s: Account, a: Double): Account = {
  s.copy(balance = s.balance + a)
}

def withdraw(s: Account, a: Double): Account = {
  if (s.balance > a) s.copy(balance = s.balance - a)
  else sys.error("not enough balance")
}

```
As you can see, it is very easy to create new immutable states in Scala, using case classes.
One could also use Kotlin's data classes, or project Lombok in the case of Java.

### Worlds
So we can map old states to new states, but we also need to keep track of the mapping between identifiers and states.
Normally, one would use ordinary JVM references, which are actually (mutable!) memory locations that act as identifiers.

In Manikin however, we use a special kind of objects - immutable *Worlds* - to store such mappings.

Using 'Worlds' to store and control object state (or program state) is not a new concept. I've nicked it from the paper
[Worlds: controlling the scope of side-effects](http://www.vpri.org/pdf/tr2011001_final_worlds.pdf) co-written by Alan Kay,
the guy who first coined the term 'Object Oriented Programming'.

To get an idea of how Worlds can help us control state, let's start with the most naive World which is just a wrapper around an immutable map:
```scala
case class World(val state: Map[Id[Any], Any]) {
  def get[O](id: Id[O]): O = {
    state.getOrElse(id, id.init).asInstanceOf[O]
  }
  
  def set[O](id: Id[O], obj: O): World = {
    World(state = state + (id -> obj))
  }
}
```

In turn, our 'wordly' transition functions would become:

```scala
case class AccountId(iban: String) extends Id[Account]

def depositW (w: World, id: AccountId, amt: Double): World = {
  w.put(id, deposit(get(i), amt))
}

def withdrawW(w: World, i: AccountId, amt: Double): World = {
  w.put(id, withdraw(get(i), amt))
}
```

### Composite Objects
Things get more involved when (composite) objects contain identifiers to other objects. 
Given our naive World, implementing transition functions for composite objects would require a lot of boilerplate. 

Here is an example:
     
```scala
case class TId(t: Long) extends Id[Transaction]
case class Transaction(
    done: Boolean, 
    from: AccountId, 
    to: AccountId, 
    amount: Double)

def book(w: World, id: TId): World = {
  val tr = w.get(id)
  
  if (tr.done) sys.error("cannot book twice")
  else {
    val w0 = withdrawW(w, tr.from, tr.amount)
    val w1 = depositW(w0, tr.to, tr.amount)
    w1.put(id, w1.get(i).copy(done = true))
  }
}
```

### Boilerplate
So, just to be purely functional and immutable, we need to explicitly pass these Worlds around, with a lot of boilerplate. 
Next to that, we could easily make a mistake, by passing the wrong one. 
Also, the code looks horrendous: probably no one would like to develop software like that.

### Monads
Enter monads, you've probably heard about them. 
Don't worry, you don't need to understand monads when using Manikin.
However, applying the monadic style would significantly reduce the boilerplate.

Here is an example of how that would look like:
                   
```scala
def book(i: W[TId]): W[Unit] =
  for {
    id   <- i
    tr   <- get(id)
    _    <- apply(tr.from, (a => withdraw(a, tr.amount)))
    _    <- apply(tr.to  , (a => deposit (a, tr.amount)))
    _    <- set(id, tr.copy(done = true))
  } yield()
}
```
... or something like that. 

The amount of boilerplate could be even further reduced with some additional tricks (like Scala implicits).

The important point that I'm trying to make is that we don't need to pass new versions of the world around in the monadic version.
We could even decide to mutate the world in place (like what the IO monad does), but we won't!

Also note, that monads are well-supported by Scala (flatMap syntax), but not so much by Java.
### Environments

### Messages
In Manikin, the Message trait is exactly such composable transition function.
Next to transitioning object states, we would like messages to return (optional) result values.
(compare this with Akka, where a message send doesn't return a result).
How would such Message look like? Here is a first try:

```scala
trait Message[I <: Id[O], O, R] {     
 def apply(id: I, w: World): (World, R)
}
```

### Guarding State

### 
Moreover, a Pure Object itself doesn't need to know its own Identity! (but other Pure Object obviously do).











