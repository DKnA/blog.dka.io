---
title: "Lessons on Event Sourcing and CQRS"
date: 2017-12-05T22:00:39-06:00
draft: false
author: "Piotr Adamski"
authorLink: "https://twitter.com/mcveat"
className: "article"
---

Last year we were put to a task:

> Design and build a social network. There will be contests. There will be videos. There will be eternal glory and street cred.

Our backend team sat down and thought. No one broke the silence, yet everyone knew. CQRS and Event Sourcing. It's time.

# What was our plan?

We were tasked to create an application that allowed users to create video contests, from bike stunts to air guitars. Contest winners were chosen by votes, comments and attention. Current standings were determined in real time by a weighing algorithm. 

This highly collaborative model has significant differences in how it is read and written. Models like these are often used as an example for [CQRS](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs) application. The _segregating_ aspect of this pattern naturally fits the [microservice architecture](http://microservices.io/patterns/microservices.html). It allows building and scaling command and query parts separately. We're usually taking the microservice direction and this project wasn't meant to be different.

[Event Sourcing](https://docs.microsoft.com/en-us/azure/architecture/patterns/event-sourcing) is often recited in the same breath as CQRS. Rightfully so. Append only event journal makes a good write model while materialized views serve well as its query counterpart. Yet, I'd argue for its _append only_ property to be the most important here. It means recording every event that changes the applications state. If a user votes and then takes back his vote 10 times, each time it is recorded. It is crucial in cases like ours, where you live and die by awarding who's air guitar is the coolest. Hell, you can even change the rules of the game, replay all the events and see who would turn out the winner.

`akka-persistence` actors is the most straightforward abstraction over journal of events. That's why we chose it. Materialized views implemented as reactive streams were also hot at the time. We wanted to try it because we had previous experience with `akka-persistence` where we made plenty of mistakes. We learned enough to make this time right.

# How did it go?

We did it on time and our client was happy. We were mostly happy with how it went. We've learned a lot. Our biggest lesson was:

> Cassandra needs a lot of memory

Starting this project we had little experience with Cassandra. We've extended what we knew about other databases and ran with it. We assumed that, much like postgres, Cassandra would scale up rather monotonous and 1GB was ok to start with. Much later we learned that official support disregards instances with less then 8GB of RAM for how unstable they are. Knowing that would have saved us time spent snooping database connections dying out of the blue.

Another lesson we learned is to be conscientiousness.

> Be diligent about your actors and materializers safety

One unhandled life cycle message cost us a lot of nerves. On timeout, our persisted actors were creating snapshots of themselves and shutting down. They didn't wait for `SnapshotSaved` event. We didn't think we could do anything with it anyway. That wasn't a problem for a long time. At some point contests we were handling involved large and more complex graphs of actors. Each actor within the graph was generating an exception for unhandled `SnapshotSaved` event on shutdown. About 50 of them in a short time triggers a circuit breaker - a mechanism that for set amount of time drops all messages, giving the underlying resource chance to recover. The mystery of these triggers was magnified by the fact that it was happening when nothing was seemingly  happening. That's when our actors were timing out.

This extends to materializers too. It took us a few tries to make sure we handled all exceptions and we had a plan when they shut down anyway. Each time they were dying in the worst moment possible. You know what kind of failures I mean.

The next lesson is on breaking the rules.

> Don't leak data between materialized views

We've been sloppy. We thought we can contain it. We thought one time doesn't count. We let two view depend on one another. Soon after we found ourselves in a dependency graph problem space. Nothing very serious yet, but also not the problem we were there to solve.

And finally we learned:

> It's ok not to Event Source everything

It was a humbling experience to event source everything. From the distance of a year, I think we could do better with a more suitable architecture here and there, handling user sessions to name an example.

# Will we do it again?

Definitely! Despite me talking mostly about things that could go better, there was a ton that went right. We had plenty of well structured tests, CI and CD worked like a charm. I am very confident we've embraced the concept and gained enough experience to take CQRS and Event Sourcing to the next level. 

There are two things I'd like to try doing differently. I am tempted to move from Event Sourcing to Command Sourcing. The difference being that instead of persisting events, you're persisting commands. Commands that cannot be executed to generate events are still significant data. It might be important, that someone tried to log in and failed three times. You can learn about exploits amd how confusing the user interface and api are. You can easily monitor failure.

At the same time, I'd not only strive to keep materialized views separate, I'd try to make them as denormalized as possible. Since these views are meant to serve a very narrow set of use cases, why wouldn't you optimize storage to their needs? That makes them more performant and harder to reuse.