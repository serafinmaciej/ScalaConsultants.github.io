---
comments: true
date: 2014-06-22 15:55:55
layout: post
slug: rabbit-with-akka-stream
title: RabbitMQ with Akka Stream
summary: Showing how Akka Streams play nicely with RabbitMQ.
author: Jakub Czuchnowski
tags:
- Akka
- akka-stream
- RabbitMQ
---

Akka-stream is an exciting new technology from Typesafe that is an implementation of the Reactive Streams specification (link here).
Write something about RabbitMQ.

##ActorProducer as RabbitMQ consumer
First thing we have to do is to somehow get the messages from a RabbitMQ server. As we are using the official Java client, we have two ways to do this - pull or push.

###the pull

~~~ scala

~~~

###the push

~~~ scala
  val consumer = new DefaultConsumer(channel) {
    override def handleDelivery(
        consumerTag: String, 
        envelope: Envelope, 
        properties: AMQP.BasicProperties, 
        body: Array[Byte]) = {
      self ! new RabbitMessage(envelope.getDeliveryTag(), new String(body, "UTF-8"), channel)
    }
  }
~~~

~~~ scala
  override def receive = {
    case msg: RabbitMessage => 
      if (isActive && totalDemand > 0) {
        onNext(msg)
      } else {
        msg.nack()
      }
  }
~~~

##Starting the Flow

~~~ scala
  implicit val actorSystem = ActorSystem("rabbit-akka-stream")

  val rabbitConsumer = ActorProducer(actorSystem.actorOf(new RabbitConsumerActor))

  val flow = Flow(rabbitConsumer)
~~~

Now have a flow, but it doesn't do anything yet. So...

##Ducting the Flow

In akka-stream 0.2, the only way of composing flow processing was through Flow[In]=>Flow[Out] functions. That's ok, but we could do better. And of course Akka guys did do better. In akka-stream 0.3 they've introduced ~Duct~'s. Duct is a placeholder for your transformations that can be created independently from the Flow and later attached to it. The signature is ~Duct[In, Out]~. As you might have guessed, ~In~ is a type that enters the Duct and ~Out~ is the type that leaves. Here's an example of a Duct:

~~~ scala
  val duct: Duct[RabbitMessage, String] = Duct[RabbitMessage].
    map { msg =>
      msg.ack()
      msg
    }.
    map { _.body }.
    map { msg => 
      logger.info(msg)
      msg 
    }
~~~

We create the Duct by a call to a companion object with an input type - Duct[RabbitMessage]. At this point our Duct has a type of Duct[RabbitMessage, RabbitMessage]. Later on we are doing some transformations using the ~map~ operator. The transformed message that leaves our Duct is a String. So the final type is ~Duct[RabbitMessage, String]~.

Ok, but still this doesn't do anything. So again...

##Putting it together

~~~ scala
  val materializer = FlowMaterializer(MaterializerSettings())

  flow append duct consume(materializer)
~~~