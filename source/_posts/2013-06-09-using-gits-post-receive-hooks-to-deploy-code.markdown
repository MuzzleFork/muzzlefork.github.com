---
layout: post
title: "Using git's post-receive hooks to deploy code"
date: 2013-06-09 13:24
comments: true
categories: git digitalocean code deployment hooks automation
---
I've recently started self-hosting some sites on some cheap, cool hosting provided by [Digital Ocean](https://www.digitalocean.com/?refcode=1d72a90eb581). On one of their VPSs, I've set up some bare git repositories and made some git hooks to automate deployment of sites to that server.<!-- more -->

This information isn't limited to only working on Digital Ocean, but my walk-through is based on that environment - using their LAMP stack image based on Ubuntu 12.04 using an SSH key (makes life so much easier :))

This setup uses Apache and some virtual hosts to handle each site. The default setup in Ubuntu for apache involves 2 folders - ```/etc/apache2/sites-available``` (where you can put a virtual host directive) and ```/etc/apache2/sites-enabled``` (where you symlink files from the sites-available folder in order to enable them). Using the ```sites-available```/```sites-enabled``` combo allows you to disable sites without removing them. This functionality isn't required here since we're going to use git to version control and getting old versions out of config is easy.

## Set up git repository for the virtual hosts
I'm using ```/var/repos``` for all of my repos. You're welcome to use any other path you see fit.
``` bash
mkdir -p /var/repos/virtualhosts.git
cd /var/repos/virtualhosts.git
git init --bare
```

This creates a bare repository (doesn't include a working folder) ready for use.

While you're in there, modify the post-receive hook (my favourite editor is vim = ```vim hooks/post-receive```)
Put the following code in there
``` bash
GIT_WORK_TREE=/etc/apache2/sites-available git checkout -f

# a2dissite all files in /etc/apache2/sites-enabled
cd /etc/apache2/sites-enabled
for f in * ; do a2dissite $f; done

# ensite the ones that are present now in /etc/apache2/sites-available
cd /etc/apache2/sites-available
for f in * ; do a2ensite $f; done

service apache2 reload
```
Then you'll have to give it execute permission
``` bash
chmod +x hooks/post-receive
```

This will:

1. Check out the files in the repo to ```/etc/apache2/sites-available```
1. Disable all the existing sites that are linked
1. Enable all the sites that are present in ```/etc/apache2/sites-available```.
1. Tell Apache to reload its configuration

Now, on your development machine, you'll need to set this up as a remote and start pushing virtual host configurations to it.
``` bash
git clone ssh://user@host-name/var/repos/virtualhosts.git
```

You're now ready to start adding virtual host directives into that repo. When you push to ```origin```, it'll restart Apache with the updated config.

Here's a sample - this is modified from the default:
``` apacheconf
<VirtualHost *:80>
	ServerAdmin you@host.com

	DocumentRoot /var/sites/<some-domain-name>
	ServerName <some-domain-name>
	ServerAlias www.<some-domain-name>

	SetEnv db.name xxxxx
	SetEnv db.user xxxxx
	SetEnv db.password xxxxx
	SetEnv db.host localhost

	<Directory />
		Options FollowSymLinks
		AllowOverride None
	</Directory>
	<Directory /var/>
		Options Indexes FollowSymLinks MultiViews
		AllowOverride All
		Order allow,deny
		allow from all
	</Directory>

	ScriptAlias /cgi-bin/ /usr/lib/cgi-bin/
	<Directory "/usr/lib/cgi-bin">
		AllowOverride None
		Options +ExecCGI -MultiViews +SymLinksIfOwnerMatch
		Order allow,deny
		Allow from all
	</Directory>

	ErrorLog ${APACHE_LOG_DIR}/error.log

	# Possible values include: debug, info, notice, warn, error, crit,
	# alert, emerg.
	LogLevel warn

	CustomLog ${APACHE_LOG_DIR}/access.log combined

    Alias /doc/ "/usr/share/doc/"
    <Directory "/usr/share/doc/">
        Options Indexes MultiViews FollowSymLinks
        AllowOverride None
        Order deny,allow
        Deny from all
        Allow from 127.0.0.0/255.0.0.0 ::1/128
    </Directory>

</VirtualHost>
```

## Set up git repo for each site
So, we can see that virtual hosts are added to Apache whenever you add a new config, but those virtual host configurations are pointing to locations that don't yet exist. Let's fix that now, back to your remote server.

``` bash
mkdir -p /var/sites/<some-domain-name>
mkdir -p /var/repos/<some-domain-name>.git
cd /var/repos/<some-domain-name>.git
git init --bare
```

And to have this git repo deploy whenever it is pushed to, ```vim hooks/post-deploy```
``` bash
#!/bin/sh
GIT_WORK_TREE=/var/sites/<some-domain-name> git checkout -f master
```
Don't forget to make it executable:
``` bash
chmod +x hooks/post-deploy
```

It's ready to go! To check it out locally, you can:
``` bash
git clone ssh://user@host-name/var/repos/<some-domain-name>.git
```
or you could add it as a remote for an existing repository
``` bash
git remote add <repo-name> ssh://user@host-name/var/repos/<some-domain-name>.git
```

Push to that remote whenever you're ready for your changes to go live.
