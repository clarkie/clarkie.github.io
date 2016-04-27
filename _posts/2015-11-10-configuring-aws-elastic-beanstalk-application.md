---
layout: post
title: Configuring an AWS Elastic Beanstalk application
categories: [devops]
tags: [aws, elastic-beanstalk, nodejs, npm]
fullview: true
---

Having spent some more time recently deploying apps using Elastic Beanstalk for [Tido](http://www.tido-music.com) I have stumbled across some really nice features and some really annoying gotchas. I predominantely use the CLI but have been known to tamper in the AWS console UI too. That ended up being one of the gotchas as you'll find out.

##Application Setup
When building an app you might not initially think about how you want your servers configured but it is actually really important. Docker does address this in a similar way by bundling the container configuration alongside the application code. This is how you can do something similar with Elastic Beanstalk. 

Firstly create a `.elasticbeanstalk` directory in the root of your application. In here you'll create a `config.yml` file similar to this:

```
branch-defaults:
  master:
    environment: api-prod
global:
  application_name: store-api
  default_platform: Node.js
  default_region: eu-west-1
  profile: eb-cli
  sc: git
```

If you run the `eb init` command you will be taken through a wizard which will create this for you. 

The next command you'll need is `eb create`. This will again take you through another wizard but this time it will deploy your app to EC2.

If you run this command from a different branch it will link the new environment to the current branch. For example we're going to deploy our develop branch to a dev environment so we now have the following config:

```
branch-defaults:
  master:
    environment: api-prod
  develop:
    environment: api-dev
global:
  application_name: store-api
  default_platform: Node.js
  default_region: eu-west-1
  profile: eb-cli
  sc: git
```

##Application Configuration
From within your application root folder create a folder `.ebextensions`. In here you can put as many `.config` files as you wish. These will be run in alphabetical order so I prefix mine with a two digit number e.g. `00-enviornment-variables.config`.

For example the [aws-sdk](https://www.npmjs.com/package/aws-sdk) npm module recommends setting a couple of environment variables which we can do like so:

```
option_settings:
  - option_name: AWS_ACCESS_KEY_ID
    value: MY_AWS_ACCESS_KEY
  - option_name: AWS_SECRET_ACCESS_KEY
    value: SUPER_SECRET_AWS_SECRET
```

You can also configure all other aspects of how your Elastic Beanstalk app runs for example the healthcheck url, min and max cluster size:

```
option_settings:
  - namespace:  aws:elasticbeanstalk:application
    option_name:  Application Healthcheck URL
    value:  /ping
  - namespace: aws:autoscaling:asg
    option_name: MinSize
    value: 5
  - namespace: aws:autoscaling:asg
    option_name: MaxSize
    value: 40    
```

[This page](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/command-options-general.html) in the AWS docs has all the available options and details the defaults.

You can also configure the proxy settings. For example we wanted to accept longer urls than the default Nginx setting. 

```
files:
    "/etc/nginx/conf.d/proxy.conf" :
        mode: "000755"
        owner: root
        group: root
        content: |
           large_client_header_buffers 8 256k;
```

###Precedence Gotcha

The caught me out big time! I had previously gone into the admin UI and changed the minimum number of nodes for our cluster from the default 1 to 2.

This means that when I tried to configure the min and max settings from within my .ebextensions file it wasn't setting the values I wanted. Eventually I found a solution with the `eb config` command. Running this command from the linked branch will open up the default text editor with the current config file for that enviornment:

![Live editing config](/assets/img/2015-11-10/config-edit.png)

By deleting the two highlighted lines and saving the file, those settings reverted back to their defaults. By then running `eb deploy` once more my `.ebextensions` config files were run correctly.
