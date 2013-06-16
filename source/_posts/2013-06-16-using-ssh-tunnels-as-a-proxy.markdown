---
layout: post
title: "Using SSH tunnels as a proxy"
date: 2013-06-16 10:16
comments: true
categories: mysql ssh tunnel aws
---
In my day job, our application is hosted in Amazon's AWS service using an autoscaled cluster and their RDS service for MySQL (among many other aspects of their environment). In this configuration the web servers boot up and shut down in order to respond to the current load of our users. This is great and all, but means that their IP addresses are always changing.

Our RDS is configured such that only the machines in our autoscaling group are allowed to access it. This is also great, but it means that if I want to look into something in the production database, I need to SSH to one of the web servers (after working out its IP address) and _then_ connect to RDS.

This can be so much easier.<!-- More -->

SSH tunnels sounded like the easy option here, but by default, that lets you access services on a remote machine. In this setup, we want to access services not _on_ a machine, but _via_ that machine. We also don't really care to keep that terminal session open.

``` bash
ssh -i <keyfile> -o StrictHostKeyChecking=no -f -L <your local port>:<proxied-to-host>:<proxied-to-port> <proxy-user>@<proxy-host> -N &
```

Explanation of each of the parts of that command:

* ``` -i <keyfile> ``` Include this if your default key isn't the one that can access your proxy (&lt;proxy-user&gt;@&lt;proxy-host&gt;).
* ``` -o StrictHostKeyChecking=no ``` This is useful when you're always connecting to a new host and don't want to type yes each time it happens to be a new host.
* ``` -f ``` As per the man page, Requests ssh to go to background just before command execution. This is useful if ssh is going to ask for passwords or passphrases, but the user wants it in the background.
* ``` -L ``` This is what sets up the proxy instead of just getting you to the remote host.
* ``` <your local port> ``` As per normal tunnels, you need to specify a local port that you want to access this service on.
* ``` <proxied-to-host> ``` This is the host name for the service you want to access via this proxy. In my case, it's an RDS host.
* ``` <proxied-to-port> ``` This is the remote port for the service you want to access via this proxy. In my case, it's ```3306```.
* ``` <proxy-user> ``` This is the username of the account you can normally SSH to. If you're using standard AWS AMIs, this is likely ```ec2-user```.
* ``` <proxy-host> ``` This is the hostname of the account you can SSH to.
* ``` -N ``` As per the man page, Do not execute a remote command.  This is useful for just forwarding ports.

This means that the command for me to access my RDS instance locally becomes something like:
``` bash
ssh -i ~/.ssh/ec2-ap-southeast-2.pem -o StrictHostKeyChecking=no -f -L 3000:myhostname-southeast-2.rds.amazonaws.com:3306 ec2-user@ec2-somethingmagic.ap-southeast-2.compute.amazonaws.com -N &
```

After running this, I can connect to port 3000 on my local machine which will be routed via my EC2 instance and straight to the RDS instance I have configured, allowing me to use local tools like phpMyAdmin or Sequel Pro etc in order to access the database instead of having to do it all through the shell. (I don't hate the shell, it can just be easier some times to have a GUI for interacting with the DB).

One thing the above doesn't solve however is how to get the host name when the host names keep changing due to autoscaling.

Assuming you have the AWS utilities installed, you can do something like the following:

``` bash
export AWS_ACCESS_KEY="xxxxxxxxx"
export AWS_SECRET_KEY="yyyyyyyyyyyy"
export EC2_HOME="/usr/local/share/aws/ec2-api-tools"
export JAVA_HOME=/usr
/usr/local/share/aws/ec2-api-tools/bin/ec2-describe-instances --region ap-southeast-2 --show-empty-fields -H --filter "tag:Name=<whatever you've named your cluster>" --hide-tags |grep INSTANCE | awk '{print $4}' |head -n 1
```

This will give you the first node in your autoscaling cluster.

I've combined this into a neat little script for my use which will go discover a valid, current node then establish the tunnel.

The MySQL client is also so good that if the tunnel drops out (eg autoscaling shuts down that node), all you have to do it reestablish the tunnel with another node and you can continue using it.

If you're using autoscaling and will need to periodically reestablish these tunnels, try adding something like the following to the start of your script:
```
kill `lsof -t -i tcp:3000`
```
This kills whatever is running on that port - most likely your dead tunnel.

Good luck, let me know in the comments if it worked for you.