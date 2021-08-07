---
layout: post
title:  My Clever Docker Environment Loading Solution Using Vault
date:   2016-09-28 12:00:00
categories: Docker
preview: The Problem, You have a dockerized application that needs 50+ environment variables that are different based on the execution environment. Let me explain how I made this completely painless with some clever design and hashicorp's vault
og_description: The Problem, You have a dockerized application that needs 50+ environment variables that are different based on the execution environment. Let me explain how I made this completely painless with some clever design and hashicorp's vault.
disqus_id: 17
---

## The Problem

You have a dockerized application that needs 50+ environment variables that are different based on the execution environment (ie: production, staging, mobile, etc).

The first solution would be to pass them in using the `-e` flag. But that can get tedious and is super error prone.. so you make an execution script that wraps it all nicely. Cool.

**But wait!** That looks a lot like docker compose so what about using that? Now you have compose files. Awesome.

**But wait!** Now you need to protect the sensitive variables like the Postgres password or your application's oauth credentials? How do you do that? A great solution for that would be [Hashicorp's Vault](https://www.vaultproject.io/) and after you put your super secret secrets into the vault, how do you get them out at runtime?

There are many solutions for interacting with vault inside your application directly but since you followed the 12factor design pattern and put everything into your environment its probably going to take a bit of work to rewrite your secrets loading to use the vault directly via your application. And why? You've already done the most versatile thing by loading through your environment? Plus, what happens when a new & better wizzbang way of saving and fetching secrets comes out? Then what? Are you going to do all that work again?! No way!

## My solution

**Load the environment at run time from vault.** It gives you the flexibility of differing credentials based on runtime environment that are secured by the Vault **without the headache** of changing your application code or getting locked into a single solution.

Our 50+ environment variables passed in at runtime are reduced to 2 or 3. We will need to pass in the environment (ie: production or staging), the vault token, & maybe the vault server url (although you could just build this into the base container).

The idea here is that we will save our super secret secrets namespaced by runtime environment.

Example:

~~~ bash
/secret/staging/postgres/password #=> The staging postgres password
/secret/production/postgres/password #=> The production postgres password
~~~

This way, we can apply the passed in environment to the path and get the correct secret.

### The Parts

You're going to need a few scripts/libraries. Here are the ones I've built (feel free to use them or adapt it).

#### The Vault Loader

This is a wrapper library for working with the vault that applies some automatic namespacing.

<script src="https://gist.github.com/parabuzzle/2fce63fb4879fd19d01fa77061751707.js"></script>

Basically this library adds the ability to run things like

~~~ ruby
vault.read('postgres/password')
~~~

which will fetch the value from vault at

~~~
/secret/development/postgres/password
~~~

You can also do a

~~~ ruby
vault.write('secret_key', 'secret_value')
~~~

which will write the value to the appropriate key

~~~
/secret/development/secret_key #=> secret_value
~~~

This library wraps all the path handling for you. We will use this library to create a script we use for loading environment variables.


#### Vault Environment Script

Now we'll use that libary to create a CLI for fetching keys.

<script src="https://gist.github.com/parabuzzle/fbfdc7dd1221a602195f7d7ce0589ad7.js"></script>

We would put this script in to a `bin` directory and make it executable. Then we could just call it like this: `vaultenv postgres/password` and it will return the value from `/secret/development/postgres/password`.

#### The Docker Entrypoint

We'll need a script that we'll set as the container's entrypoint that will load the environment and then pass off to a new shell with that environment

<script src="https://gist.github.com/parabuzzle/e525dc564d7fc8bf83d189ae919ae56b.js"></script>

This script has a little magic built in but essentially what its doing is verifying the `VAULT_TOKEN` is set and then sources a custom environment file and then passes off to whatever arguments are provided.

Example:

~~~ bash
with_app_env bash
~~~

This will load the custom environment and then run a bash shell inside of it.

We'll accomplish this as the default for the container by setting the `entrypoint` as this script and the `cmd` as bash.

~~~ Dockerfile
ENTRYPOINT [ "with_app_env" ]
CMD [ "bash" ]
~~~

#### The Custom Environment File

The final bit is the custom environment file. Inside of it we can use our Vault Environment script to load things:

~~~ bash
export PG_USER=$( vaultenv postgres/user )
export PG_PASS=$( vaultenv postgres/password )
~~~


### Wire it Together

Now we can wire it all together with a Dockerfile. Once we create this container, we would build all of our applications off of this container and like magic, our application containers now only need the 2 required variables (`VAULT_TOKEN` & `APP_ENV`) at runtime to set everything up!

<script src="https://gist.github.com/parabuzzle/1b880d9b4543d7a2308ea1471574f928.js"></script>

True bliss.

(For a complete working implementation of this solution, [Check out my example project](https://github.com/parabuzzle/docker_vault_autoloader))
