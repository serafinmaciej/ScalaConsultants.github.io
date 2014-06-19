---
comments: true
date: 2014-06-22 15:55:55
layout: post
slug: rabbit-with-akka-streams
title: RabbitMQ with Akka Streams
summary: Shows how Akka Streams play nicely with RabbitMQ and provides interesting new way of working with RabbitMQ messages.
author: Jakub Czuchnowski
tags:
- Scala
- Akka
- Akka Streams
- RabbitMQ
---

Akka Streams is an exciting new technology from [Typesafe](http://www.typesafe.com/) that is an implementation of the [Reactive Streams](http://www.reactive-streams.org/) specification. To this point two other implementations have been declared (from the [Reactor team](https://github.com/reactor/reactor) and Netflix's [RxJava team](https://github.com/Netflix/RxJava/issues/1000)) but only the Typesafe's implementation is mature enough to do some experimenting with it. This blog covers the 0.3 version of Akka Streams. You can find the 0.3 API [here](http://doc.akka.io/api/akka-stream-experimental/0.3/?_ga=1.31946302.1218088554.1402534155#package)

[RabbitMQ](http://www.rabbitmq.com/) is a messaging broker implementing AMQP 0-9-1 protocol. It's known for its reliability, speed and simplicity in everyday use.

These two technologies seem like a perfect fit, so in this post I'm going to explore some basic integration possibilities and example usage.

###Disclaimer
To try these examples you'll need to have access to RabbitMQ server. If you don't have it already you can following the instructions [here](http://www.rabbitmq.com/download.html).

I'm not going to show every detail of this solution in this post. I want to concentrate on transforming the RabbitMQ messages into the Reactive Stream. I encourage you to take a look at my Activator [template](https://github.com/jczuchnowski/rabbitmq-akka-stream) to see more details - like connecting to the RabbitMQ server.

##ActorProducer as RabbitMQ consumer

First thing we have to do (after connecting to the broker) is to get the messages from RabbitMQ somehow and then pass them into the Akka Stream Flow. Getting messages from the broker means that we're a consumer and passing them to the Flow means that at the same time we're a producer. Both of these activities will be performed by the same entity in our application - namely the ActorProducer.

As we are using the official RabbitMQ Java client, we have two ways to do this - pull or push.

###the pull

In this scenario we'll be calling the broker every time we want to get a message. That means that we should do this only if we're able to process the message. This is the way we'll propagate the backpressure from our Flow into the RabbitMQ.

ActorProducer is a trait that enables our Actor to communicate with the Flow. Every time the Flow is ready to take some new messages, it'll send a Request message indicating how many messages it is willing to process. On receiving the message, but before calling the `receive` function, ActorProducer will update its internal `totalDemand` parameter. At this point we're ready to service the message by calling RabbitMQ broker and passing the received messages using the ActorProducer's `onNext` method (that in turn will also update the `totalDemand` accordingly).

~~~ scala
  val autoAck = false
  val queueName = "queue.name"

  override def receive = {
    case Request(elements) => if (isActive) {
      (1 to elements) foreach { _ =>
        val response = channel.basicGet(binding.queue, autoAck)
        if (response != null) {
          val msg = new RabbitMessage(
            response.getEnvelope().getDeliveryTag(), 
            new String(response.getBody(), "UTF-8"), 
            channel)
          onNext(msg)
        }
      }
    }
  }
~~~

You'll probably notice that if RabbitMQ doesn't have any messages for us, it'll return null. That means that if we're consuming the messages faster than the producer is sending them to the broker, we will be generating much unnecessary web traffic and we'll be wasting resources.

That is why we will use the Push method next.

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

  def register(channel: Channel, queue: String, consumer: Consumer): Unit =  {
    val autoAck = false
    val queue = "queue.name"
    ch.basicConsume(queue, autoAck, consumer)
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

In Akka Streams 0.2, the only way of composing flow processing was through `Flow[In] => Flow[Out]` functions. That's ok, but we could do better. And of course Akka guys did do better. In Akka Streams 0.3 they've introduced `Duct`'s. Duct is a placeholder for your transformations that can be created independently from the Flow and later attached to it. The signature is `Duct[In, Out]`. As you might have guessed, `In` is a type that enters the Duct and `Out` is the type that leaves. Here's an example of a Duct:

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

We create the Duct by calling the companion object with an input type - `Duct[RabbitMessage]`. At this point our Duct has a type of `Duct[RabbitMessage, RabbitMessage]`. Later on we are doing some transformations using the `map` operator:

* acknowledge the message
* retrieve the message body
* log the message 

The transformed message that leaves our Duct is a String. So the final type is `Duct[RabbitMessage, String]`.

Ok, but still this doesn't do anything. So next...

##Putting it together

~~~ scala
  val materializer = FlowMaterializer(MaterializerSettings())

  flow append duct consume(materializer)
~~~