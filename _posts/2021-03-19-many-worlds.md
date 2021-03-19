---
layout: post
author: Robbert van Dalen
title: ""
---

#### Version Control
Just like you, I like version control, especially when working on the same code within a team.

But why is it that we want tight version-control over our application *code*, but don't bother to have the same level 
of control over our application *state*?

What would happen if we would have the same level of version control over our application state,
like we have over with our code? 
I think it would be very liberating, at least, that's what I would like to show in this post.
                
#### Immutable Snapshots
Let's draw an analogy between source code and application state. Is there a difference?

Sure there is:

* Application source code is typically text that is stored in different files under various directories on a filesystem.
* While application state is typically a (big) set of objects, that have references to other objects and are stored in 
(volatile) computer memory.
  
But holistically, there is not difference: both are big blobs of digital data. And with
version control we would like to keep track of these blobs (also called immutable snapshots or commits). 

Interestingly enough, GIT exactly follows this big blob approach to version control.

#### Detail Level
GIT *stores* blobs as top-level commits to enable *actual* version control that happens at the detail 
level: with commits (and some simple bookkeeping) we can answer the following questions:

* Are there any new files in a commit, compared to its previous (parent) commit?
* Are there any differences between the lines of text?
* What are the differences between the three snapshot versions of the same file? (three-way merge)
* Who is the author of the commit?
* etc, etc

#### 


