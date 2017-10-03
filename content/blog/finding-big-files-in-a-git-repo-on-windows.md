+++
author = "Jakob Højgaard"
date = 0001-01-01T00:00:00Z
description = ""
draft = false
slug = "finding-big-files-in-a-git-repo-on-windows"
title = "Finding big files in a git repo on windows"

+++

I recently joined a project that had been working on a product for a few years already. The team had used TFS for source control until just a few weeks before I joined. Because of external circumstances the team had decided to migrate the complete source tree to git (still hosted on TFS though).

After the migration to the new git repository it turned out that the repository was quite big, to more precise it was around 1,2 GB. This could of course just be ignored since TFS repositories with a few years of age can easily grow to that size - we stick anything in there don't we. Anyway with a  recent [story in mind](http://thedailywtf.com/articles/Forever-Alone) and because we had to do a lot of cloning of the repository, I decided to have a closer look at the repo to see if we had any big files checked in by mistake.

## Analyzing the repository
I found [this](http://naleid.com/blog/2012/01/17/finding-and-purging-big-files-from-git-history) article describing exactly this task, but the article described the process with bash commands and I work work mainly on Windows and all of my team mates do too. So I thought this might just be the case for me to do a little diving into more advanced PowerShell. It turned out to take a bit longer than expected. After all git still returns text not objects even on windows.

### Files and Hashes
To begin with, I use poshgit (installed with [Chocolatey](http://chocolatey.org)) to interact with git from PowerShell, and this is what i came up with after a bit/lot of help from my good college Uffe.

First, get all the hashes of files in the repo.

    $sortedHashes = git rev-list --objects --all | sls "^\w+\s\w" | ConvertFrom-Csv -Delimiter ' ' -head hash,path | %{$h=@{}}{$h[$_.hash]=$_.path}{$h}

Do a gc.

    git gc

Get the path of the index file.

    $indexFile = ".git\objects\pack\" + (".git\objects\pack" | gci -Filter 'pack-*.idx')

List all files ever in the repo ordered by size (this does not contain the filename and path).

    $bigfiles = git verify-pack -v $indexFile | %{$_  -creplace '\s+', ','} | sls "^\w+,blob\W+[0-9]+,[0-9]+,[0-9]+$" | ConvertFrom-Csv -Header hash, type, size | %{New-Object pscustomobject -Property @{Hash=$_.hash; Size=[int]$_.size}} | sort size -Descending

Match the hashes from the list of big files with the filenames from the sorted hashes to produce a list of the biggest files ever in the repo and send to a csv file.

    $bigfiles | select size,@{n='Name';e={$sortedHashes[$_.hash]}} | Export-Csv -Delimiter ';' -Path "big-files.csv" -NoTypeInformation

It turned out that there was a actually a big (~10 MB) zip file in the repo i quite a few versions, but no longer in the index. Next step - get rid of it!


## Rewriting history
Git has a very nice tool for this - filter-branch. Here it important to be very cautious. When doing these kind of operations it is very important delete working copies of the remote repo and do a fresh clone after the changes have been pushed back. Now let's get to it.

First, checkout all remote branches locally.

    git branch -a | sls -CaseSensitive remotes |  sls -CaseSensitive -NotMatch HEAD | sls -CaseSensitive -NotMatch master |%{
    "git branch --track $($_ -replace '^\s*remotes/origin/','') $_" | iex
    }

Remove the identified file and rewrite commit history.

    git filter-branch --prune-empty --index-filter 'git rm -rf --cached --ignore-unmatch THE-FILE-TO-REMOVE' --tag-name-filter cat -- --all

This can potentially take quite a bit of time...
Now since git does makes quite an effort not to loose data all the removed files are not actually yet removed from the repo, they are just not cleaned up yet. In order to fix this it is necessary to expire all reflogs and do a gc with the aggressive flag set as here

    git rm -rf .git/refs/original/
    git reflog expire --expire=now --all
    git gc --prune=now --aggressive

Success! My repo has shirked to about a third of the size before the cleanup.
Last step - push back the repository to TFS.

    git push --force --all

A few things started to go wrong here. The push stopped at about 7% of the upload and never resumed. After a bit of googling i realized it might be because of https. Luckily we also have a http endpoint inside our company firewall, so I just changed origin to use the default TFS http endpoint at port 8080.

    git remote set-url origin http://tfs.Endpoint:8080/tfs/path.to.repo

This made the upload succeed. Everything look just as they should, I could browse the code on the TFS website and i made a new clone just as a last check. This is when the second strange thing happened. When cloning the repo it was the same size as before the cleanup!

This seems quite odd and i have not yet found an answer to why this is. I know that running the expire reflog and gc commands on the newly cloned repo will shrink it to the same size as the one I force pushed, but other than that, this is where i got to. I will file a bug report with Microsoft and keep you updated.

**Update:**
I Just got an [answer from microsoft](http://connect.microsoft.com/VisualStudio/feedback/details/1019193/unable-to-clean-a-git-repo-in-tfs) and I'll paste their answer here:
> TFS does not currently perform garbage collection on git objects. We are aware that this is an important feature to have and we are tracking it in our backlogs. Currently, if you perform a git clone, TFS attempts to optimize for processing speed and hands you all objects that were associated with that repo without filtration, under the assumption that almost all of it will be live/reachable. After a filter operation like the one you performed, it might be prudent to delete the entire git repository from your TFS server and create a new one using the locally pruned and repacked git repo. All the standard caveats when deleting a repository apply - you lose all permission information and any TFS state associated with that repository and you will have to recreate them appropriately.
