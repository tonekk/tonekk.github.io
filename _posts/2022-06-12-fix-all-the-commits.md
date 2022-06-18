---
layout: post
title:  "Fix all the commits!"
date:   2022-06-12 10:48:51 +0200
categories: coding
---

Before we start with today's blog post, I would like to take you on a little trip to 2012. A time when the *mobile-first*-transition was still in progress, Dubstep was still a thing and all memes were made using the same dozen templates and, of course, the infamous [Impact font](https://en.wikipedia.org/wiki/Impact_(typeface)).

![Fix all the commits!](/img/fix-all-the-commits.jpg "Fix all the commits!")

Feeling melancholic right now? Good. Just the right mood to fix some of your git workflows.


## Faulty patterns

I'm pretty sure we have all seen or have even been responsible for commit histories like this one:

```
commit 25710bfc981a300e945126555d6b0e5ebbff2929 (HEAD -> main)
Author: Finn Heemeyer <finn@tonekk.de>
Date:   Sun Jun 12 14:36:23 2022 +0200

    Fix typo in User model

commit 27b02255b1eec61209c39f820263ab35e2c1eb0c
Author: Finn Heemeyer <finn@tonekk.de>
Date:   Sun Jun 12 14:35:38 2022 +0200

    Fix User spec
 
commit b57080fb1fd88e47c1798d7fa32219f222488aaf
Author: Finn Heemeyer <finn@tonekk.de>
Date:   Sun Jun 12 14:34:20 2022 +0200
 
    Add User spec
 
commit 997a13f6a8cbe7a7cef5cb39bc42c097d5acca63
Author: Finn Heemeyer <finn@tonekk.de>
Date:   Sun Jun 12 14:31:53 2022 +0200
 
    Fix User model
 
commit 9b1d56785c41b34fd0b998a44b4565970c2e7568
Author: Finn Heemeyer <finn@tonekk.de>
Date:   Sun Jun 12 14:31:39 2022 +0200
 
    Add User model
```

Let's assume the commits above all belong to the same pull-request. We can see two faulty patterns here:

1. Pre-mature commits that have been corrected right away but using another commit
2. Mistakes in the code that have been discovered while still working on the same PR

In a flawless world, the fixes of those errors would be included in the commits that introduced them. It does not provide any benefits for my coworkers to know that I fixed them afterwards.
Let's explore how we can accomplish that, starting with the former *faulty pattern*.


## Pre-mature commits

Committing prematurely is a genuinely humane thing. It happens on a daily basis. After having finished writing that commit message, my eyes wander back to the editor where I discover: Damn it, there is a typo in my code.
I can feel my blood pressure rise, and without a glimpse I switch back to my editor, correct that lousy mistake, head back to my terminal and do what every developer would do...

{% highlight bash %}
$ git commit -a -m "Fix typo"
{% endhighlight %}

Fixed! Except for the fact that my colleague always bothers me with "squashing" my commits... What is this guy talking about?


### Squashing

Squashing is a feature in git that allows you to combine a bunch of commits into one. This is done by performing a so-called "interactive rebase".
Typing 

{% highlight bash %}
$ git rebase -i HEAD~5
{% endhighlight %}

will result in an editor window (I use nvim) displaying the following:

```
pick 9b1d567 Add User model
pick 997a13f Fix User model
pick b57080f Add User spec
pick 27b0225 Fix User spec
pick 25710bf Fix typo in User model
```

In this example, we would want to squash `Fix User model` into `Add User model` and `Fix User spec` into `Add User spec`.

*Note: `HEAD` refers to the tip commit of the current branch. The tilde notation let's us move up the ancestor chain of that commit. It is also worth mentioning that HEAD~5 does not point to 9b1d567 (Add User model) as you might expect - it points to its parent. When rebasing interactively like this, you always have to specify the ancestor of the last commit that is relevant for your fix up.*

To accomplish that, we need to replace the keyword `pick` before the commits that we want to squash with the keyword `squash`. When we close our editor, git will begin with the rebasing process. You will see your editor pop up when git hits a squashed commit, displaying the two commits messages beneath each other. I always found this behavior pretty annoying - in 95% of the cases I wanted the upper commit message to stay as it was, so I deleted the commit messages of the squashed commit and moved on. Years went by, before I learned that there was a way to skip that repetitive nonsense.


#### Fixup

By replacing the `pick` keyword with `fixup` instead of `squash`, git will preserve the commit message of the parent commit that we melt the other commit into. This makes the process fully automatic, saving our precious time and energy.

