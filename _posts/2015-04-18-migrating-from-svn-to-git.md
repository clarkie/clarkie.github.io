---
layout: post
title: Migrating from SVN to git
categories: [dev]
tags: [git, svn]
fullview: true
---
When other attendees at conferences and meetups asked what version control we used I was always a little embarrased to say that we were still using SVN. Starting the CP4 project at [Concrete](www.concrete.cc) allowed us to start afresh and we had reasonably free reign to pick the technologies that we wanted to use. This meant that picking git for versioning was the obvious choice. 

This has worked great for the past nine or so months for our new repositories but it's now time to migrate our old SVN repositories over. The main CP3 application is in it's own repository and a team of engineers work on this daily so my view was that it wasn't going to be a simple switch over. This isn't the sort of thing you do every day so there's always going to be a bit of a risk and it couldn't impact on our feature delivery schedule or the monthly release cycle. 

## When to do the migration

The monthly release is pushed out into production on the last Wednesday of every month. The week preceeding this is considered as our UAT/QA week where we can run our regression tests and perform final demos to the clients and with our sprints running Monday to Friday we have a code freeze the friday before the UAT week. On that Friday we normally create a new branch for the next release and I felt that it would be at this point we would make the switch from SVN to Git

## Preparing to migrate

There are a number of resources that have helped with getting this working but I wanted to summarise our process as we have a few other repositories to migrate too:

### 1. Create the authors mapping text file
This command will output all the users that have ever commited to your svn repository. This is really useful when you have an old repository like ours as it means you can re-create the git commits using real git users later.
{% highlight bash %}
svn log --xml | grep author | sort -u | perl -pe 's/.*>(.*?)<.*/$1 = /'
{% endhighlight %}

We edited the author's file as our gitlab server is hooked up to active directory and our svn server isn't so every user's details were slightly different. 

### 2. Checkout/clone the svn repository as a git repo
Using the text file created in the previous step I created a local git version of our svn repository.
{% highlight bash %}
git svn clone --authors-file=../authors-transform.txt \
    -T trunk \
    -b branches/releases \
    http://[SVN_SERVER]/svn/concretePlatform
{% endhighlight %}
I had to use the `-b branches/releases` flag as we haven't follow the standard svn naming convention of having branches directly under the branches folder. 

### 3. Create a remote repository on gitlab
I created a new repository in the CP3 group called concrete-platform and then added it as a remote to the newly checked out repository
{% highlight bash %}
git remote add gitlab git@[GITLAB_SERVER]:cp3/concrete-platform.git
{% endhighlight %}
Notice that I've named the gitlab remote `gitlab`. This is because the SVN remote is by default named `origin`. By default gitlab, github and bitbucket all tell you to add a remote called `origin` but when I did this on the repository checked out from SVN I then couldn't fetch any new commits from the SVN remote. I'm sure there must be a way to correctly reference the SVN/origin remote vs the git/origin but I found the easiest and clearest way was to call the new git remote `gitlab`.

### 4. Checkout and push trunk/master branch to the gitlab remote
Navigate into the repository folder and checkout the master branch. Once this is complete it can be pushed to the newly created remote pointing to the gitlab server.
{% highlight bash %}
cd concretePlatform/
git checkout master
git push gitlab master
{% endhighlight %}

### 5. Checkout and push the 2015-apr release branch
We only have one other active branch at this time so I also checked this one out and pushed it up to gitlab. 
{% highlight bash %}
git checkout 2015-apr
git push gitlab 2015-apr
{% endhighlight %}

We now have a copy of our SVN repository on our gitlab server. During the time between the initial clone and code freeze a number of new commits have come in but it's just a matter of running the following to fetch the commits from SVN and push them up to gitlab:

{% highlight bash %}
git svn fetch
git checkout 2015-apr
git push gitlab 2015-apr
{% endhighlight %}

The final step is to duplicate our Jenkins jobs and updating the SCM details. We're going to run a number of dev site builds from git and then run the April release to production on 29th April.

Both the [git tutorial](http://git-scm.com/book/en/v2/Git-and-Other-Systems-Migrating-to-Git) and [Atlassian's tutorial](https://www.atlassian.com/git/tutorials/migrating-overview) were very useful with helping with this process.