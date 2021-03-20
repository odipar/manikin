---
layout: post
author: Robbert van Dalen
title: "Concurrent Worlds"
---
We *depend* on version control, especially when we work on a code base that is managed by a team of people.

We need version control because of *concurrency*: we want team members to work on different parts of the 
source code, without blocking each other. 

But concurrency comes at a price: we have to be more careful when *committing* concurrent work.
With concurrency, we have to think about consensus and solving potential conflicts. Version control systems 
help us perform such difficult tasks.

But why don't we exercise the same level of control over *application state*? 
Surely the same concurrency patterns apply when dealing with state.

#### Snapshots
Let's draw a comparison between source code and application state. Is there a difference?

Sure there is:

* Source code is text that is typically stored in a hierarchical filesystem in different files and directories.
  
* Application state is typically a big pile of objects, with references to other (piles of) objects, stored in 
(volatile) computer memory.
  
Holistically however, there is no difference: both can be serialized into big blobs of digital data (also called 
snapshots or commits). [GIT](https://git-scm.com) exactly follows this snapshot approach to version control.

#### Detail Level

But while GIT only stores commits at the 'blob level', the *actual* version control happens at the 'bit 
level'. 

It is amazing that, with only snapshots (and some simple bookkeeping) we can already answer so many 
detailed questions about our source code:

* Are there any new files in a commit, compared to its previous (parent) commit?
* Are there any conflicts between two commits?
* What are the differences between the three versions of the same file? (three-way merge)
* Who is the author of the commit?

#### Application State

Then what are the questions we would like to ask when dealing with application state?

* Are there any new objects in this commit?
* What objects changed between commits?
* Are there any conflicts between objects?
* Are there any data dependencies that would be violated when we merge two commits?
            
As it turns out, some of these questions cannot be answered with a simple snapshot model, so
we need an alternative approach.

#### Manikin 
[Manikin](https://github.com/odipar/jmanikin) is a framework that enables you to precisely control concurrent 
application state that goes beyond GIT's capabilities.

I've been working on Manikin for over 3 years, but I have only recently started writing about the general design 
of Manikin to gather some feedback. So if you want to understand more about Manikin's design, please read 
[this](https://odipar.github.io/manikin/2021/03/19/my-first-post.html) first.
(feedback would be much appreciated!)

In this installment I will go into more detail on how to manage concurrent state with Manikin.

#### Banking
Let's start with a compelling example, which is an unrealistic banking application written in 
non-idiomatic Scala code:
```scala
class Account(var balance: Double = 0.0)

def transfer(from: Account, to: Account, amount: Double) = {
   if (amount <= 0.0)         sys.error("amount should be positive")
   if (from == to)            sys.error("same accounts")
   if (from.balance < amount) sys.error("not enough balance")
   
   from.balance -= amount
   to.balance   += amount
}
```
Banking is easy right? We have accounts with a balance, and we can transfer money between them.

But our example is already a source of many intricate concurrency bugs. (can you spot them?)

Not surprisingly, this example pops up in many textbooks that discuss concurrency. It demonstrates interesting 
concurrency problems that may not be immediately obvious.

Of course, the biggest source of bugs is when concurrent processes mutate data *in-place* without any coordination.
Installing locks is one way to combat such bugs, but locks also severely impacts performance: processes will start 
blocking each other in their attempt to acquire the same locks. 
                                      
#### Databases
Another option is the Multi Version Concurrency Control 
([MVCC](https://en.wikipedia.org/wiki/Multiversion_concurrency_control)) approach that is taken by many modern
databases such as Postgres and Oracle. The MVCC model is very similar to GIT's snapshot model, but not exactly. 

And with databases, there is also a wide range of concurrency trade-offs to consider: do we want 
[Serializable Snapshot Isolation](https://wiki.postgresql.org/wiki/SSI) 
(slower, but correct), or are we OK with just Snapshot Isolation (faster, but with caveats). Manikin gives 
you all the options that databases give you, but than on the application level.
                                     
#### Rewrite
Let's rewrite our example in the *Manikin* style, starting with accounts:
```scala
case class Account(balance: Double = 0.0)

case class AccountId(id: String) extends Id[Account] { 
   def init = Account() 
}

trait AccountMsg extends ScalaMessage[AccountId, Account, Unit]

case class Withdraw(amount: Double) extends AccountMsg {
   def scala = 
      pre { amount > 0 && obj.balance > amount }.
      app { obj.copy(balance = obj.balance - amount) }.
      eff { }.
      pst { obj.balance == old.balance - amount }
}

case class Deposit(amount: Double) extends AccountMsg {
   def scala =
      pre { amount > 0 }.
      app { obj.copy(balance = obj.balance + amount) }.
      eff { }.
      pst { obj.balance == old.balance + amount }
}
```
That's ... very different. What's going on? 

In Manikin we can only manipulate (immutable!) objects by dispatching messages via Worlds.
In turn, each message dispatch goes through 4 additional stages of execution:

1. __pre__(condition): checks whether conditions hold *before* the state transition
   
2. __app__(ly): *transitions* the current object state to a new object state 
   
3. __eff__(ect): *send* messages to other objects, and (optionally) return a *result*
   
4. __p__(o)__st__(condition): checks whether conditions hold *after* the state transition (with access to old state)

After success, an event is written to an append-only log, stating that the message dispatch was indeed successful.
As such, we can think of a message as being a small 4-stage mini-transaction.

                    
#### Deposit Money
So let's deposit some money into the world and see what happens:
```scala
val a1 = AccountId("A1")

val world = EventWorld().
   send(a1, Deposit(50)).
   send(a1, Withdraw(30)).
   world

println(world.obj(a1).value) // Account(20.0)
println(world)
/*
SENT Deposit(50.0) => AccountId(A1):0
  PRE:
  EFF: ()
  PST:
    READ AccountId(A1):1
    READ AccountId(A1):0

SENT Withdraw(30.0) => AccountId(A1):1
  PRE:
    READ AccountId(A1):1
  EFF: ()
  PST:
    READ AccountId(A1):2
    READ AccountId(A1):1
*/
```
As you can see, the EventWorld has captured all messages that have been sent and 'turned' them into events. 
Next to that, it also tracked all the *reads* that were issued in the pre- and post-conditions.

Also notice the versions numbers that are post-fixed to each account id. All this tracking and tracing happens in 
Manikin's 'backend'. Exactly because Manikin stores a full record of what happened, it is able to efficiently compare, 
merge and rebase concurrent state.
                 
#### Transfer Money
But before we are going to combine concurrent state, we extend our example with a money transfer:
```scala
case class TransferId(id: Long) extends Id[Unit] {
   def init = { }
}

trait TransferMsg extends ScalaMessage[TransferId, Unit, Unit]

case class Open(initial: Double) extends AccountMsg {
   def scala =
      pre { initial > 0 }.
      app { obj.copy(balance = initial) }.
      eff { }.
      pst { obj.balance == initial } 
}

case class Book(from: AccountId, to: AccountId, amount: Double) extends TransferMsg {
   def scala = 
      pre { from != to }.
      app { }.
      eff { send(from, Deposit(amount)); send(to, Withdraw(amount)) }.
      pst { obj(from).balance + obj(to).balance == old(from).balance + old(to).balance }
}
```
Notice that the Book message has a composite effect: it sends two additional messages to other accounts.

#### Book
Let's see what happens when we book a transfer:
```scala
val a1 = AccountId("A1")
val a2 = AccountId("A2")
val t1 = TransferId(1)

val world = EventWorld().
   send(a1, Open(50)).
   send(a2, Open(80)).
   send(t1, Book(a1, a2, 30)).
   world

println(world.obj(a1).value)  // Account(20.0)
println(world.obj(a2).value)  // Account(110.0)
println(world.events.iterator.next.prettyPrint)
/*
SENT Book(AccountId(A1),AccountId(A2),30.0) => TransferId(1):0
  PRE:
  EFF: ()
    SENT Withdraw(30.0) => AccountId(A1):1
      PRE:
        READ AccountId(A1):1
      EFF: ()
      PST:
        READ AccountId(A1):2
        READ AccountId(A1):1
    SENT Deposit(30.0) => AccountId(A2):1
      PRE:
      EFF: ()
      PST:
        READ AccountId(A2):2
        READ AccountId(A2):1
  PST:
    READ AccountId(A1):2
    READ AccountId(A2):2
    READ AccountId(A1):1
    READ AccountId(A2):1
 */
```
What we see is that Manikin also keeps a detailed record of composite (recursive) effects.

#### Conflicting Worlds
Until now, our examples didn't show any concurrency problems. That's because messages were dispatched in a
single *thread* of control. 

To complicate matters, we are going to spawn multiple concurrent (and immutable!) worlds and then 
try to reconcile (merge) them.

Merging worlds is easy when there are no conflicts, but what kind of conflicts can we actually expect?

1. __Write/Write__ conflicts: An object has *divergent* messages sent to it, so the resulting states 
   are assumed also to be divergent.
   
2. __Read/Write__ conflicts: An object's state has been read at a certain point, but has been *invalidated* by a more
   recent state.
   
#### Merging Worlds
Our first trivial example demonstrates the merging of two concurrent Worlds without any conflicts:
```scala
val world1 = EventWorld().
   send(a1, Open(50)).
   world

val world2 = EventWorld().
   send(a2, Open(80)).
   world

val world3 = world1 merge world2

println(world3.obj(a1).value)  // Account(50.0)
println(world3.obj(a2).value)  // Account(80.0)
```
As expected, there are no conflicts when merging, because either World operates on different accounts. 

But when we transfer money that target the same accounts, Manikin detects a Write/Write conflict:
```scala
val t1 = TransferId(1)
val t2 = TransferId(2)

val world4 = world3.
   send(t1, Book(a1, a2, 20)).
   world

val world5 = world3.
   send(t1, Book(a1, a2, 30)).
   world

try { 
   val world6 = world4 merge world5 // FAILS
}
catch { 
   case e: Exception => println(e) 
   // CANNOT MERGE: WRITE/WRITE INTERSECTION Set(AccountId(A1), AccountId(A2)) 
} 
```
And indeed, both world4 and world5 have transferred money between the same accounts, so their write *sets* intersect.

#### Rebasing Worlds
But not all is lost as Manikin also supports a *rebase* operation. Rebasing effectively replays or resends messages
from the point where Worlds diverge from their common ancestor, so it is more expensive than merging.

A standard practice is to first try a merge (fast path), and when that doesn't work, do a rebase (slow path).
When a rebase is successful, we are sure that the final merge will be successful too.
```scala
try {
   val world6 = world4 merge world5 // FAILS
} 
catch { 
   case e: Exception => {
      try {
         val world7 = world5 rebase world4  // SUCCEEDS
         val world8 = world4 merge world7   // SUCCEEDS
         
         println(world8.obj(a1).value) // Account(0.0)
         println(world8.obj(a2).value) // Account(130.0)
      }
      case e: Exception => println(e)
   }
} 
```
#### Write Skew
My last example in this installment demonstrates an interesting anomaly called *[write skew](http://justinjaffray.com/what-does-write-skew-look-like/)*. 
This happens when writes *depend* on reads that are invalidated by other concurrent writes.
          
It is a dirty little secret that a lot of databases are not capable of preventing write skew. That's because it 
requires a very high transaction level called Serializable. 

The good news is, Manikin is able to detect write-skew! Here is the (canonical) black/white marble example:
```scala
case class MarbleId(id: Long) extends Id[Marble] { def init = Marble() }
case class AlignId (id: Long) extends Id[Unit]   { def init = { } }

case class Marble(color: String = "blue")

trait MarbleMsg extends ScalaMessage[MarbleId, Marble, Unit]
trait AlignMsg  extends ScalaMessage[AlignId, Unit, Unit]

case class SetColor(color: String) extends MarbleMsg {
   def scala =
      pre { color != null }.
      app { obj.copy(color = color) }.
      eff { }.
      pst { obj.color == color }
}

case class AlignColors(m1: MarbleId, m2: MarbleId) extends AlignMsg {
   def scala =
      pre { m1 != null && m2 != null}.
      app { }.
      eff { if (obj(m1).color != obj(m2).color) send(m1, SetColor(obj(m2).color)) }.
      pst { true }
}
```
The write-skew anomaly happens when we try to align colors between marbles concurrently, but in a different order:

```scala
val m1 = MarbleId(1)
val m2 = MarbleId(2)
val a1 = AlignId(1)
val a2 = AlignId(2)

val world1 = EventWorld().
   send(m1, SetColor("black")).
   send(m2, SetColor("white")).
   world

val world2 = world1.
   send(a1, AlignColors(m1, m2)).
   world

val world3 = world1.
   send(a2, AlignColors(m2, m1)).
   world

try {
   val world4 = world2 merge world3 // FAILS (WRITE-SKEW)
}
catch {
   case e: Exception => println(e)
   // CANNOT MERGE: READ/WRITE INTERSECTION Set(MarbleId(2))
}
```
#### Conclusion
1. With Manikin we can version-control application state like we can version-control code with GIT.

2. But Manikin is more powerful than GIT because it tracks (semantic) read/write dependencies. This is something 
GIT cannot do because of its snapshot semantics. 

3. Manikin also has Serializable Snapshot Isolation (SSI) semantics, which is one of the highest transaction levels that
can be achieved.

So what's next? 

In my next post I am going to dive more into the details of the EventWorld implementation. 