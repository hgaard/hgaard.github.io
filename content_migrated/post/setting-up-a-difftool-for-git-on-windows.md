+++
author = "Jakob Højgaard"
date = 0001-01-01T00:00:00Z
description = ""
draft = false
slug = "setting-up-a-difftool-for-git-on-windows"
title = "Setting up a difftool for git on windows"

+++

Well i have been using [DiffMerge](https://sourcegear.com/diffmerge/) from SourceGear as the default diff tool in Visual Studio for quite a few years now. I use git more and more now but I never took the time to add a diff tool to the git configuration. But since i recently got a new dev machine I thought i might just set that up as well.

### Installation
I installed with [Chocolatey](http://chocolatey.org/) (remember to run cmd og PowerShell as admin) which i use to manage most of my dev tools. In fact i maintain a script on github with all the tools I use, so if you haven't used Chocolatey for application installation, now's your chance. 

`choco install diffmerge`

### Configuraion
The following [Guide](https://sourcegear.com/diffmerge/webhelp/sec__git__windows__msysgit.html) is just copied from SourceGear and works at least for version 4.2.x

For diff:

	git config --global diff.tool diffmerge

	git config --global difftool.diffmerge.cmd "C:/Program\ Files/SourceGear/Common/DiffMerge/sgdm.exe `$LOCAL `$REMOTE"

For merge:
	
    git config --global merge.tool diffmerge

	git config --global mergetool.diffmerge.trustExitCode true

	git config --global mergetool.diffmerge.cmd "C:/Program\ Files/SourceGear/Common/DiffMerge/sgdm.exe /merge /result=`$MERGED `$LOCAL `$BASE `$REMOTE\"
        
Well of course you can also just edit the .gitconfig and add the settings directly into it. That would look something like this.

	[diff]
        tool = diffmerge
	[difftool "diffmerge"]
        cmd = "C:/Program\ Files/SourceGear/Common/DiffMerge/sgdm.exe $LOCAL $REMOTE"
	[merge]
        tool = diffmerge
	[mergetool "diffmerge"]
        trustExitCode = true
        cmd = "C:/Program\ Files/SourceGear/Common/DiffMerge/sgdm.exe /merge /result=$MERGED $LOCAL $BASE $REMOTE"

