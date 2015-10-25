---
layout: post
title:  "Binding an Akka HTTP route to a stream"
date:   2015-10-25 15:00:00
image:  /assets/article_images/DSCF0808.JPG
---

As I mentioned [earlier]({% post_url 2015-10-21-akka-stream-http-sandbox-overview %}) I started working on an imaginary problem, which involves receiving messages from an HTTP endpoint and processing them in an Akka Stream. Note that the client then forgets about that message, its only concern is that it should be put into the processing queue, so the server will respond with "OK" without actually finishing the processing.

Akka HTTP gives us a nice [stream interface](http://doc.akka.io/docs/akka-stream-and-http-experimental/1.0/scala/http/low-level-server-side-api.html#Streams_and_HTTP), basically we get a source of requests and a sink of responses for each connection. (As well as a byte source for each message.) However this doesn't really fit our needs:

* Firstly, we get a separate source/sink pair for each connection, while we want only one processing stream. Unfortunately we cannot bind a new source to an existing stream.

* Secondly, the messages should come in on only one endpoint, e.g., POST /api/v1/message, so we can have other useful things in the same app. (E.g., health check, ping, version info, documentation) Akka gives the request source per connection, not per endpoint, so either we implement our routing logic in the stream or... or we use Akka HTTP routing. The problem with that is it does not have a stream interface, i.e. you cannot make a `Source` out of a route.

We need to build a [Source](http://doc.akka.io/api/akka-stream-and-http-experimental/1.0/#akka.stream.scaladsl.Source$) which we can put messages into later, after it is created. Fortunately an `Actor` is just good for such thing; with basically copying the [example](http://doc.akka.io/docs/akka-stream-and-http-experimental/1.0/scala/stream-integrations.html#ActorPublisher) and adding some boilerplate code we get what we want.

(The following file is on GitHub, [here](https://github.com/szmg/akka-stream-http-sandbox/blob/master/akkatest-main/src/main/scala/com/mate/akkatest/main/web/ToStreamHandler.scala).)

{% highlight scala linenos %}
package com.mate.akkatest.main.web

import akka.actor.{ActorRef, Props}
import akka.http.scaladsl.model.StatusCodes
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.server.Route
import akka.stream.actor.ActorPublisher
import akka.stream.actor.ActorPublisherMessage.{Cancel, Request}
import akka.stream.scaladsl.Source
import akka.util.Timeout

import scala.annotation.tailrec
import scala.util.{Failure, Success}

object ToStreamHandler {
  import akka.pattern.ask

  def props[T](bufferSize: Int): Props = Props[ToStreamHandler[T]](new ToStreamHandler[T](bufferSize))
  def source[T](bufferSize: Int): Source[T, ActorRef] = Source.actorPublisher[T](ToStreamHandler.props(bufferSize))

  sealed trait StreamResponse
  object Accepted extends StreamResponse
  object Rejected extends StreamResponse
  case class Send[T](value: T)

  def sendTo[T](toStreamHandler: ActorRef, what: T)(implicit timeout: Timeout): Route =
    onComplete(toStreamHandler ? Send(what)) {
      case Success(Accepted) =>
        complete("OK")
      case Success(Rejected) =>
        complete(StatusCodes.TooManyRequests)
      case Failure(e) =>
        failWith(new IllegalStateException("Internal server error", e))
      case somethingElse =>
        // TODO log
        failWith(new IllegalStateException("Internal server error - " + somethingElse))
    }
}


class ToStreamHandler[T](val bufferSize: Int) extends ActorPublisher[T] {
  import ToStreamHandler._

  var buffer = Vector.empty[T]

  override def receive: Receive = {
    case Send(_) if buffer.size >= bufferSize =>
      sender ! Rejected
    case Send(elem: T @unchecked) =>
      sender ! Accepted
      buffer :+= elem
      deliverBuf
    case Request(_) =>
      deliverBuf
    case Cancel =>
      context.stop(self)
  }

  @tailrec final def deliverBuf: Unit =
    if (totalDemand > 0) {
      val tooBigForOneBatch = totalDemand > Int.MaxValue
      val limit = Math.min(Int.MaxValue, totalDemand)
      val (use, rest) = buffer.splitAt(limit.toInt)
      buffer = rest
      use.foreach(onNext)
      if (tooBigForOneBatch) {
        deliverBuf
      }
    }
}
{% endhighlight %}

The actual logic starts on line 46: we send `Send(payload)` messages to this actor and it will put the payload into its internal buffer and respond with `Accepted` unless the buffer is full in which case the response will be `Rejected`. Later the internal buffer will function as an Akka Stream source.

Then there is the method `sendTo` on line 26: that is the glue between Akka Routing and the source actor. It returns a `Route` which responds with "200 OK" if the message was successfully buffered, "429 Too many requests" otherwise.

All we need now is to put this into use:

(Note that this code was just written for the sake of this post... Obviously it misses some imports and will not compile, but it'll give you an idea about the usage. For a working example, go [here](https://github.com/szmg/akka-stream-http-sandbox/blob/master/akkatest-main/src/main/scala/com/mate/akkatest/main/Main.scala).)

{% highlight scala %}
object Main extends App {
  implicit val system = ActorSystem("akkatest")
  implicit val materializer = ActorMaterializer()

  /** source of entities, its materialization will be the ActorRef to send the messages to */
  private val entitiesSource: Source[SomeClass, ActorRef] = ToStreamHandler.source[SomeClass](bufferSize = 10)

  /** processing flow; in this case it'll only print the input */
  private val wholeFlow = entitiesSource.to(Sink.foreach(println))

  /** materializing the flow, and saving the result as we have to send messages there */
  private val entities: ActorRef = wholeFlow.run()
  
  /** Akka Http route with a simple ping and an endpoint linked to the flow above (wholeFlow) */
  private val route =
    (path("admin" / "ping") & get & complete("pong")) ~
    (path("api" / "v1" / "something") & post & entity(as[SomeClass])) { entity => sendTo(entities, entity) }
  
  // Start server
  Http().bindAndHandle(route, "localhost", 8080)
  println("Server started on localhost:8080. Ctrl-C to exit...")
}
{% endhighlight %}

So we can have multiple services in one app (like ping and _/api/v1/something_ here) and we have only one instance of the processing queue, in which the messages from all the clients will go.

* * *

Note that this was written with Akka Stream experimental 1.0 and Akka HTTP experimental 1.0. Later versions may contain a built-in solution for this scenario.
