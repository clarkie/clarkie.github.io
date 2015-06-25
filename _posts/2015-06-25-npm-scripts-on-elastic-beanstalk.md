---
layout: post
title: npm scripts on AWS Elastic Beanstalk
categories: [nodejs]
tags: [aws, elastic-beanstalk, nodejs, npm]
fullview: true
---

I've been building a little app running a Hapi server and decided I wanted to plug a web client in front of it. I decided to try webpack as I'd heard people raving about it. The front-end tech is irrelevant for this post as what I've learnt applies to any pre deployment processes; running grunt tasks, less compilation, etc. The basic problem I'm trying to solve is that I want to run some front end compilation before starting my Hapi server. In my case this will create a bundle.js file containing the webpacked js code.

As a quick hack I knew I could just include the bundle.js file in my git repo. This obviously works but it's not very DRY; all the code is in the repository twice. This is clearly not the right way to do it but it works. Another option I looked at was to use the npm postinstall script. This worked perfectly on my local environment but when I pushed to Elastic Beanstalk a number of things didn't work quite how I expected. 

1. When Elastic Beanstalk runs npm install it includes the --production flag.
2. The postinstall script doesn't run

For #1 this is kind of obvious but I'd considered my front end libraries as pre-production. I was only going to ship the bundled code so why would I need to include the likes of Backbone, Marionette, etc in my dependencies. The fix for this was to move them from devDependencies to dependencies.

For #2 I decided to run an experiment. I created a pacage.json file with all the possible scripts with just a console.log in each one:

{% highlight javascript %}
{
	"name": "npm-script-test",
	"version": "1.0.0",
	"description": "",
	"scripts": {
		"prepublish": "node -e \"console.log('prepublish');\"",
		"publish": "node -e \"console.log('publish');\"",
		"postpublish": "node -e \"console.log('postpublish');\"",
		"preinstall": "node -e \"console.log('preinstall');\"",
		"install": "node -e \"console.log('install');\"",
		"postinstall": "node -e \"console.log('postinstall');\"",
		"preuninstall": "node -e \"console.log('preuninstall');\"",
		"uninstall": "node -e \"console.log('uninstall');\"",
		"postuninstall": "node -e \"console.log('postuninstall');\"",
		"preversion": "node -e \"console.log('preversion');\"",
		"version": "node -e \"console.log('version');\"",
		"postversion": "node -e \"console.log('postversion');\"",
		"pretest": "node -e \"console.log('pretest');\"",
		"test": "node -e \"console.log('test');\"",
		"posttest": "node -e \"console.log('posttest');\"",
		"prestop": "node -e \"console.log('prestop');\"",
		"stop": "node -e \"console.log('stop');\"",
		"poststop": "node -e \"console.log('poststop');\"",
		"prestart": "node -e \"console.log('prestart');\"",
		"start": "node -e \"console.log('start');\"",
		"poststart": "node -e \"console.log('poststart');\"",
		"prerestart": "node -e \"console.log('prerestart');\"",
		"restart": "node -e \"console.log('restart');\"",
		"postrestart": "node -e \"console.log('postrestart');\""
	}
}
{% endhighlight %}

The resulting log on Elastic Beanstalk in `/var/log/nodejs/nodejs.log`:

{% highlight bash %}
> worth-sharing@1.0.0 prestart /var/app/current
> node -e "console.log('prestart');"

prestart

> worth-sharing@1.0.0 start /var/app/current
> node -e "console.log('start');"

start

> worth-sharing@1.0.0 poststart /var/app/current
> node -e "console.log('poststart');"

poststart
{% endhighlight %}

It appears that these are the only npm scripts run by Elastic Beanstalk.

My solution was to move my webpack compilation from the `postinstall` to `prestart` and everything started working!
