---
layout: post
title:  "Proxy SSH Over SOCKS (the easy way)"
date:   2014-08-12 12:00:00
categories: SysAdmin
tags: [Tech, System Administration, Git, SSH, Tricks, DevOps]
preview: Today I had a strange problem. I needed to work with an internal git repo behind a firewall from my home on my personal laptop. The git server only allows SSH connections (git@git.corp.example.com) and doesn't support HTTP requests.
og_type: article
og_description: The simple way to proxy SSH over a SOCKS proxy without any additional software.
disqus_id: 3
---

Today I had a strange problem. I needed to work with an internal git repo behind a firewall from my home on my personal laptop (read: no vpn client). The git server only allows SSH connections (git@git.corp.example.com) and doesn't support HTTP requests.

<div class="note">
<b>note:</b> git supports HTTP proxies out-of-the-box for <a href="http://www.aireadfun.com/blog/2013/08/27/using-git-through-a-socks-proxy-or-ssh-tunnel/" target="_blank" alt="Using Git Through a SOCKS Proxy">HTTP based repos</a>
</div>

Anyway, there are a ton of solutions on the internet for how to do this sort of thing, but most of them are very confusing or require additional software to accomplish SSH based repo proxy-ing.

<img class="img-fluid" src="/img/postimgs/splat.gif"/>

**THERE MUST BE A BETTER WAY... right?**

There is a better way and I'm going to show you how I did it with minimal effort using only the functionality built into OpenSSH and netcat.

### 1. Setup Your Socks Proxy

If you didn't know this already, you can use OpenSSH to create a SOCKS proxy.

*(this is most useful for web browsing like you're on the box you connected to via SSH)*

You do this by adding the -D flag with a port to listen on:

```
ssh -D 1080 jumphost.corp.example.com
```

What this does is connect you to jumphost.corp.example.com just like a normal SSH session with one exception. It will also setup a SOCKS proxy on localhost:1080

You can test this by opening up your browser's network settings and setting your socks proxy to localhost with a port of 1080.

Now when you're browsing the web your requests are tunneled through the SOCKS server that OpenSSH provided for you with the -D flag and originate from jumphost.corp.example.com. *pretty nifty eh?*


### 2. Add The Proxy to Your SSH Config

So now that you have a SOCKS proxy running on localhost:1080, you can tell OpenSSH to use that tunnel for all SSH requests destined for a specific hostname. We will do this using netcat (nc).

Add this to your ```~/.ssh/config```:

```
Host git.corp.example.com
  ProxyCommand=nc -X 5 -x localhost:1080 %h %p
```

As you can see here, when we do ```ssh git.corp.example.com```, OpenSSH will actually proxy the network stream through localhost:1080 using netcat.

<div class="note">
  <b>note:</b> if you don't know what netcat is, you should! It sends raw data streams over a network connection.
</div>

What's happening is actually quite elegant. The data stream that OpenSSH would normally send directly to git.corp.example.com is actually sent through the SOCKS tunnel we made in step 1 via netcat's awesomeness.

### 3. Use Git Like A Boss

Now that we have started a SOCKS proxy locally (step 1) and told OpenSSH to use it for any SSH activity to git.corp.example.com (step 2), there is nothing more to do... other than use git like we normally would.

```
git clone git@git.corp.example.com/parabuzzle/jedimaster.git
```

This will result in pulling down the internal code that I need... (without requiring my VPN client)

**You're done! Go have a beer and bask in the glory of SysAdmin magic!**







