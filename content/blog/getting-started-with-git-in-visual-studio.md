+++
author = "Jakob Højgaard"
categories = ["Visual Studio", "git"]
date = 0001-01-01T00:00:00Z
description = ""
draft = false
slug = "getting-started-with-git-in-visual-studio"
tags = ["Visual Studio", "git"]
title = "Getting started with git in Visual Studio"

+++

The last year or so at [Schultz](http://schultz.dk) we have started to use git for more and more projects, and I have become the goto guy for git related issues. The last few times new developers have joined my project or started a new project using I have compiled an email with useful links on git and git from Visual Studio. So thought I might just put it in a blog post instead. 


### Getting git under your skin
First of all i think it is extremely important understand the paradigm of git, and for most of my fellow .net developers the difference between a centralized source control system - like TFS, and a distributed like git. I think these resource are a great place to start:

* [https://www.atlassian.com/git](https://www.atlassian.com/git)
* [http://rogerdudler.github.io/git-guide/](http://rogerdudler.github.io/git-guide/)

For a litle deeper insight, have a look at this one:

* [http://think-like-a-git.net/](http://think-like-a-git.net/)

### Try it out
With a bit of background, just go ahead and get typing! I really recommend starting at the console, and Visual Studio will install the git client for you. But since playing around with git requires changing stuff in files, it can be a lot easier to practice in a more controlled environment. These two resources are great for that:

* [http://www.wei-wang.com/ExplainGitWithD3/](http://www.wei-wang.com/ExplainGitWithD3/)
* [http://pcottle.github.io/learnGitBranching/](http://pcottle.github.io/learnGitBranching/)
 
### Using git form Visual Studio
Since Visual Studio 2013 the git source control provider has been part of the installation, so no extra installs necessary. 

For detailed info on git in Visual Studio, MSDN has a great tutorial here: [http://msdn.microsoft.com/en-us/library/hh850437.aspx](http://msdn.microsoft.com/en-us/library/hh850437.aspx)

### A few caveats
Even though most tasks can be accomplished from Visual Studio, I think it is necessary to get familiar with other tools like [SourceTree](http://www.sourcetreeapp.com/) or [PoshGit](https://chocolatey.org/packages/poshgit), which also just might make you more productive.
Further there are some very useful features of git like below are not possible form Visual Studio:

* rebase
* stash
* cherry-pick


### Other resources
* [Git casts](https://www.youtube.com/playlist?list=PLttwD7NyH3omQLyVtan0CFOX_UWItX_yG)
* [SourceTree](http://www.sourcetreeapp.com/)
* [Git tools on chocolatey](https://chocolatey.org/packages?q=git)
* [Setting up a merge tool](http://blog.hgaard.dk/setting-up-a-difftool-for-git-on-windows/)

