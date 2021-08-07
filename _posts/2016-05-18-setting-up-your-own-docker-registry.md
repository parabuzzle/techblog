---
layout: post
title:  Setting Up Your Own Private Docker Registry
date:   2016-05-18 12:00:00
categories: Docker
preview: If you want to run your own Docker Registry using registry-v2 this post is for you! Its pretty simple but there are a few gotchas you need to know about. The purpose of this blog post is to help you past the problems so you experience the joy of setting up and having your own private docker registry at little to no cost (depending on if you want to purchase SSL certs or not.)
og_type: article
og_description: If you want to run your own Docker Registry using registry-v2 this post is for you! Its pretty simple but there are a few gotchas you need to know about. The purpose of this blog post is to help you past the problems so you experience the joy of setting up and having your own private docker registry at little to no cost (depending on if you want to purchase SSL certs or not.)
disqus_id: 16
---

If you want to run your own Docker Registry using registry-v2 this post is for you! Its pretty simple but there are a few gotchas you need to know about. The purpose of this blog post is to help you past the problems so you experience the joy of setting up and having your own private docker registry at little to no cost (depending on if you want to purchase SSL certs or not.)


### Assumptions

  * You are doing this on a mac
  * You have installed the latest Docker Toolkit
  * You are using `docker-machine`
  * You know how to use `docker` & `docker-machine`

### Let's Get Started

In this post I'm going to use my own versions of the containers because I've added some things that make this way easier. Feel free to explore the GitHub repositories and make your own or fork if you like :)


Let's look at the `Dockerfile` of [my version](https://hub.docker.com/r/parabuzzle/registry) of the registry:

~~~
FROM registry:2.4.0
MAINTAINER Mike Heijmans <parabuzzle@gmail.com>

COPY config/config.yml /config/config.yml

VOLUME /config
VOLUME /etc/ssl

CMD ["serve", "/config/config.yml"]
~~~

As you can see we are using registry:2.4.0 and overriding the config.yml. I'm also exporting a `/config` volume and a `/etc/ssl` volume. This way you can make your own config.yml and just mount the directory that its in as `/config` and the same can be done for putting SSL certs in too.


Let's look at the `config.yml`

~~~yaml
version: "0.1"
log:
  fields:
    service: registry
storage:
  filesystem:
    rootdirectory: /var/lib/registry
  cache:
    blobdescriptor: inmemory
  maintenance:
    uploadpurging:
      enabled: true
      age: 168h
      interval: 24h
      dryrun: false
http:
  addr: 0.0.0.0:5000
  secret: insecure
~~~

This is as simple as it gets. It runs the registry on `http` instead of the default `https` because `tls` isn't defined here.

So by default if you run this container, you will get the insecure registry running. (great for running behind a firewall)

~~~
docker run -d -v /registry:/var/lib/registry parabuzzle/registry:2.4.0.0
~~~
Awesome! so its running now!

**DO NOT EXPOSE THIS TO THE PUBLIC INTERNET LIKE THIS!**

*We'll talk about security later in the post*

Let's push something to it! We already have the container for the registry, lets put that in our new local registry...

~~~
docker tag parabuzzle/registry:2.4.0.0 localhost:5000/registry:2.4.0.0
docker push localhost:5000/registry:2.4.0.0
~~~