But, there is still a gotcha. Imagine you just committed, stretching out your arms while leaning back on your office chair, only to have this brief moment of comfort destroyed instantaneously: One glance back at your code, and you discover... a typo!
With our current strategy, you would have to deal with your rage while smashing a seemingly endless series of  three commands into your helpless keyboard (lord have mercy):

{% highlight bash %}
$ git add FileWhereIJustFixedTheTypo.js
$ git commit -m "F*****ing fix that stupid typo"
$ git rebase -i HEAD~2
{% endhighlight %}

**and** replace that `pick` with a `fixup`... Maybe you think I'm exaggerating a bit here, but be honest with yourself: You are not good in those moments of burning fury. Better to keep them as short as possible. Fortunately, there is a shortcut.

### Amend

I had heard about the `--amend` switch of the `git commit` command a while back. In fact, I used it quite regularly to edit the commit message of my HEAD commit. What I didn't know until quite recently: `git commit --amend` can be used to add changes to the last commit.
To accomplish what we did above, we would now only have to type

{% highlight bash %}
$ git add FileWhereIJustFixedTheTypo.js
$ git commit --amend
{% endhighlight %}

Then, our editor opens and displays the commit message of the commit that we are altering. To avoid that, we can add the `--no-edit` flag. If we don't have any other unstaged changes, we could even throw in the `--all` flag to tell git that we want all current changes to be committed, making this whole thing a one-liner.

{% highlight bash %}
$ git commit --amend --all --no-edit
{% endhighlight %}


## Discovering mistakes later on

Some years back, when I was still pretty new to software development and git, I used to work on a feature until it was finished without doing a single commit in between.
Then, I executed git status and got completely flabbergasted by the long list of changed files... I had lost track! What did I even do, and how can I split that into reasonable chunks now?!

Today, I'm trying to force myself to a *commit early*-kind of strategy. Change one "unit" of code, add one behavior or do one part of the feature (e.g. the backend part) and commit that.
While this strategy works well to keep an overview of what I worked on (provided I wrote proper commit messages), it leaves room for discovering tiny changes needed in code that I already "finished". Changes, that would be better off merged into the commit where I worked on that chunk. In our example above, it would be this commit:

```
pick 25710bf Fix typo in User model
```

that we would like to melt into this one:

```
pick 9b1d567 Add User model
```

Let's see how we can achieve that. The examples below assume that we did not fix that typo yet, so the "bad" commit 25710bf does not exist.

### Interactive rebase + amend
The first way to do this, is to combine the two commands we just learned. We are going to rebase interactively, but instead of squashing, we will replace the `pick` keyword with `edit` this time.

*Note: Instead of using `HEAD~{n}`, we can also use a commit hash with `git rebase -i`. Will have to prepend that hash with `^` though because we need the parent of the commit.*

{% highlight bash %}
$ git rebase -i 9b1d567^
{% endhighlight %}

```
edit 9b1d567 Add User model
pick 997a13f Fix User model
pick b57080f Add User spec
pick 27b0225 Fix User spec
```

Then we apply the change using `--amend` as we learned above: 

{% highlight bash %}
$ git commit --amend --all --no-edit
{% endhighlight %}


And finish the process using

{% highlight bash %}
$ git rebase --continue
{% endhighlight %}

### Fixup commit and autosquash
Git wouldn't be git if there was only one way to do this. Let me introduce you to fixup commits and interactive rebase with the `---autosquash` switch. In this example, we first fix the mistake that we made and the commit using the fixup flag like and the hash of the commit we want to alter:

{% highlight bash %}
$ git commit --all --fixup=9b1d567
{% endhighlight %}

Now we execute our interactive rebase with the autosquash.

{% highlight bash %}
$ git rebase -i "9b1d567^" --autosquash
{% endhighlight %}

The output will be this:

```
pick 9b1d567 Add User model
fixup f7017f7 fixup! Add User model
pick 997a13f Fix user model
pick b57080f Add User spec
pick 27b0225 Fix User spec
```

As we can see, our fixup commit was placed write below the commit that we wanted to fix.

To speed up the process even more, we can add the following to our gitconfig:

```
[alias]
  fixup = "!fn() { git commit --fixup ${1} && GIT_EDITOR=true git rebase --autosquash -i ${1}^; }; fn"
```

Then

{% highlight bash %}
git fixup 9b1d567
{% endhighlight %}

will handle the whole everything process in one command.


## Wrap up

Today, we learned that we can fix all the commits! By doing so, we can avoid littering up our git history with useless `Fix foobar` / `Fix typo` commits. We learned about amending and interactive rebasing to fix the most recent commit, as well as combining the two to fix older commits. Lastly, we introduced fixup commits with git's autosquash feature.

I hope this blog post provided some value for you, feel free to reach out if you have any questions.



