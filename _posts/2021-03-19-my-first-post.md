---
layout: post
author: Robbert van Dalen
title: "An introduction"
---

#### FP And OO
I like strong typing and purely Functional Programming. I also like to model my business domain with objects that have 
identity and a well-defined lifecycle. 
That is why I prefer Scala, a programming language that elegantly marries both FP and Object Orientation.

#### Place Oriented Programming
However, there are two things I dislike, and that's *shared mutable state* and *boilerplate*.
Most OO programs mutate state *in place* and that always gives me a lot of headaches.

I'm not the only one who has a problem with mutable state:

* Pat Helland points out in [Immutability Changes Everything](https://queue.acm.org/detail.cfm?id=2884038):

> Accountants don't use erasers; otherwise they may go to jail. All entries in a ledger remain in the ledger.

* Rich Hickey calls it PLace Oriented Programming ([PLOP](http://www.infoq.com/presentations/Value-Values)).
  Rich's solution to managing state is via Software Transactional Memory 
  ([STM](https://en.wikipedia.org/wiki/Software_transactional_memory)), or by employing a database called 
  [Datomic](https://www.datomic.com).
  
* On the front-end side, they also came to the conclusion that immutability gives you real benefits: 
  [Redux](https://redux.js.org/faq/immutable-data), 
  [XState](https://xstate.js.org/docs/) and 
  [Observable Store](https://www.npmjs.com/package/@codewithdan/observable-store) 
  are diverse examples, and there are many, many 
  [more](https://github.com/markerikson/redux-ecosystem-links/blob/master/immutable-data.md#immutable-update-utilities).
  
* On the back-end side, [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) has been the most prominent
  approach to immutable state management, based on append-only streams of immutable events.
  Another notable example is [Akka](https://akka.io) which is both an Actor framework and an Event Sourcing framework.
  
* Early me with [Enchilada](http://www.enchiladacode.nl) and [Spread](https://github.com/odipar/spread) where I 
  experimented with *extreme* immutability.

* And of course Bitcoin's Blockchain, but I'm not going to spend any energy on that. 

[Manikin](https://github.com/odipar/jmanikin) - a new framework that I've been developing for the past 3 years - is another attempt that's more sympathetic 
to version control. I will discuss this 'version control' approach in more detail in separate future posts, because 
I think that's probably the most interesting capability of Manikin.

Before we dive into versions, let us first start with a couple of definitions and then slowly build towards 
the design and semantics of Manikin.

#### Pure Objects
Our first definition is that of an 'ordinary' object:

An ordinary object has (mutable - in place) state and identity. Given the same identity or *id*, an object can be at 
different states at different points in time, but its history is destroyed or not available.

Consequently, Manikin defines a *Pure* Object to be an object that has *immutable* state and *immutable* identity with
the optional capability that we do have (second-class) access to its history.

To *type* a Pure Object, we first relate identity `Id[O]` and state `O` with the following trait:
```scala
trait Id[O] {
  def init: O
}
```
(note that `Id[O]` is also factory for the pristine or initial state `O`).

Then, we type a Pure Object to be a single *pair* or *mapping* from `Id[O]` to `O`:
```scala
type PureObject[O] = (Id[O], O)
```
#### Transitions
As Pure Objects are defined to be immutable, can we also do something useful with them?

Yes we can, and the key is that we always create *new* immutable states: state is explicitly *transitioned* from old to new.

To stay with the *Accountants don't use erasers* example, here is an Account state transition system:
```scala
case class Account(balance: Double, interest: Double)

def deposit (ac: Account, amt: Double): Account = {
  if (amt <= 0) sys.error("amount should be positive")
  
  ac.copy(balance = ac.balance + amt)
}

def withdraw(ac: Account, amt: Double): Account = {
  if (amt <= 0) sys.error("amount should be positive")
  if (ac.balance < amt) sys.error("not enough balance")
  
  ac.copy(balance = ac.balance - amt)
}
```
With the aid of case classes and copy constructors, it is very easy to create new immutable states in Scala.
Not surprisingly, Kotlin's [Data Classes](https://kotlinlang.org/docs/data-classes.html) provide exactly the same
capability. 

Unfortunately, with Java it would require more boilerplate, 
but we can achieve more or less the same result with [Project Lombok](https://projectlombok.org)
or with the newfangled Java 16 [Records](https://openjdk.java.net/jeps/395).

#### Worlds
Now that we can transition old states to new states, we also have to manage the mapping between identifiers and 
states somewhere.

Normally we would use ordinary JVM references for that, as references are (mutable!) memory locations that 
act as identifiers. Alas, we cannot use references: our definition of the Pure Object forbids it.

Manikin provides a special kind of immutable object, called a *World*, to store and manage Pure Objects.

Using 'Worlds' to store and control program state is not a new concept. I've nicked it from the paper
[Worlds: controlling the scope of side-effects](http://www.vpri.org/pdf/tr2011001_final_worlds.pdf) co-written by 
Alan Kay, the guy who first coined the term 'Object Oriented Programming'.

To get an idea of how Worlds can help us control state, let us start with a very simple World that is just a wrapper 
around an immutable Map:
```scala
case class World(state: Map[Id[_], _]) {
  def get[O](id: Id[O]): O = state.getOrElse(id, id.init).asInstanceOf[O]
  def put[O](id: Id[O], st: O): World = World(state = state + (id -> st))
}
```
So, every time we put a Pure Object into a World, it returns a new version of a World (containing that object).
To accommodate for Worlds, we also define the following 'Wordly' transition functions:
```scala
type AccountId = Id[Account]

def transition[O](w: World, id: Id[O])(tr: O => O): World = {
  w.put(id, tr(w.get(id)))
}

def depositInWorld(w: World, id: AccountId, amt: Double): World = {
  transition(w, id)(ac => deposit(ac, amt))
}

def withdrawInWorld(w: World, id: AccountId, amt: Double): World = {
  transition(w, id)(ac => withdraw(ac, amt))
}
```
#### Composite Objects
Of course, things get more intricate when (composite) objects contain identifiers to other objects. 
And indeed, implementing transition functions for composite objects requires even more boilerplate, as we have
to explicitly pass new versions of a World to the next stages of a composite transition. 

Here is an example of a money transfer between two accounts:
```scala
type TransferId = Id[Transfer]

case class Transfer(done: Boolean, from: AccountId, to: AccountId, amt: Double)

def transfer(w: World, id: TransferId): World = {
  val tr = w.get(id)
  
  if (tr.done) sys.error("cannot transfer twice")
  
  val w0 = withdrawInWorld(w, tr.from, tr.amt)
  val w1 = depositInWorld(w0, tr.to, tr.amt)
  
  transition(w1, id)(t => t.copy(done = true))
}
```
So, in order to stay purely functional and immutable, we need to explicitly pass all these pesky Worlds around, 
with a big chance of making a stupid mistake. 
But most importantly, the resulting code looks awful: probably nobody would like to develop software like that.

#### Monads
Enter monads! Yeah, you've probably heard about them. The monad is *the* most important FP construct to
[*exactly*](https://www.microsoft.com/en-us/research/wp-content/uploads/1993/01/imperative.pdf) solve this problem.
It is interesting to see how the monadic approach can reduce boilerplate significantly
(but don't worry, you don't have to understand monads or apply them in Manikin).
               
When we write our transfer method 'in' the monad `W`, it would look something like this:
```scala
def transfer(i: W[TransferId]): W[Unit] =
  for {
    id   <- i
    tr   <- get(id)
         <- ifte (tr.done)     (sys.error("cannot transfer twice"))
    _    <- transition(tr.from)(ac => withdraw(ac, tr.amt))
    _    <- transition(tr.to)  (ac => deposit (ac, tr.amt))
    _    <- transition(id)     (tr => tr.copy(done = true))
  } yield()
}
```
... or close to that. We could even decide to throw in some fancy 
[Either](https://www.freecodecamp.org/news/a-survival-guide-to-the-either-monad-in-scala-7293a680006/) for exceptions 
or go full [ZIO](https://zio.dev).

The important point to take home is that, with monads, we don't have to directly deal with Worlds anymore.
We could even decide to mutate the (hidden) World in place, exactly like how the IO monad does it (but we won't!)

The issue with monads is that they are not well-supported in Java (as it lacks Scala's syntactic flatMap support). 
Even with support, monads add syntactic noise to code, for example when lifting values in and out of a monad.
Monads also tend to '[color](http://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)' 
the type signatures of your methods and functions, making them more complicated.

Lack of support and syntactic noise are the most important reasons why I've decided *against* monads.
Instead, I've come up with an alternative solution that's more or less the *inverse* of a monad.

#### Environments
Like in the monad case, we don't want to deal with Worlds directly, so we encapsulate them into a mutable
`Environment`. Although Environments introduce a bit of mutation, we *gain* ordinary imperative code. 

Environments hide immutable Worlds, but they do dispatch all Wordly matters to them, like this:
```scala
class Environment1 {
  private var world: World = _
  
  def get(id: Id[O]) = world.get(id)
  def transition[O](id: Id[O])(tr: O => O) = {
    world = world.set(id, tr(world.get(id)))
  } 
}

def transfer(e: Environment, tid: TransferId): Unit = {
  val tr = e.get(tid)
  
  if (tr.done) sys.error("cannot transfer twice")
  
  transition(tr.from)(ac => withdraw(ac, tr.amt))
  transition(tr.to)  (ac => deposit (ac, tr.amt))
  transition(tid)    (tr => tr.copy (done = true))
}
```

But there is one big caveat with this approach: Environments cannot be shared across different threads.
They should also not 'escape' the (thread) scope where they are used, or bad things will happen.

As I will explain in later posts, both issues are countered by Manikin's messaging and concurrency model.
#### Messages
We've already condensed our Account example into imperative code, while keeping our
Objects pure. But what's still missing in our approach is another important OO concept, and that's behaviour:

In Java, you would naturally specify and structure your application with classes and methods, but in Manikin, 
we've adopted the `Message` type.
You can think of a Message as an immutable data structure that packages a single method call with its parameters, return 
value and implementation. 

Here is a first (incomplete) version 1 of the Message type and an example:
```scala
trait Message1[O, R] {     
 def apply(self: Id[O], env: Environment1): R
}

case class Deposit1(amt: Double) extends Message1[Account, Unit] {
  def apply(self: Id[Account], env: Environment1) = {
    if (amt <= 0.0) sys.error("amount should be positive")
    env.transition(self)(ac => ac.copy(balance = ac.balance + amt))
  }  
}
```
This does the job, but we are going to further refactor the Message type in such a way that it also becomes 
amenable to Event Sourcing (as we will explain later).

Our first refactor is to lift the self parameter into the Environment. We then refactor checks into a separate
pre-condition stage and lift the transition function from the Environment into the Message type.

Next to that, we introduce an additional `effect` stage: this is the place where we can send Messages to other objects
and return an (optional) result value `R`.

```scala
trait Environment2[O] {
  def self: Id[O]
  def obj[O2](id: Id[O2]): O2
  def obj: O = obj(self)
  def send[O2, R2](id: Id[O2], msg: Message2[O2, R2]): R2
}

trait Message2[O, R] {
  def preCondition: Environment2[O] => Boolean
  def apply: Environment2[O] => O
  def effect: Environment2[O] => R
}

case class Deposit2(amt: Double) extends Message2[Account, Unit] {
  def preCondition = env => amt > 0.0
  def apply = env => env.obj.copy(balance = env.obj.balance + amt)
  def effect = env => { }
}
```
#### Local Message
What's annoying however, that an Environment still has to be passed as a parameter to every stage.

To remove that annoyance, we are going to temporarily store and retrieve the 'current' Environment parameter in a 
global, mutable *ThreadLocal* variable. The result is that a LocalMessage is both a Message *and* an Environment.
                                                 
You may feel this 'global variable' solution is too dirty. No worries, you are always free to extend a 
regular Message if you don't want anything to do with ThreadLocal (at the expense of more boilerplate).

```scala
val tenv = new ThreadLocal[Environment2[_, _]]()

trait LocalMessage2[O, R] extends Environment2[O] {
  protected def env = tenv.get().asInstanceOf[Environment2[O, R]]
  
  def self: Id[O] = env.self
  def obj[O2](id: Id[O2]): O2 = env.obj(id)
  def obj: O = env.obj
  def send[O2, R2](id: Id[O2], msg: Message2[O2, R2]): R2 = env.send(id, msg)
  
  def preCondition: Boolean
  def apply: O
  def effect: R
}

case class Deposit3(amt: Double) extends LocalMessage2[Account, Unit] {
  def preCondition = amt > 0.0
  def apply = obj.copy(balance = obj.balance + amt)
  def effect = { }
}
```

#### Post Conditions
The addition of post-conditions is our last improvement to our Message type. We've already
shortly introduced pre-conditions which are checks (or predicates) that must evaluate to `true` *before* a state 
transition is applied. 

But sometimes we also need to check whether certain conditions holds *after* we do a transition
(and after Messages have been sent to other objects in the effect stage).

Post-conditions are very useful when you want to make sure that your system is always in a consistent, validated and sane
state. 

Unfortunately, there aren't many programming languages out there that have first-class post-conditions.
[Eiffel](https://www.eiffel.org/doc/eiffel/Eiffel) was one of the first OO languages that included them. 
Post-conditions are also more likely to be found in formal languages, such as the Java Modeling Language 
([JML](https://en.wikipedia.org/wiki/Java_Modeling_Language)).

So why are post-conditions not more widely used? 

I think the most important reason is that post-conditions are difficult to implement on top of mutable state, where 
history is destroyed. To get an idea of how difficult it is to implement post-conditions I refer to
[Extensible Code Contracts for Scala](https://ethz.ch/content/dam/ethz/special-interest/infk/chair-program-method/pm/documents/Education/Theses/Rokas_Matulis_MA_report.pdf).

But why do post-conditions need history? Because a post-condition can refer to stuff *after* a state transition
but also to stuff *before* a transition: they have access to *historical* state.

Of course, in Manikin we don't have this problem because it is easy to access historical states via Worlds.
Indeed, implementing post-conditions correctly is one of the main reasons why Manikin came into existence.

For now, we won't go deeper into the subject, but in the last section of this post, you'll find some example 
post-conditions.

#### The Whole Enchilada
Given our discussion so far, we are ready to understand the (slightly abbreviated) core types that make up Manikin. 
The core types are a bit more involved than our previous examples, but hopefully it will be clear why they have their 
particular shape:
```scala
trait Id[O] {
  def init: O
}

trait Pre[I <: Id[O], O, R] { def pre(pre: => Boolean): App[I, O, R] }
trait App[I <: Id[O], O, R] { def app(app: => O      ): Eff[I, O, R] }
trait Eff[I <: Id[O], O, R] { def eff(eff: => O      ): Pst[I, O, R] }
trait Pst[I <: Id[O], O, R] { def pst(pst: => Boolean): Msg[I, O, R] }

trait Msg[I <: Id[O], O, R] {
  def pre: () => Boolean
  def app: () => O
  def eff: () => R
  def pst: () => Boolean
}

trait Env[I <: Id[O], O, R] extends Pre[I, O, R] {
  def self: I
  def obj: O = obj(self)
  def old: O = old(self)
  def old[O2](id: Id[_ <: O2]): O2
  def obj[O2](id: Id[_ <: O2]): O2
  def send[I2 <: Id[O2], O2, R2](id: I2): R2
}

trait World[W <: World[W]] {
  def old[O](id: Id[O]): WorldValue[W, O]
  def obj[O](id: Id[O]): WorldValue[W, O]
  def send[I <: Id[O], O, R](id: I, msg: Message[I, O, R]): WorldValue[W, R]
}

trait WorldValue[W <: World[W], V] extends World[W] {
  def world: W
  def value: V
}

trait Message[I <: Id[O], O, R] {
  def msg(env: Env[I, O, R]): Msg[I, O, R]
}

trait LMessage[I <: Id[O], O, R] extends Message[I, O, R] with Env[I, O, R] {
  def local: Msg[I, O, R]
}
```
So that's it: the whole core of Manikin in 42 lines of code. And as you can see, there is not much to it 
(although it took me three years to make it *that* simple, with dozens of tries, improvements and failures).

You may disagree that Manikin is simple. And yes, I've introduced some extra types, like `Pre` and `Msg`, and it 
is not immediately clear what they are for. Also, the World type looks very different from the previous version we discussed.

#### Fluent Builder
Let's unpack the types a bit to get a better understanding, starting with the 
[fluent builder](https://en.wikipedia.org/wiki/Fluent_interface) that is responsible for building `Msg` objects. 

Say we want implement `LMessage`. For that, we need to implement the `local` method which must return a `Msg` object.
So how do we build one?
                                                                                    
If you look closely at `Env`, you see it defines a `pre` method that takes a single function parameter. 
And as `LMessage` extends `Env`, we can call `pre` in the scope of `LMessage`.

In turn, the `pre` method returns an `App` object with a single method `app`, which, again, takes a single function 
parameter. We then continue to 'fluently' build a `Msg` object by providing one function at a time, for each stage. 
Here is an example:
```scala
trait AccountMsg extends LMessage[AccountId, Account, Unit]
trait TransferMsg extends LMessage[TransferId, Transfer, Unit]

case class Deposit(amt: Double) extends AccountMsg {
  def local = 
    pre { amt > 0 }.
    app { obj.copy(obj.balance + amt) }.
    eff { }.
    pst { obj.balance = old.balance + amt}
}

case class Withdraw(amt: Double) extends AccountMsg {
  def local =
    pre { amt > 0 && obj.balance > amt }.
    app { obj.copy(obj.balance - amt) }.
    eff { }.
    pst { obj.balance = old.balance - amt}
} 

case class Book(from: AccountId, to: AccountId, amt: Double) extends TransferMsg {
  def local =
    pre { !obj.done }.
    app { obj.copy(done = true) }.
    eff { send(from, Withdraw(amt)) ; send(to, Deposit(amt)) }.
    pst { 
      obj(from).balance + obj(to).balance == old(from).balance + old(to).balance 
    }
}
```
Also take a note on the post conditions, especially the Book post condition. There we state:

> After we Book a Transfer, the sum balance of both Accounts should stay the same.

How cool is that?                                                                                 
          
#### Alternatives
But why do we need a builder to implement the four stages of a Message? Can't we  just implement them with ordinary 
methods?

We could, but then it would end up like this:
```scala
case class Deposit(amt: Double) extends AccountMsg {
  def pre = { amt > 0 }
  def app = { obj.copy(obj.balance + amt) }
  def eff = { }
  def pst = { obj.balance = old.balance + amt }
}
```
.. which looks pretty nice in Scala, but the alternative in Java would introduce extra boilerplate:
```java
class Deposit implements AccountMsg {
  public final Double amount;
  public Deposit(Double amount) { this.amount = amount; }

  @Override public pre { return () -> amount > 0.0; }
  @Override public app { return () -> new Account(obj().balance + amount); }
  @Override public eff { return () -> null; }
  @Override public pst { return () -> obj().balance == old().balance + amount; }
}
```
#### Alternative Worlds
Now that we have all the pieces of the puzzle in place, we can finally leverage the real value that Manikin brings:
the alternative Worlds that can be *dynamically* applied at runtime for all kinds of interesting (infrastructural) 
behaviour. 

And most importantly, behaviour that is *independent* of our business logic, for example:

* We could implement a World that tests our business logic in all kinds of scenario's 
  (a light form of [Model Checking](https://en.wikipedia.org/wiki/Model_checking))
* Or inject mock objects into a World to further zoom into specific scenario's
  (a light form of [Dependency Injection](https://en.wikipedia.org/wiki/Dependency_injection))
* Or store Messages in an Event Store (apply Event Sourcing)
* Or build an Object Oriented Version Control System (like GIT).

I've already tried out all these use-cases without needing to change a single line of business logic. 
Of course, only the business logic that was built 'the Manikin way'.
#### Final Example
We didn't entirely finish our simple 'Bank' example. So for completeness’s sake I'm filling in the missing bits in the
final example that ends this post. 

In my next posts I'm going to explain in more detail how we can use Manikin to implement Version Control,
Event Sourcing and much more.


```scala
case class AccountId(iban: String) extends Id[Account] {
  def init = Account(0.0, 0.0)
}

case class TransactionId(id: Long) extends Id[Transaction] {
  def init = Transfer(false, null, null, 0.0)
}

case class Open(init: Double) extends AccountMsg {
  def local =
    pre { init > 0.0 }.
    app { obj.copy(obj.balance = init ) }.
    eff { }.
    pst { obj.balance = inint }
}

def main(arg: Array[String]) = {
  val a1 = AccountId("A1")
  val a2 = AccountId("A2")
  val t1 = TransactionId(1)
  
  val world = new SimpleWorld()
  
  val nworld = world.
          send(a1, Open(50.0)).
          send(a2, Open(80.0)).
          send(t1, Book(a1, a2, 30.0))
  
  println("a1: " + nworld.obj(a1).value) // Account(20.0)
  println("a2: " + nworld.obj(a2).value) // Account(110.0)
}
```