Let's slap a frontend on this so we can browse our registry... for this we'll use my [Crane Operator](https://hub.docker.com/r/parabuzzle/craneoperator) app:

~~~
docker run -d \
  -p 80:80 \
  -e REGISTRY_PROTO=http \
  -e REGISTRY_HOST=`docker-machine ip` \
  parabuzzle/craneoperator:latest
~~~

Now we can navigate to our `docker-machine ip` in a browser and see that we have a the registry container in our registry (how meta).

YAY! Its working.. sort of... If you attempt to use the `docker-machine ip` instead of `localhost` you will get an error...

~~~
$> docker push 192.168.99.100:5000/registry:latest
The push refers to a repository [192.168.99.100:5000/registry]
Get https://192.168.99.100:5000/v1/_ping: tls: oversized record received with length 20527
~~~

This is because your docker client assumes TLS is setup and working. So we need to turn this off. *(again, we'll talk about security later)*

On our docker-machine, let's set this server as an insecure registry:

~~~
docker-machine ssh <machine>
sudo vi /var/lib/boot2docker/profile
~~~

(in my case my `docker-machine ip` is `192.168.99.100`) so we need to add the `--insecure-registry 192.168.99.100:5000` to the `EXTRA_ARGS`:

~~~
EXTRA_ARGS='
--label provider=virtualbox
--insecure-registry 192.168.99.100:5000
'
~~~

Lastly, just restart docker on your machine:

~~~
sudo /etc/init.d/docker restart
~~~

(you will need to restart your registry and crane operator too)

Now that you've done this you should be able to push and pull to that ip without tls support:

~~~
$> docker push 192.168.99.100:5000/registry:latest
The push refers to a repository [192.168.99.100:5000/registry]
560a9d4ac0fb: Pushed
655fda973434: Pushed
5f70bf18a086: Pushed
d73a4514b81b: Pushed
ec5e6f89215b: Pushed
7ef965e0bbca: Pushed
6eb35183d3b8: Pushed
latest: digest: sha256:02990be74348da61dd174f8041cf3072937ee7bda9a05b7731c1567033e45b31 size: 2580
~~~

This is on the client and not on the server! so if you want to use an insecure registry *(like behind a firewall)* you will need to set this on all machines that will be pulling or pushing containers to this registry.

You can set this automatically when you build your docker-machine image like this:

~~~
docker-machine create --driver virtualbox \
  --engine-insecure-registry myregistry:5000 default
~~~

(This is considered the correct way to do this and will prevent the setting from being lost on VM reboot)


### Great! how do I secure it?

Ok, so I promised to explain how to secure it... I will explain how to fix your registry from needing the `--insecure-registry` flag on all your clients but I'm not going to cover password authentication. That may be a follow up post (or it may not because I don't need this myself).

This assumes that you're going to run your registry behind a firewall and that password authentication is not needed. Just container integrity and TLS over the wire.

So, let's say you're going to run the registry at `registry.mydomain.com` and you have SSL certs for that domain name.

On your server, put your cert and key in `/etc/ssl` and make a directory for your config like `mkdir -p /var/registry/config`.

In your `/var/registry/config` directory let's put a `config.yml` like this:

~~~yaml
version: "0.1"
log:
  fields:
    service: registry
storage:
  filesystem:
    rootdirectory: /var/lib/registry
  cache:
    blobdescriptor: inmemory
  maintenance:
    uploadpurging:
      enabled: true
      age: 168h
      interval: 24h
      dryrun: false
http:
  addr: 0.0.0.0:5000
  secret: randomsecret12312124124123121
  tls:
    certificate: /etc/ssl/mycert
    key: /etc/ssl/mykey
~~~
make sure to replace the `secret` with something random that you make up and that the `certificate` and `key` point to your cert and key that you put in `/etc/ssl`.

now we run the app and mount those volumes:

~~~
docker run -d \
-p 5000:5000 \
-v /registry:/var/lib/registry \
-v /var/registry/config:/config \
-v /etc/ssl:/etc/ssl \
parabuzzle/registry:2.4.0.0
~~~

This will use your config file and certs.

And TADA! no more `--insecure-registry` needed!

for full docs on the cool things you can tweak in the `config.yml` check out the [docs](https://docs.docker.com/registry/configuration)

#### Ok I lied.. here's how to setup password auth

create an `htpasswd` file in your `/var/registry/config` directory on the host with your username and password entries:

**NOTE: Only bcrypt passwords are supported!**

*(you can generate a bcrypt htpasswd entry [here](http://aspirine.org/htpasswd_en.html)*

~~~
admin:$2y$11$6c5OT3idh3VJxgXoqrkmAO9spPnWOMIKyP.Mgf1ynJ.Ximi2DZwie
engineer:$2y$11$6c5OT3idh3VJxgXoqrkmAO9spPnWOMIKyP.Mgf1ynJ.Ximi2DZwie
~~~

set it in your `config.yml`

~~~yaml
version: "0.1"
log:
  fields:
    service: registry
storage:
  filesystem:
    rootdirectory: /var/lib/registry
  cache:
    blobdescriptor: inmemory
  maintenance:
    uploadpurging:
      enabled: true
      age: 168h
      interval: 24h
      dryrun: false
http:
  addr: 0.0.0.0:5000
  secret: randomsecret12312124124123121
  tls:
    certificate: /etc/ssl/mycert
    key: /etc/ssl/mykey
auth:
  htpasswd:
    realm: basic-realm
    path: /config/htpasswd
~~~

run the registry:

~~~
docker run -d \
-p 5000:5000 \
-v /registry:/var/lib/registry \
-v /var/registry/config:/config \
-v /etc/ssl:/etc/ssl \
parabuzzle/registry:2.4.0.0
~~~

login on your client:

~~~
$> docker login myregistry:5000
$> Username: admin
$> Password: password
$> Login Succeeded
~~~

You'll also need to give some credentials to your Crane Operator to browse around:

~~~
docker run -d \
  -p 80:80 \
  -e REGISTRY_HOST=mysecureregistry \
  -e REGISTRY_USERNAME=admin \
  -e REGISTRY_PASSWORD=password \
  parabuzzle/craneoperator:latest
~~~

all done! you are good to go :)

