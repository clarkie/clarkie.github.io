---
layout: post
title: Predictable reads with dynamodb
categories: [devops]
tags: [aws, dynamodb, nodejs]
fullview: true
---

## The scenario

Three weeks ago I decided to replace a MySQL database with a couple of DynamoDB tables. The main reason for this was that [Tido](http://www.tido-music.com) are still very much in the 'discovery phase' of startup life and so requirements are changing thick and fast. MongoDB might have been a more obvious choice but that opens up lots of configuration options. I wanted something much simpler. 

At Tido I'm the backend server guy, frontend developer and ops guy all in one; true 'full stack'. DynamoDB seemed like an ideal solution: zero setup overhead and very limited configuration choices. All you get are two knobs to twiddle with that make things go faster, read and write capacity. 

I still felt a little uncomfortable having never used DynamoDB before and it felt like we were stepping into the unknown. First off, how do we interact with it? We could use the [aws-sdk](https://www.npmjs.com/package/aws-sdk) but this seemed very verbose and clunky since you have to specify the type of property you want to set on every object. I looked at [dynamoose](https://www.npmjs.com/package/dynamoose) but this didn't seem to be being maintained (whatever that means these days). Then at a [SushiJS](https://ti.to/sushijs-ldn/) event a [friend](https://twitter.com/export_mike) told me about [vogels](https://www.npmjs.com/package/vogels). This was exactly what I had been looking for: a sequelize/mongoose style abstraction over the aws-sdk.

I won't bore you with the details of our setup but we went from 11 MySQL tables to two DynamoDB tables. 

The reduction in the amount of code needed was nice and it 'felt' fast; not only to read and write but to work with too. We were able to completely swap out the database in a couple of weeks. 

This feeling that we'd made the right decision was short lived. Our IOS team had started implementing against the api and were reporting that it wasn't responding. And sure enough, a quick look in the logs showed response times well above 30s. This is way too high for an api that is just returning around 100 products. Through some basic browser refreshes I struggled to replicate the issue. Welcome ~~minigun~~ [Artillery](https://artillery.io/).

## Testing

This isn't the best use of Artillery as we didn't need to walk through a user journey as such but instead we just had to replicate a couple of ios developers reloading apps and browsers simultaneously. From what I'd read about DynamoDB the read capacity units (RCU) that you can set on your tables is the maximum requests per second (up to 1000kb). If you are happy with 'dirty reads' (eventually consistent) then you get twice the throughput. Based on this I estimated that for our product catalogue 100 RCU would be plenty. So I tested it by running the following commands:

```bash
artillery quick -d 60 -r 1 "https://dynamo.eu-west-1.elasticbeanstalk.com/volumes"
artillery quick -d 60 -r 2 "https://dynamo.eu-west-1.elasticbeanstalk.com/volumes"
artillery quick -d 60 -r 3 "https://dynamo.eu-west-1.elasticbeanstalk.com/volumes"
```

At this point I started to see some throttled requests but I wasn't seeing enough of a trend to be able to predict what would be a correct value for the RCU. Running the tests for longer would probably make sense so I started running them for 10 minutes. I also increased the number of requests per second to 5 and the RCU to 200:

```bash
artillery quick -d 600 -r 5 "https://dynamo.eu-west-1.elasticbeanstalk.com/volumes"
```

This is the resulting cloudwatch monitoring output for those first two sets of tests. The spike on the far left being the first set of three tests (1-3 req/s) and the persistently high blue line on the right being the 10 minute test:

![First test runs](/assets/img/2016-04-24/test1.png)

That last test was interesting because a consistent behaviour was starting to appear. For a catalogue of 100 items at 5 'dirty' requests per second, the metrics were showing 250 consumed RCU (100x5/2). You can see from the artillery output that I was not seeing the best performance but that was down to the throlled requests. If your consumed RCU goes above your provisioned RCU then the extra requests will be throttled or rejected. AWS do give you a little leeway but not when you're over for a prolonged period of time. 

![First test results](/assets/img/2016-04-24/artillery1.png)

I now wanted to see how predictable this really could be. If I doubled the requests per second, could I guess the correct RCU? The results:

![Second test run](/assets/img/2016-04-24/test2.png)

And from the artillery metrics I was pleased to see better performance than with the previous test as very few of the requests were being throttled. 

![Second test results](/assets/img/2016-04-24/artillery2.png)
