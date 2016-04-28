---
layout: post
title: Predictable reads with dynamodb
categories: [devops]
tags: [aws, dynamodb, nodejs]
fullview: true
---

## The scenario

Three weeks ago I decided to replace a MySQL database with a couple of DynamoDB tables. The main reason for this was that Tido is still very much in the 'discovery phase' of startup life and so it is very difficult to pin down what the business will look like next month, let alone next year. After much deliberation, a document database seemed to be the right choice but that just opens up more options. MongoDB would have been the natural choice but again that opens up even more options in terms of configuration. I wanted something much simpler. At Tido I'm the backend server guy, frontend developer and ops guy all in one; true 'full stack'. DynamoDB seemed like an ideal solution; zero setup overhead and very limited configuration choices; you get two knobs to twiddle with make things go faster, read and write capacity. 

We went for it although I still felt a little uncomfortable going into the unknown. First off, how do I interact with it? I could use the [`aws-sdk`](https://www.npmjs.com/package/aws-sdk) but this seemed very verbose and clunky - you have to specify the type of property you want to set on every object. I looked at [dynamoose](https://www.npmjs.com/package/dynamoose) but this didn't seem to be being maintained (whatever that means these days). Then at a SushiJS event a friend[Mike, LINK] told me about [vogels](https://www.npmjs.com/package/vogels). This was exactly what I had been looking for, a sequelize/mongoose abstraction over the aws-sdk.

I won't bore you with the details of our setup but we went from 11 MySQL tables to two DynamoDB tables. The reduction in the amount of code needed was nice and it 'felt' fast, not only to read and write but to work with too. Even though vogels is callback based we were able to completely swap out the databse in a couple of weeks. This feeling that we'd made the right decision was shortlived. Our IOS team had started implementing against the api and were reporting that it wasn't responding. A quick look in the logs and sure enough we were seeing response times well above 30s. This is way to high for an api that is just returning about 100 products. Through some basic browser refreshes I struggled to replicate the issue. Welcome --minigun-- [Artillery](https://artillery.io/).

## Load testing

This isn't the best use of Artillery I've seen as we didn't need to walk through a user journey as such but instead we just had to replicate a couple of ios developers reloading apps and browsers simultaneously. From what I'd read about DynamoDB the read capacity units (RCU) that you can set on your tables is the maximum requests per second (up to 1000kb). If you are happy with 'dirty reads' then you get twice the throughput. Based on this I estimated that for our product catalogue and estimated inital load 100 RCU would be plenty. So I tested it. 

![First test run](/assets/img/2016-04-24/test1.png)

## So what was happening?

![Second test run](/assets/img/2016-04-24/test2.png)