---
layout: post
title: The right way to get start with git flow
category: dev log
author: Chris Kong

excerpt: do not just google it... 

---


# The cheatsheet

You might start with git flow by checking the cheating sheet([http://danielkummer.github.io/git-flow-cheatsheet/index.html](http://danielkummer.github.io/git-flow-cheatsheet/index.html "git flow")) just like me.

The first thing you might thought after seeing this page is "Wow this page really save me a cup of coffee!"

Then you get started.

You inited.

You started a feature.

You finished a feature.

You published a feature.

You pulled a feature.

Then you felt something wrong.

First of all. You might notice that this proceed wouldn't work out if just follow it. Because you can just publish a feature after finishing it, because a finish will delete the local feature branch and merge it into local develop. And you might struggle for a long time about which one of these magic words created the remote develop branch, which none of them did.

So, it just a cheatsheet, meaning if you forget something **after** you have walked throught git flow, this page would bring some good memories back to you. 

One more thing that could be confusing even to those who have fooled around git for years, is the *git flow [whatever] pull* does not merge. And even more confusing, this command does not track.([http://stackoverflow.com/questions/18412750/why-doesnt-git-flow-feature-pull-track](http://stackoverflow.com/questions/18412750/why-doesnt-git-flow-feature-pull-track))

So, if you just did a pull, you need an extra step to track the origin by

    git branch -u origin/feature

# The Getting Started

Ok, we already know that this cheating sheet is not such a good idea. So where should we get started?

This site [http://yakiloo.com/getting-started-git-flow](http://yakiloo.com/getting-started-git-flow/) offers a really practical way to use git flow. If you already knew the basic use of git, you could just get started by searching **Git-Flow Commands** in the page. And then you could find a real scenario of using git flow.

To make a quick brief of how to use git flow to develop, this work flow below could do you some help.
(This scenario suggusts you are in charge of this project. It is helpful to have this POV since you could get the whole picture of how this works.)

	// for some reason, someone got a crazy idea
    git flow init 

	// everyone should do this step before doing anything else(stupid)
	// or they will wait to SUFFUR!!!!
	git pull origin develop

	// PM are just killing us
	git flow feature start IEVENTDONTKNOWWHATITIS
	
	/* a lot of 
	git add -A 
	git commit -m "I think this might be a fix"
	which I don't know why it could work
	*/
	
	// Some dude wishes this feature is finished
	git flow feature finish IEVENTDONTKNOWWHATITIS
	
	// which make him back to develop branch
	// And he is so luck to be the first one to push the code!
	// meaning he has to do an extra step
	// creating the remote develop branch
	git push origin develop

	// some other guy was just told to start a new feature
	// so...
	git pull origin develop

	git flow feature start AREALLYBIGONE

	/* a lot of 
	git add -A 
	git commit -m "I think this might be a fix"
	which I don't know why it could work
	*/ 
	
	// after a few commit, this guy realized 
	// I CAN'T DO IT ALONG!!!
	// he publish this feature, and begs his friends to help him
	git flow feature publish AREALLYBIGONE

	// his powerful friend is online
	git flow feature track AREALLYBIGONE

	/* a lot of 
	git add -A 
	git commit -m "Who the hell wrote this!?"
	*/
	
	git flow feature publish AREALLYBIGONE
	// after this, the guy really should buy his friend beer

	// So, this guy back to office
	git flow feature pull AREALLYBIGONE

	// He finds his friend is awesome
	// the code is good enough to submit
	git flow feature finish AREALLYBIGONE

	git push origin develop