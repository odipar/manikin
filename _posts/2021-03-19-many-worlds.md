---
layout: post
author: Robbert van Dalen
title: "Concurrent Worlds"
---

#### Version Control
I heavily *depend* on version control, especially when working on a code base in a team together.

We need version control because of *concurrency*: we want team members to work on different parts of the 
source code, without blocking each other. 

But concurrency comes at a price: we have to be much more careful when *committing* concurrent work.
With concurrency, we have to think about reaching consensus and solving potential conflicts. That's why we want 
 version control over our code.

But why don't we bother to have the same level of control over application *state*? 
Surely, similar concurrency patterns also apply for state. What would happen if we could wield
the same power over state as we currently wield over code?

I think it would be very liberating.
                
#### Snapshots
Let's draw an analogy between source code and application state. Is there a difference?

Sure there is:

* Source code is text that is typically stored in a hierarchical filesystem in different files and directories.
* Application state is typically a big pile of objects, with references to other (piles of) objects, stored in 
(volatile) computer memory.
  
Holistically however, there is no difference: both can be serialized into big blobs of digital data and with
version control we keep track of these blobs (also called snapshots or commits). 

GIT exactly follows this snapshot approach to version control.

#### Detail Level

But GIT only stores commits at the 'top level'. The *actual* version control happens at the 'detailed 
level'. 

It is amazing that, with just GIT commits (and some simple bookkeeping), we can answer so many 
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
we need a different approach.

#### Manikin 
Manikin is a framework that enables you to precisely control concurrent application state.

I've been working on Manikin for the past 3 years, but I recently started writing about the general design 
of Manikin to gather some feedback. If you want to understand more about Manikin's design, please read this post first.

In this post I will go into more detail on how Manikin can be leveraged for all kinds of concurrency patterns.

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

def close(accounts: Seq[Account]): Double = {
  if (accounts.size == 0) sys.error("no accounts")      
  
  accounts.map(_.balance).sum
}

```
Banking is easy right? We have accounts with a balance, and we can transfer money between accounts.
We also want to be able to 'close the book' by the end of the year: here is where all the balances should add up.

But our example is already a source of many intricate concurrency bugs. Not surprisingly, this 
example pops up in many textbooks that discusses concurrency. It demonstrates interesting concurrency problems
that may not be immediately obvious.

Of course, the biggest source of bugs is when concurrent processes mutate data *in-place* without any coordination.
Using locks is one way to combat such bugs, but locks also severely impacts performance: processes will start 
blocking each other in their attempt to acquire the same locks. 
                                      
#### Databases
Another approach is Multi Version Concurrency Control (MVCC) which is taken by modern
databases such as Postgres and Oracle. The MVCC model is very similar to GIT's snapshot model, that we 
discussed earlier, but not exactly. 

And with databases, there is also a wide range of concurrency trade-offs to consider: do we want Full Serializability 
(slower, but correct), or are we OK with just Snapshot Isolation (faster, but with caveats)? Manikin gives 
you the choice, but than on the application level.
                                     
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
    pst { obj.balance == old.balance + amount  }
}
```
That's ... very different. What's going on? 

In Manikin you can only manipulate (immutable!) objects by dispatching Messages to them via Worlds.
Each message dispatch goes through 4 stages of execution:

1. __pre__(condition): checks whether conditions hold *before* the state transition
2. __app__(ly): *transitions* the current object state to a new object state 
3. __eff__(ect): *send* messages to other objects, and (optionally) return a result. 
4. __p__(o)__st__(condition): checks whether conditions hold *after* the state transition (with access to old state)

When each stage is executed successfully, an event is optionally written to an append-only log as a witness of that *fact*.

In a way, you can think of a message as being a small mini-transaction. You could also interpret a message as a method 
call turned into an (immutable!) data structure.

So let's try this out with the predefined EventWorld to see what's happening behind the scenes:
```scala
  val a1 = AccountId("A1")
  val a2 = AccountId("A2")

  val world = EventWorld().
    send(a1, Deposit(30)).
    send(a2, Deposit(80))

  println(world)
```
```scala
case class TId(id: Long) extends Id[Unit]
trait TransferMsg extends ScalaMessage[TId, Unit, Unit]
trait CloseMsg extends ScalaMessage[TId, Unit, Double]

case class Book(from: AccountId, to: AccountId, amount: Double) extends TransferMsg {
  def scala = 
    pre { from != to}.
    app { }.
    eff { send() }.
    pst { obj(from).balance + obj(to).balance == old(from).balance + old(to).balance }
}

case class Close(accounts: Seq[AId]) extends TransferMsg {
  def scala =
    pre { accounts.size > 0 }.
    app { }.
    eff { account.map(id => obj(id).balance).sum }.
    pst { true }
}
```

