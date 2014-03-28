---
layout: post
title: "backgroud part I : akka actor"
date: 2013-01-26 17:33:11 +0800
comments: true
categories: actor code-digger
---

#Actor Systems in akka

Hierarchical Structure: One actor, which is to oversee a certain function in the program might want to split up its task into smaller, more manageable pieces.
<!-- more -->

_taopier: each actor has exactly one supervisor, which is the actor that created it._

## what's an ackor

An actor is a container for State, Behavior, a Mailbox, Children and a Supervisor Strategy. All of this is encapsulated behind an Actor Reference.

**Actor Reference** : actors are represented to the outside using actor references, which are objects that can be passed around freely and without restriction.

_taopier: restarting an actor without needing to update references elsewhere_

**State** : Actor objects will typically contain some variables which reflect possible states the actor may be in. This can be an explicit state machine (e.g. using the FSM module), or it could be a counter, set of listeners, pending requests, etc. 

what's [FSM](http://doc.akka.io/docs/akka/snapshot/scala/fsm.html#fsm-scala): The FSM (Finite State Machine) is available as a mixin for the akka Actor and is best described in the Erlang design principles
```
A FSM can be described as a set of relations of the form:

State(S) x Event(E) -> Actions (A), State(S')

These relations are interpreted as meaning:

If we are in state S and the event E occurs, we should perform the actions A and make a transition to the state S'.
```

**Behavior** : Every time a message is processed, it is matched against the current behavior of the actor. Behavior means a function which defines the actions to be taken in reaction to the message at that point in time

**Mailbox** : The piece which connects sender and receiver is the actor’s mailbox: each actor has exactly one mailbox to which all senders enqueue their messages. 

_taopier: the default strategy is FIFO, and can be set using Priority( the current behavior must always handle the next dequeued message, there is no scanning the mailbox for the next matching one. so this strategy cat actually put messages into front)_

**Children**: Each actor is potentially a supervisor: if it creates children for delegating sub-tasks, it will automatically supervise them. The list of children is maintained within the actor’s context and the actor has access to it. Modifications to the list are done by creating (context.actorOf(...)) or stopping (context.stop(child)) children and these actions are reflected immediately. The actual creation and termination actions happen behind the scenes in an asynchronous way, so they do not “block” their supervisor.

**Supervisor Strategy** : handling faults of its children

## Supervision and Monitoring

### what supervision means
When a subordinate detects a failure (i.e. throws an exception), it suspends itself and all its subordinates and sends a message to its supervisor, signaling failure.the supervisor has a choice of the following four options:

1. Resume the subordinate, keeping its accumulated internal state
2. Restart the subordinate, clearing out its accumulated internal state
3. Terminate the subordinate permanently
4. Escalate the failure

_taopier: the default behaviour of the preRestart hook of the Actor class is to terminate all its children before restarting, but this hook can be overridden_

### The Top-Level Supervisors

**/user: The Guardian Actor** The actor which is probably most interacted with is the parent of all user-created actors, the guardian named "/user". Actors created using system.actorOf() are children of this actor.

**/system: The System Guardian**  in order to achieve an orderly shut-down sequence where logging remains active while all normal actors terminate

**/: The Root Guardian** The root guardian is the grand-parent of all so-called “top-level” actors and supervises all the special actors mentioned in Top-Level Scopes for Actor Paths using the SupervisorStrategy.stoppingStrategy, whose purpose is to terminate the child upon any type of Exception.

### what restarting means

The precise sequence of events during a restart is the following:

1. suspend the actor (which means that it will not process normal messages until resumed), and recursively suspend all children
2. call the old instance’s preRestart hook (defaults to sending termination requests to all children and calling postStop)
3. wait for all children which were requested to terminate (using context.stop()) during preRestart to actually terminate; this—like all actor operations—is non-blocking, the termination notice from the last killed child will effect the progression to the next step
4. create new actor instance by invoking the originally provided factory again
5. invoke postRestart on the new instance (which by default also calls preStart)
6. send restart request to all children which were not killed in step 3; restarted children will follow the same process recursively, from step 2
7. resume the actor

###supervision strategies

two classes of supervision strategies which come with Akka: OneForOneStrategy(by default) and AllForOneStrategy.

_difference: the former applies the obtained directive only to the failed child, whereas the latter applies it to all siblings as well._

##Actor References, Paths and Addresses

{% img /images/code/entity-of-actor.png entity of actor %}

The above image displays the relationship between the most important entities within an actor system, please read on for the details.

**What is an Actor Reference?**

An actor reference is a subtype of ActorRef, whose foremost purpose is to support sending messages to the actor it represents.

**What is an Actor Path?**
```
"akka://my-sys/user/service-a/worker1"                   // purely local
"akka.tcp://my-sys@host.example.com:5678/user/service-b" // remote
"cluster://my-cluster/service-c"                         // clustered (Future Extension)
```

**Logical Actor Paths / Physical Actor Paths:** 

like our system: /home/taopier/ vs ../taopier/

**How are Actor References obtained?**

Creating Actors by using ActorSystem.actorOf method or using ActorContext.actorOf

Looking up Actors by Concrete Path: ActorSystem.actorFor or ActorContext.actorFor

You can for example send a message to a specific sibling:
```
context.actorFor("../brother") ! msg
```
Absolute paths may of course also be looked up on context in the usual way, i.e.
```
context.actorFor("/user/serviceA") ! msg
```
will work as expected.

**Querying the Logical Actor Hierarchy**

using the ActorSystem.actorSelection and ActorContext.actorSelection methods and do support sending messages:
```
context.actorSelection("../*") ! msg
```
will send msg to all siblings including the current actor.

**The Interplay with Remote Deployment**

When an actor creates a child, the actor system’s deployer will decide whether the new actor resides in the same JVM or on another node. In the second case, creation of the actor will be triggered via a network connection to happen in a different JVM and consequently within a different actor system. The remote system will place the new actor below a special path reserved for this purpose and the supervisor of the new actor will be a remote actor reference (representing that actor which triggered its creation). In this case, context.parent (the supervisor reference) and context.path.parent (the parent node in the actor’s path) do not represent the same actor.


**Top-Level Scopes for Actor Paths**

At the root of the path hierarchy resides the root guardian above which all other actors are found; its name is "/". The next level consists of the following:
```
"/user" is the guardian actor for all user-created top-level actors; actors created using ActorSystem.actorOf are found below this one.
"/system" is the guardian actor for all system-created top-level actors, e.g. logging listeners or actors automatically deployed by configuration at the start of the actor system.
"/deadLetters" is the dead letter actor, which is where all messages sent to stopped or non-existing actors are re-routed (on a best-effort basis: messages may be lost even within the local JVM).
"/temp" is the guardian for all short-lived system-created actors, e.g. those which are used in the implementation of ActorRef.ask.
"/remote" is an artificial path below which all actors reside whose supervisors are remote actor references
```

# Reference

http://www.gtan.com/akka_doc/intro/getting-started-first-scala.html

http://doc.akka.io/docs/akka/snapshot/
