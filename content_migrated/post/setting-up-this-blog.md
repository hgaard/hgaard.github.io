+++
author = "Jakob Højgaard"
categories = ["Getting Started", "ghost", "blogg"]
date = 0001-01-01T00:00:00Z
description = ""
draft = false
image = "/images/2015/11/20151110_223603558_iOS-copy-1.jpg"
slug = "setting-up-this-blog"
tags = ["Getting Started", "ghost", "blogg"]
title = "Setting up this blog"

+++

# Introduction
I have had this blog brewing for a while, but have not really had the time to finish it before - I might get back to why in a later post.  

Anyway here goes: 

I started out hosting it on Heroku. It's a great way of hosting web applications and you can start out on their free tier which also has a lot of free add-ons like Postgres and New Relic. How ever i like to tinker and a Pluralsight course - [Mastering Your Own Domain](https://app.pluralsight.com/library/courses/master-domain/table-of-contents) by Rob Conery - inspired me to be more in control. For instructions on how to run Ghost on Heroku I used [this](http://www.therightcode.net/deploy-ghost-to-heroku-for-free/) guide by [Greg Bergé](http://www.therightcode.net/).

# The basics
The blogging software is [Ghost](http://tryghost.org), which is based in node and I use Postgres as datastore and have an nginx server running in front of node. The whole thing runs on a Ubuntu 15.10 server on [DigitalOcean](http://digitalocean.com).

# The how
The Pluralsight course mentioned earlier basically describes how to setup Ghost on a new server droplet on DigitalOcean. So for detailed description have a look at that.

##Setting up the server
**Create droplet**

Head over to Digital, create an account if you don't already have one and then create a new virtual machine which at Digital Ocean is called a Droplet. I use the smallest instance they provide with Ubuntu 15.10. Remember to check the backup checkbox, it's $1 more a month but having a backup is nice. Last thing; add a SSH key for convenience when logging on to the machine. When the machine is ready (usually takes around 1 minute) ssh into it to get working.

   
 
**Setup a user**

    adduser ghost
    visudo	     # And then assign super user privileges to ghost user
    #Add ssh key to the new user (Install ssh-copy-id with homebrew first
    ssh-copy-id <username>@<host> 

**Now install all the necessary software**   

    
   
**Install nginx**

    sudo apt-get install nginx -y
    
**Intstall Postgres and dev lib**

    sudo apt-get install postgresql libpq-dev

**Add ghost user to postgress**

    sudo su postgres # Switch to postgress user
    psql             # Open terminal to postgres
    create database <ghost-db-name>;
    create role <db-user> with password '<uder-pw>' LOGIN; #Create role and grant login
    alter database <ghost-db-name> owner to <db-user>; #transfer ownership
    

**Install Node**
    
    sudo apt-get install curl # Install curl to obtain node
    curl --silent --location https://deb.nodesource.com/setup_0.12 | sudo bash -
    sudo apt-get install --yes nodejs npm
    sudo ln -s /usr/bin/nodejs /usr/bin/node # Add symlink

**Install Ghost**

    wget https://ghost.org/zip/ghost-0.7.1.zip # Version 0.7.1 is latest at the time of writing
    unzip ghost-0.7.1.zip -d <folde-to-run-ghost-from>
    
**Prepare and start Ghost**

    npm install --production
    npm start
    
Now this will starts ghost on the server running the default configuration, so we'll need to edit the config.js in the root of the ghost installation to user Postgres instead of SQLite and for it to know the blog root url. The configuration should be changed to something like this:

      production: {
        url: '<blog-url>',
        database: {
          client: 'postgres',
          connection: {
              host     : 'localhost',
              user     : '<db-user>',
              password : '<user-pw>',
              database : '<db-name>',
              charset  : 'utf8'
          },
          debug: false
      },


Since I would like to hand over the webserver part to nginx (and have it runnign on port 80) i need to configure it to proxy to node. There are different ways to do so and since I'm no nginx wizard I chose the one I have used before.

**Add config file to nginx**

    sudo touch /etc/nginx/sites-available/ghost
    sudo vim /etc/nginx/sites-available/ghost
    
**Configure proxying in nginx**

    server {
        listen 80;
        server_name your_domain.tld;
        location / {
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   Host      $http_host;
            proxy_pass         http://127.0.0.1:2368;
        }
	}

**Symlink to sites-enabled and restart nginx**

    sudo ln -s /etc/nginx/sites-available/ghost /etc/nginx/sites-enabled/ghost
    sudo service nginx restart

**Keep ghost up and running with pm2**

    sudo npm install pm2 -g 	# Install
    pm2 startup 				# Ensure pm2 starts the app in system startup
    NODE_ENV=production pm2 start index.js # Startup production instance
    
    

## That's it
That concludes my basic setup of this blog. Next up:

* Adding comments with Discus and insights with Google Analytics
* Setting up deployment with git
* Running and maintaining a Ghost blog with Docker