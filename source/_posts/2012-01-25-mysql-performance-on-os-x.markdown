---
layout: post
title: "MySQL Performance on OS X"
date: 2012-01-25 22:50
comments: true
categories: mysql performance osx lion
---
Ever wondered why performance of MySQL on OS X was so much worse than a Linux or windows counterpart - especially when doing DROP/CREATE statements (ie in the setup of unit tests etc)? It turns out Mac OS has a safer file creation function than is available on other operating systems and the use of that slows down performance of file operations in MySQL.

You can fix this quickly by adding the following to your <strong>my.cnf</strong> file in the<strong> [server]</strong> block. It's probably best not to do this in production, but who actually uses OS X as a production web server anyway?
<pre>skip-sync-frm=OFF</pre>
Source: <a href="http://bugs.mysql.com/bug.php?id=56550">The original MySQL bug report</a> (although it's not really a bug).