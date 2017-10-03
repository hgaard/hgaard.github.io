+++
author = "Jakob Højgaard"
date = 0001-01-01T00:00:00Z
description = ""
draft = true
slug = "building-a-git-deploy-for-node-web-sites"
title = "Building a git deploy for node web sites"

+++

# Setup deployment
Since Rob Conery has a great course on Pluralsight - [Hacking Ghost](todo) - I'll refer to that for in detail description and just state the necessary installations here.
##Building a post-receive script

Run and develop local on a mac.
When moving to my Ubuntu server on DigitalOcean, a few things needed to be updated. First of all, the path... Secondly it is necessary to add a few commands to the sudoer file

### Images
Symlink