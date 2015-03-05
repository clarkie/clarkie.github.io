---
layout: post
title: Using AWS SQS with Node.js
categories: [aws]
tags: [sqs, aws, elastic-beanstalk]
fullview: true
---

At [Concrete](http://www.concrete.cc) we're looking at utilising some of AWS's services to speed up devleopment of our new platform. As part of this I've been investigating the use of their SQS service to help coordinate the delivery of thumbnails and assets globally. It's quite a simple queueing mechanism that office a guarantee of delivery "at least once". This isn't really that much of a problem for us as we'll generate keys in our apps and if we create the same thumbnail twice it's just a waste and nothing more.

Pumping messages into the queue was simple enough using the [aws-sdk](https://www.npmjs.com/package/aws-sdk) module:

{% highlight JavaScript %}
var AWS = require('aws-sdk');

AWS.config.update({accessKeyId: 'KEY', secretAccessKey: 'SECRET'});

var sqs = new AWS.SQS({region:'eu-west-1'}); 

var msg = { payload: 'a message' };

var sqsParams = {
    MessageBody: JSON.stringify(msg),
    QueueUrl: 'QUEUE_URL'
};

sqs.sendMessage(sqsParams, function(err, data) {
    if (err) {
        console.log('ERR', err);
    }

    console.log(data);
});
{% endhighlight %}

The response  will look something like this
{% highlight JavaScript %}
{ ResponseMetadata: { RequestId: '232c557d-b1ed-54a1-a88c-180f7aaf3eb3' },
  MD5OfMessageBody: '80cbb15af483887b15534f2ac3dfa46f',
  MessageId: '6cc50b09-17a8-4907-beeb-ed3a620b562f' }
{% endhighlight %}

On the other end of the queue you need to create a worker to do something with the message. The [sqs-consumer](https://www.npmjs.com/package/sqs-consumer) module by the BBC that handles polling the queue for you. Using this module my worker looked something like this:

{% highlight JavaScript %}
var Consumer = require('sqs-consumer');

var app = Consumer.create({
    queueUrl: 'QUEUE_URL',
    region: 'eu-west-1',
    batchSize: 10,
    handleMessage: function (message, done) {

        var msgBody = JSON.parse(message.Body);
        console.log(msgBody);

        return done();

    }
});

app.on('error', function (err) {
    console.log(err);
});

app.start();
{% endhighlight %}

Calling `done()` handles the removal of the message from the queue. Clearly, you will want to do a little bit more with your message but it gives an idea of what's going on. 

All this was very exciting but what I didn't realise was that when deploying apps using Elastic Beanstalk you don't need to worry about polling the queue yourself, all your app needs to do is expose a POST route that takes the message as the payload. I'm a big fan of the [Hapi](http://hapijs.com) framework so my worker ended up like this:

{% highlight JavaScript %}
var Hapi = require('hapi');

var server = new Hapi.Server();
server.connection({ port: process.env.PORT || 80 });

server.route({
    method: 'post',
    path: '/',
    handler: function (request, reply) {

        var msgBody = request.payload;
        console.log(msgBody);

    }
});

server.start(function() {
    console.log('server started');
});
{% endhighlight %} 

