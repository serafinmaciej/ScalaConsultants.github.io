---
comments: true
date: 2014-06-06 16:10:00
layout: post
slug: geecon-report
title: GeeCON from Scala perspective
summary: At this year's GeeCON I was helping a little bit with the organization as a volunteer. Obviously, I was mainly interested in presentations related to Scala. Unfortunately, there were only two of those, "Keep it Simple with Scala" by Adam Warski and "DDDing with Akka Persistence" by Konrad Malawski.
author: Sławomir Wójcik
tags:
- conferences
---

At this year's GeeCON I was helping a little bit with the organization as a volunteer. Obviously, I was mainly interested in presentations related to Scala. Unfortunately, there were only two of those: "Keep it Simple with Scala" by Adam Warski and "DDDing with Akka Persistence" by Konrad Malawski.

Adam's presentation was divided into three parts. For those who had been at Scalar conference, the third part about Spray didn't introduce anything new.

Similarly, the first part about the [new async/await block in Scala](http://docs.scala-lang.org/sips/pending/async.html) wasn't very enlightening for those who had done the "Principles of Reactive Programming" course on Coursera. However, the presented example was really good and simple to grasp. I asked Adam whether he knows when this feature will enter to Scala library and he thinks that it will remain as a separate library. That would be disappointing, because in my opinion these kind of features which simplify code and improve readability are really important for languages that have so many complicated features. Maybe it's not as big as for-comprehensions, but still.... 

The second part was about [MacWire](https://github.com/adamw/macwire) - a dependency injection framework. Actually they advocate it not as a framework because it's only a library with a bunch of macros. Compared to Java frameworks like Spring or Guice it's better because wiring is done at compile time instead of runtime and it doesn't need any container to work. It seemed easy to use and a bit simpler than some other scala solutions like [SubCut](https://github.com/dickwall/subcut), [Scaldi](http://scaldi.org/Scaldi.html), dependency injection using implicits or the Cake pattern(especially with all it quirks like initialization issues, and - come on - dependency injection is a simple/basic thing, do we really need to use such a complicated solution with so much boilerplate to achieve that?). However, I'm not too enthusiastic about it, because for dependency injection I prefer to use plain old constructor injection with Scala default values reducing the clutter of constructing. I really believe it's the simplest solution out there. Of course, it has some disadvantages, modularization is probably harder to achieve than in some other solutions and it lacks scoping too but from my experience you don't need scoping that often.

Oh did I mention the whole presentation was live coding? It was. And a nice one too, I would say. Well, of course the code was somewhat prepared but so much knowledge in so little time couldn't be conveyed without some good preparation.


In Konrad's case I knew that it was going to be about akka persistence as the solution to event sourcing similar to his presentation at Scalar. I only hoped that he would show it in a broader DDD example. Unfortunately he didn't, although the example he showed was better than the one on his previous presentation. It was easier to understand and presented in more detail probably because he had twice as much time as on his Scalar presentation. Moreover, this time there was some time left for questions and discussion which was great.

I'm really happy that this kind of event sourcing solution is emerging in Akka because when I was investigating event sourcing solutions about a year ago there weren't many solutions out there. There was some Java framework Axon - which was pretty decent and some database specifically designed for event sourcing but unfortunately with driver for C# only. It's good too that Akka persistence supports many storage solutions(mongodb, hbase etc.) with decent performance. Well, at least that's what Konrad claims.

Overall I was disappointed by GeeCON Scala presentations. Not because they were bad, on the contrary, they were very good, especially for people quite new to Scala but unfortunately not for me and others from Scalar conference.


Among other presentations, well... some were interesting. Keynotes was boring for me as always, but it's hard to impress me with a keynote on every conference. I've been to many keynotes at lots of conferences and I think most are similar and they don't add much value to conferences. In devops world which I'm fond of, there was nice introduction to docker.io by Marek Goldmann. Lightning talks were rather disappointing, I really prefer discussion panels.

There were some really nice presentations in JavaScript field too. Hazem Saleh from IBM did a presentation of Jasmine, a JavaScript framework for writing tests. It was one of the best presentations this year, really pro. He did manage to present most of the features of the framework in a way that was easy to grasp. Despite the fact that I prefer to develop on the backend in 100% cases, I do like to code properly on the frontend side as well and keep the high standards of quality. Moreover, I don't fully agree with Michał Ostruszka who did another talk about JavaScript and its tools in general. He thinks that backend developers should never touch the frontend and frontend developers shouldn't touch the backend. Well, that's true if everyone in your team is a kind of person that hates to program on the frontend/backend, but I think there are a lot of people like me or developers specialized in the frontend but writing high quality code in the backend as well.

Sander Mak and Paul Bakker did a good presentation on different solutions to achieve modularity in JavaScript. They showed all solutions in general and concentrated on [RequireJS](http://requirejs.org/) which they are using in their project. There was some talk about AngularJS done by Marek Matczak. He did some simple examples and shared his experience from the project in which his team used AngularJS. It was fine introduction of best features of the framework.


From perspective of talks that I've been to, probably it wasn't the best edition of GeeCON but I think it was OK.

Next week I will be at 33degree. We are one of the conference partners. Roland Kuhn from typesafe will be presenting there so I hope for some interesting talks.