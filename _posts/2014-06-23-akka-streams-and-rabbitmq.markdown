---
comments: true
date: 2014-06-23 00:11:55
layout: post
slug: akka-streams-and-rabbitmq
title: Akka Streams and RabbitMQ
summary: Akka Streams is an exciting new technology from Typesafe that is an implementation of the Reactive Streams specification. RabbitMQ is a messaging broker implementing AMQP 0-9-1 protocol. It's known for its reliability, speed and simplicity in everyday use. These two technologies seem like a perfect fit, so in this post I'm going to explore some basic integration possibilities and example usage.
author: Jakub Czuchnowski
tags:
- Scala
- Akka
- Akka Streams
- Reactive Streams
- RabbitMQ
---

Akka Streams is an exciting new technology from [Typesafe](http://www.typesafe.com/) that is an implementation of the [Reactive Streams](http://www.reactive-streams.org/) specification. To this point two other implementations have been declared (from the [Reactor team](https://github.com/reactor/reactor) and Netflix's [RxJava team](https://github.com/Netflix/RxJava/issues/1000)) but only the Typesafe's implementation is mature enough to do some experimenting with it. This blog covers the 0.3 version of Akka Streams. You can find the API [here](http://doc.akka.io/api/akka-stream-experimental/0.3)

[RabbitMQ](http://www.rabbitmq.com/) is a messaging broker implementing AMQP 0-9-1 protocol. It's known for its reliability, speed and simplicity in everyday use.

These two technologies seem like a perfect fit, so in this post I'm going to explore some basic integration possibilities and example usage.

###Disclaimer
To try these examples you'll need to have access to RabbitMQ server. If you don't have it already you can follow the instructions [here](http://www.rabbitmq.com/download.html).

I'm not going to show every detail of this solution in this post. I want to concentrate on transforming the RabbitMQ messages into the Reactive Stream. I encourage you to take a look at my Activator [template](https://github.com/jczuchnowski/rabbitmq-akka-stream) to see more details - like connecting to RabbitMQ server, initiating channels etc.

##Simple model

We're going to use a simple representation of RabbitMQ message. It's not in any way a proper representation that would get you all the way in your RabbitMQ usage, but it is enough to show some basic stuff in this blog post.

~~~ scala
class RabbitMessage(val deliveryTag: Long, val body: ByteString, channel: Channel) {

  def ack(): Unit = channel.basicAck(deliveryTag, false)
  
  def nack(): Unit = channel.basicNack(deliveryTag, false, true)
}
~~~

The key takeaway here is that we are remembering the channel and the delivery tag to be able to acknowledge or reject this message later during the processing. The body of course is the message itself.

##ActorProducer as RabbitMQ consumer

First thing we have to do (after connecting to the broker) is to get messages from RabbitMQ somehow and then pass them into the Akka Stream Flow. Getting messages from the broker means that from the RabbitMQ point of view we're a consumer. On the other hand, passing them to the Flow means that at the same time we're a producer from Reactive Streams perspective. Both of these activities will be performed by the same entity in our application - namely the ActorProducer.

As we are using the official RabbitMQ Java client, there are two ways to do this - pull or push.

###the pull

In this scenario we'll be calling the broker every time we want to get a message. That means that we should do this only if we're able to process the message. This is the way we'll propagate the backpressure from our Flow into RabbitMQ.

ActorProducer is a trait that enables our Actor to communicate with the Flow. Every time the Flow is ready to take some new messages, it'll send a Request message indicating how many messages it is willing to process. On receiving the message, but before calling the `receive` function, ActorProducer will update its internal `totalDemand` parameter. At this point we're ready to service the message by calling RabbitMQ broker and passing the received messages using the ActorProducer's `onNext` method (that in turn will also update the `totalDemand` accordingly).

~~~ scala
class RabbitConsumerActor extends ActorProducer[RabbitMessage] {

  val autoAck = false
  val queueName = "queue.name"

  ...

  override def receive = {
    case Request(elements) if isActive =>
      (1 to elements) foreach { _ =>
        val response = channel.basicGet(binding.queue, autoAck)
        
        if (response != null) {
          val msg = new RabbitMessage(
            response.getEnvelope().getDeliveryTag(), 
            ByteString(response.getBody()), 
            channel)
          
          onNext(msg)
        }
      }
  }
}
~~~

You'll probably notice that if RabbitMQ doesn't have any messages for us, it'll return null and we will still be able to process more messages. It means that if we're consuming the messages faster than the (imaginary for now) producer is sending them to the broker, we will be generating a lot of unnecessary web traffic and we'll be wasting resources. Imagine annoying child constantly asking "are we there yet? are we there yet? and now? ..." - this is exactly what our RabbitConsumerActor is doing right now while RabbitMQ broker has nothing new to say.

That is why we will use the Push method next.

###the push

For this scenario we will need to create an instance of `com.rabbitmq.client.Consumer` and register it to listen to the RabbitMQ broker. This way we will get notified whenever there's a new message. Our Consumer will do one thing only - it'll send a wrapped message to be serviced in the `receive` function.

~~~ scala
class RabbitConsumerActor extends ActorProducer[RabbitMessage] {

  ...

  val consumer = new DefaultConsumer(channel) {
    override def handleDelivery(
        consumerTag: String, 
        envelope: Envelope, 
        properties: AMQP.BasicProperties, 
        body: Array[Byte]) = {
      self ! new RabbitMessage(envelope.getDeliveryTag(), ByteString(body), channel)
    }
  }

  def register(channel: Channel, queue: String, consumer: Consumer): Unit =  {
    val autoAck = false
    val queue = "queue.name"
    ch.basicConsume(queue, autoAck, consumer)
  }

  register(channel, queue, consumer)
}
~~~

Now that we actually `receive` our message we have to check if there's a demand for it. If there is, we use the `onNext` method to pass it to the Flow. If not, we reject it by calling `nack()`. The message will be later resent by the broker and hopefully processed when free resources are available.

~~~ scala
class RabbitConsumerActor extends ActorProducer[RabbitMessage] {

  ...

  override def receive = {
    case msg: RabbitMessage => 
      if (isActive && totalDemand > 0) {
        onNext(msg)
      } else {
        msg.nack()
      }
  }
}
~~~

##Creating the Flow

It's time for the Flow. Akka Streams is obviously based on Akka Actors, so the first thing we have to do is to create an ActorSystem. Our RabbitConsumerActor is then created by calling the `ActorProducer.apply()` and is later passed as a producer to the Flow.

~~~ scala
object RabbitApp extends App {
  
  implicit val actorSystem = ActorSystem("rabbit-akka-stream")

  val rabbitConsumer = ActorProducer(actorSystem.actorOf(new RabbitConsumerActor))

  val flow = Flow(rabbitConsumer)

  ...
}
~~~

Nothing really interesting here. Now we have a flow, but it doesn't do anything. So...

##Ducting the Flow

We can now call some Flow methods to process our message. Akka Streams allow us to do a lot of things here - `map`, `foreach`, `group`, `zip` and more. I'm not going to go through all of these as there are already some good descriptions out there - like in Frank Sauer's [CEP using Akka Streams](http://www.franklysauer.com/2014/05/cep-using-akka-streams/). What I'm interested in here is what to do when you want to define your stream manipulation separately from the Flow declaration itself. 

In Akka Streams 0.2, the only way of composing flow processing was through `Flow[In] => Flow[Out]` functions. That's ok, but we could do better. And of course Akka guys did do better. In Akka Streams 0.3 they've introduced `Duct`'s. Duct is a placeholder for your transformations that can be created independently from the Flow and later appended to it. The signature is `Duct[In, Out]`. As you might have guessed, `In` is a type that enters the Duct and `Out` is the type that leaves. Here's an example of a Duct:

~~~ scala
object RabbitApp extends App {

  ...

  val duct: Duct[RabbitMessage, String] = Duct[RabbitMessage].
    map { msg =>
      msg.ack()
      msg
    }.
    map { _.body.utf8String }.
    map { msg => 
      logger.info(msg)
      msg 
    }
}
~~~

We create the Duct by calling the companion object with an input type - `Duct[RabbitMessage]`. At this point our Duct has a type of `Duct[RabbitMessage, RabbitMessage]`. Later on we are doing some processing using the `map` operator:

* acknowledge the message
* retrieve the message body as String
* log the message 

The transformed message that leaves our Duct is a String. So the final type is `Duct[RabbitMessage, String]`.

Ok, but still ... this doesn't do anything.

##Putting it all together

We've created our ActorProducer, connected it to the Flow and defined some processing in a Duct. Time to connect it together and consume the stream of messages. Here it is:

~~~ scala
object RabbitApp extends App {

  ...

  val materializer = FlowMaterializer(MaterializerSettings())

  flow append duct consume(materializer)
}
~~~

And it's done. We're consuming messages from RabbitMQ. You can go to the management console now and send some messages to the queue your RabbitConsumerActor is listening to. What you should see at the very least is messages being logged to the console. You can play with the `Duct` definition to do whatever else you want.

It's not over. Akka Streams is 0.3 right now. It'll be exciting to see how it evolves and I'll be there to watch and write about it. Particularly from RabbitMQ's point of view. Leave a comment here, drop me an email at jakub@scalac.io or tweet @jczuchnowski if you want to talk about this some more.

