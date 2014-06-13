---
comments: true
date: 2014-06-09 18:00:00
layout: post
slug: sbt-process-runner
title: Running subprocesses from SBT console
summary: Preparing environment for integration tests is not easy. Usually you need to run one or more external services - database, rabbitmq, web server, etc. What's more, you have to be sure that they are up and running. After tests you have to be able to turn them off. My plugin makes it possible to start all required applications directly from SBT console with minimal effort.
author: Jan Ziniewicz
tags:
- SBT
- Scala
- Integration tests
---

### TL;DR

Preparing environment for integration tests is not easy. Usually you need to run one or more external services - database, rabbitmq, web server, etc. What's more, you have to be sure that they are up and running. After tests you have to be able to turn them off. My plugin makes it possible to start all required applications directly from SBT console with minimal effort.

Check it on Github: https://github.com/whysoserious/sbt-process-runner

or continue reading.

### Process definition

You 'll have to create your own process definition. Create an object extending [ProcessInfo](https://github.com/whysoserious/sbt-process-runner/blob/master/process-runner/src/main/scala/jz/io.scalac.processrunner/ProcessInfo.scala#L52) trait. There are two important pieces here:
```scala
def processBuilder: ProcessBuilder
```
defines how process should be started. For that you can use very convenient [scala.sys.process](http://scala-lang.org/api/2.10.4/index.html#scala.sys.process.package) API. 
```scala
def isStarted: Boolean
```
Many proccesses need much time to start. The method above is periodically ran by a plugin to check whether process is up and running. The easiest implementation is just:
```scala
override def isStarted: Boolean = true
```
but you can also check for existence of a PID file
```scala
override def isStarted: Boolean = {
  import java.nio.file._
  Files.exists(Paths.get("/tmp", "webappp"))
}
```
or test whether a certain port is open:
```scala
override def isStarted: Boolean = {
  try {
    new Socket("127.0.0.1", port).getInputStream.close()
    true
  } catch {
    case _: Exception => false
  }
}
```

You can see more examples in a [test-project](https://github.com/whysoserious/sbt-process-runner/blob/master/test-project%2Fproject%2FBuild.scala).

### Installation

Check manual on [Github](https://github.com/whysoserious/sbt-process-runner/blob/master/README.md)

### New commands with TAB COMPLETION

For each process you defined you 'll have three new commands in SBT console:

* `process-runner:status your-process-id`: check status of the process - `Idle`, `Starting` or `Running`.
* `process-runner:start your-process-id`: start process
* `process-runner:stop your-process-id`: stop process

All commands support **Tab completion**.

### Frustration

[SBT](http://www.scala-sbt.org/) is crap. Crap with long, boring and complicated documentation. With barely documented code. With complicated API (eg. why do we have separation between Tasks and InputTasks?!).

The only reason I use it is that it has great incremental compiler and there's no alternative for that. 

Sorry [guys](https://github.com/sbt/sbt/graphs/contributors), you made great job creating such powerful tool and utterly failed in making it user-friendly.
