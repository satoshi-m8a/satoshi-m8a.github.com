---
layout: post
title: "Play Framework 2.1のWebSocketで一対一通信"
date: 2013-05-18 12:50
comments: true
categories: 
---



Play2のサンプルコードの[websocket-chat](https://github.com/playframework/Play20/tree/master/samples/scala/websocket-chat,'websocket-chat')では`Concurrent.brodacast`を使って
```
val (chatEnumerator, chatChannel) = Concurrent.broadcast[JsValue]
```

この`chatChannel`に対して
```
chatChannel.push(msg)
```
とすることでchatRoomに接続している全てのクライアントに対してメッセージを送信しています。

全てのクライアントに同じメッセージを送る場合は`Concurrent.broadcast`で十分ですが、特定のクライアントに対してメッセージを送りたい場合はどうすればいいでしょうか。
とりあえず全員にメッセージを送って、クライアントに自分宛てに送られたメッセージかどうかを判断してもらうという解決方法もありますが、画像などを送る場合負荷が大きいですし、別のクライアントには見られたくないメッセージもあるでしょう。


この課題は`Concurrent.unicast`を使うことで解決できます。
`Concurrent.unicast`の使い方は以下のように
Concurrent.unicastメソッドに対して、接続開始、完了、エラー用の関数を引数にしてEnumeratorを生成します。
``` scala
 def onStart: Channel[JsValue] => Unit = {
    channel =>
      chatChannel = channel
      println("start")
      self ! NotifyJoin("you")
  }

  def onError: (String, Input[JsValue]) => Unit = {
    (message, input) =>
      println("onError " + message)
  }

  def onComplete = println("onComplete")

  val chatEnumerator = Concurrent.unicast[JsValue](onStart, onComplete, onError)
```
このとき、接続開始時に実行される関数（上の例ではonStart関数）の中で、channelを保持することで、以降はそのchannelに対して`channel.push(msg)`とすることで特定のクライアントに対してだけメッセージを送ることができます。




以下、websocket-chatをベースにしたサンプルコード

<!-- more -->

websocket-chatと主に異なる点はActorをクライアント毎に作っている点と`Concurrent.broadcast`の代わりに`Concurrent.unicast`を使っている点です。
このActorとusernameのマップをどこかに保持することで、サーバー側は好きなタイミングで特定のクライアントに対して個別にメッセージを送ることができるようになります。


参考:
<https://github.com/playframework/Play20/tree/master/samples/scala/websocket-chat>


``` scala
package models

import akka.actor._
import scala.concurrent.duration._

import play.api._
import play.api.libs.json._
import play.api.libs.iteratee._
import play.api.libs.concurrent._

import akka.util.Timeout
import akka.pattern.ask

import play.api.Play.current
import play.api.libs.concurrent.Execution.Implicits._
import play.api.libs.iteratee.Concurrent.Channel

object Robot {

  def apply(chatRoom: ActorRef) {

    // Create an Iteratee that logs all messages to the console.
    val loggerIteratee = Iteratee.foreach[JsValue](event => Logger("robot").info(event.toString))

    implicit val timeout = Timeout(1 second)
    // Make the robot join the room
    chatRoom ? (Join("Robot")) map {
      case Connected(robotChannel) =>
        // Apply this Enumerator on the logger.
        robotChannel |>> loggerIteratee
    }

    // Make the robot talk every 30 seconds
    Akka.system.scheduler.schedule(
      30 seconds,
      30 seconds,
      chatRoom,
      Talk("Robot", "I'm still alive")
    )
  }

}

object ChatRoom {

  implicit val timeout = Timeout(1 second)

  def join(username: String): scala.concurrent.Future[(Iteratee[JsValue, _], Enumerator[JsValue])] = {
    val roomActor = Akka.system.actorOf(Props[ChatRoom])

    Robot(roomActor)

    (roomActor ? Join(username)).map {

      case Connected(enumerator) =>

        // Create an Iteratee to consume the feed
        val iteratee = Iteratee.foreach[JsValue] {
          event =>
            roomActor ! Talk(username, (event \ "text").as[String])
        }.mapDone {
          _ =>
            roomActor ! Quit(username)
        }

        (iteratee, enumerator)

      case CannotConnect(error) =>

        // Connection error

        // A finished Iteratee sending EOF
        val iteratee = Done[JsValue, Unit]((), Input.EOF)

        // Send an error and close the socket
        val enumerator = Enumerator[JsValue](JsObject(Seq("error" -> JsString(error)))).andThen(Enumerator.enumInput(Input.EOF))

        (iteratee, enumerator)

    }
  }
}

class ChatRoom extends Actor {
  var chatChannel: Channel[JsValue] = null

  def onStart: Channel[JsValue] => Unit = {
    channel =>
      chatChannel = channel
      println("start")
      self ! NotifyJoin("you")
  }

  def onError: (String, Input[JsValue]) => Unit = {
    (message, input) =>
      println("onError " + message)
  }

  def onComplete = println("onComplete")

  val chatEnumerator = Concurrent.unicast[JsValue](onStart, onComplete, onError)

  def receive = {

    case Join(username) => {
      sender ! Connected(chatEnumerator)
      //self ! NotifyJoin(username)
    }

    case NotifyJoin(username) => {
      notifyAll("join", username, "has entered the room")
    }

    case Talk(username, text) => {
      notifyAll("talk", username, text)
    }

    case Quit(username) => {
      notifyAll("quit", username, "has left the room")
    }

  }

  def notifyAll(kind: String, user: String, text: String) {
    val msg = JsObject(
      Seq(
        "kind" -> JsString(kind),
        "user" -> JsString(user),
        "message" -> JsString(text)
      )
    )
    chatChannel.push(msg)
  }
}

case class Join(username: String)

case class Quit(username: String)

case class Talk(username: String, text: String)

case class NotifyJoin(username: String)

case class Connected(enumerator: Enumerator[JsValue])

case class CannotConnect(msg: String)

```

`Enumerator.imperative`を使った方法もありますが今はdeprecatedなので。  
参考：  
・<https://groups.google.com/forum/?fromgroups#!topic/play-framework/iKPFqusaosM>
・<http://qiita.com/items/e1a7461455be187d0565>  
やった。これで、ロボットと一対一でチャットできる。＼(^o^)／

