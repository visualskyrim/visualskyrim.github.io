---
layout: post
title: "Make sync call to web socket server from Play Framework"
description: ""
modified: 2016-01-14
tags: WebSocket ServerDeveloping
category: ServerDeveloping
image:
  feature: abstract-1.jpg
  credit: Chris Kong
  creditlink: http://visualskyrim.github.io/
comments: true
share: true
---

# My case (just ignore)

Recently, I was trying some complex distributed pattern in server development.
In one scenario, client calls the web service, and web service parse request then pass it to a background worker via DynamoDB or message queue. The worker will send result to the message bus, and the web socket server fetch this result to notify the browser.

However, some how this async design must have a sync implementation. Namely, instead of return empty response to the client then let client wait for the notification, the web service must return the result itself, which means the web service should play roles as a web socket client to fetch the result from web socket server.


# Use of socket.io-java-client

To integrate with my Play! 2 (Java), I choose [nkzawa/socket.io-client.java](https://github.com/nkzawa/socket.io-client.java) as socket client library, because its dependency can easily be managed in sbt, like:

{% highlight scala %}

libraryDependencies ++= Seq(
  // your dependences
  "com.github.nkzawa" % "socket.io-client" % "0.1.2"
)
{% endhighlight %}


After importing this library, you could create socket within your API. In my case, I have to create socket for each request due to web socket server implementation:

{% highlight java %}
public class WebSocketHelper {
    private static String _socketServer;

    public static Socket GetParamSocket(String paramKey, String paramValue, String requestId){
        String url = String.format("%s?%s=%s&request=%s", _socketServer, paramKey, paramValue, requestId);

        IO.Options options = new IO.Options();
        // if you want to create multiple socket like me, use forceNew
        options.forceNew = true;
        return IO.socket(url, options);
    }
}
{% endhighlight %}


Note that in this library, Socket internally manages a cache to store established socket ***using a hash map with host of the url as its key***.

Since I want to create socket for each request and every socket I will create definitely has the same host, so I will have to use **forceNew** option to make `IO.socket` return me a new socket every time.

# Use socket to implement sync call

To make API wait for the result sent by web socket server, I must hold this socket until socket server send things back after connecting. So, let us make a new class for that:

{% highlight java %}
public class Feedback  {

    public static final String responseString = "message";

    private Socket socket;
    private boolean isFinished;
    private String message;
    private int retryTime;

    public Feedback(Socket socket) {
        this.isFinished = false;
        this.message = "";
        this.retryTime = 0;
        this.socket = socket;
    }

    public static Feedback GetFeedback(String serviceId, String playerId, String stimulusId) throws URISyntaxException {

        Socket socket = WebSocketHelper.GetStimulatedSocket(serviceId, playerId, stimulusId);
        Feedback feedback = new Feedback(socket);
        return feedback;
    }

    public void Start() throws URISyntaxException {

        socket.on("message", new Emitter.Listener() {
            @Override
            public void call(Object... args) {
                message = args[0].toString();
                isFinished = true;
            }
        });

        socket.open();
    }

    public String GetMessage() throws InterruptedException {
        while (!isFinished && this.retryTime < 21) {
            Thread.sleep(500);
            this.retryTime ++;
        }

        socket.disconnect();
        socket.close();

        return this.message;
    }
}
{% endhighlight %}


# Integrate with Play!

Play! framework has very good sense to deal with thread holding situation.

To avoid blocking the web service, we are going to use `F.Promise<Result>`, despite that every action in play is actually returning `F.Promise<Result>`.

{% highlight java %}
public class SyncController extends Controller{

    public static F.Promise<Result> SyncAction(String paramValue) {
        // a lot of validations
        // ...

        Feedback feedback = null;
        String requestId = UUID.randomUUID().toString().replace("-", "");
        try {
            feedback = Feedback.GetFeedback(requestId);
            feedback.Start();
        } catch (URISyntaxException e) {
            e.printStackTrace();
            return F.Promise.promise(new F.Function0<Result>() {
                @Override
                public Result apply() throws Throwable {
                    return internalServerError();
                    // return proper information
                }
            });
        }

        Worker.sendTask(paramValue, requestId);

        final Feedback finalFeedback = feedback;
        F.Promise<String> promiseMessage = F.Promise.promise(
            // note it is Function0 here
            new F.Function0<String>() {
                @Override
                public String apply() throws Throwable {
                    return finalFeedback.GetMessage();
                }
            }
        );

        return promiseMessage.map(
            // and here is Function
            new F.Function<String, Result>() {
                @Override
                public Result apply(String message) throws Throwable {
                    // maybe some wrapping here
                    return ok(message);
                }
            }
        );
    }
}
{% endhighlight %}


# Simple test web socket code

You can also build a simple socket server to check your code(node.js with socket.io):

{% highlight javascript %}
var app = require('http').createServer(handler)
var io = require('socket.io')(app);
var fs = require('fs');
var AWS = require('aws-sdk');
AWS.config.loadFromPath('./aws.json');
var fs = require('fs');

var config = JSON.parse(
	fs.readFileSync('config.json'));

app.listen(3000);

function handler (req, res) {
  fs.readFile(__dirname + '/index.html',
  function (err, data) {
    if (err) {
      res.writeHead(500);
      return res.end('Error loading index.html');
    }

    res.writeHead(200);
    res.end(data);
  });
}


var requestMap = {};

io.on('connection', function(socket){
  console.log('socket connected!');
	var stimulusId = socket.handshake.query.requestId;
  console.log("connected " + requestId);
	requestMap[requestId] = socket;
});


setInterval(function () {
  // get result from worker
  var results = checkResults();
  for (var i = results.length - 1; i >= 0; i--) {
    var r = results[i];
    var incomingId = r.requestId;
    if (incomingId in requestMap) {
      // get origin socket
      var origin = requestMap[incomingId];
      // send back result
      origin.emit('message', record);
      delete requestMap.stimulus_id;
    }
  };
  });
}, 2000);

{% endhighlight %}



Maybe you just need one or two parts above to solve your own problem, like use of socket client library or simple use of Promise in Play!. But it is my first post in Qiita, and I am really fond of some opinions and critics. For example, I don't think create socket every time is very efficient, maybe I could build a helper to take charge of that.

Thanks for advance for you to put anything here!
