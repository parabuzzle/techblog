---
layout: post
title:  Managing Chef with Codeship
date:   2014-09-19 12:00:00
categories: Chef
#preview: A simple step-by-step guide to setting up Codeship.io to test and deploy Chef changes to your Chef server automatically.
tags: [Tech, Chef, Startup, Tips, Tricks, Codeship]
og_type: article
og_description: A simple step-by-step guide to setting up Codeship.io to test and deploy Chef changes to your Chef server automatically.
og_image: http://www.mikeheijmans.com/img/postimgs/chef-logo.jpg
disqus_id: 9
---

I'm deeply passionate about Continuous Integration and Continuous Deployment practices. I don't consider a project to be "ready for production" until I have it building and deploying on its own.

*(this includes proper test coverage that can make sure the automation doesn't push bunk code out)*

At [Niche](http://www.niche.co), we use hosted Chef to manage our infrastructure and [Codeship.io](http://codeship.io) to build and deploy our application code. After manually managing Chef changes for the past month, I decided that the automation can't wait any longer. There are too many places to make mistakes or miss critical updates. Plus, I want to follow Github flow with the Github repo as the source of truth and not just for it to be a place we store what we pushed up to [manage.opscode.com](http://manage.opscode.com) manually.

This means we need to flip our process around and have automation push anything that's merged into master on Github and also make it so that the automation is responsible for updating the Chef server without any manual steps. **The goal here is to have our build system update chef when things are merged to master**

This guide is written for Codeship because we use Codeship but the fundamentals and architecture is the same whether you want to use Travis-CI, Jenkins, or something else.

## Setup Your Repo

There is some general setup that needs to happen. The first thing I did was created a `Rakefile` and `Gemfile` so that all the scripting on the build system was basically rake commands.


We use [ChefDk](https://downloads.getchef.com/chef-dk/), but since Codeship doesn't have ChefDk as an option for a build environment, we need to manage the needed libraries with a `Gemfile`. *(yes I know we could just install ChefDk in the setup process but its a bit bloated for what we need it for in the build environment) :tongue:*

### Setup a Gemfile

In my Gemfile I have all the needed gems for the build system:

~~~ ruby
source "https://rubygems.org"

gem "berkshelf", "~> 3.1.0"
gem "chef", "~> 11.12.8"
gem 'foodcritic', "~> 4.0.0"
gem 'knife-env-diff'
gem 'unf'
gem 'aescrypt'
~~~

This is so that we can setup our environment with `bundle install`

*We'll get to the `aescrypt` gem later and what it's used for.*

### Setup a Rakefile

The goal here is to have all the needed tasks for updating your chef server without using knife directly.

*(why you ask? because this allows us to put more business logic around different tasks.. ie: if the cookbook version is already on the server, delete it first then re-upload)*

In this example I'm just going to show you the Data Bag update piece and you can expand that to environments, roles, cookbooks, etc on your own.

we can create a rake task for uploading all Data Bags like this:

~~~ ruby
namespace :databag do

  task :check_secret do
    raise 'missing encryption key - ./.chef/data_bag_secret.pem' unless File.exists?('.chef/data_bag_secret.pem')
  end

  namespace :upload do
    desc "Upload all Data Bags"
    task :all => :check_secret do
      sh 'knife data bag from file -a'
    end
  end # namesapce - databags:upload
end # namespace - databags
~~~

You would run this like this: `rake databag:upload:all`

Since we use encrypted Data Bags, I have a check_secret task that I use to make sure the key file is there before attempting to upload.. If it wasn't there, we run the risk up uploading data to chef in the clear.

You probably also want tests in place so you should write a test task that does whatever testing you want. (I recommend [Test Kitchen](http://kitchen.ci)). For the purposes of this guide, we'll just hookup foodcritic, but you should do more than simple linting. :wink:

~~~ ruby
task :lint do
  sh 'foodcritic -f any ./cookbooks'
end
~~~

This task is run like this: `rake lint` and will fail the build if any foodcritic errors exist.

## Setup Your Chef User

You can decide to use your user or create a new one. In my case I created a deployment user on manage.opscode.com and pulled down that user's key (`deployuser.pem`). Either way you will need to get the key for the user you want to use on your build system. If you're running your own build system (like jenkins) you could just install the needed keys on your build slave and be done but with Codeship or any hosted service, that can be tricky.

I came up with a clever way to get them on the build instance that's self contained and secure. I encrypt the key files in place using the `aescrypt` gem (I told you we would get to that) and check in the AES encrypted version of the needed key files so that the build system can decrypt them at run time.

### Setup Encryption and Decryption

You need to add the libraries and the rake tasks to your `Rakefile`:

~~~ ruby
# there is a bug in aescrypt that requires
# we manually pull in base64
require 'base64'
require 'aescrypt'

namespace :keys do

  desc "encrypts the needed keys for saving in to source"
  task :encrypt do
    passphrase = ENV['KEYS_PASSPHRASE']
    raise "\nMissing ENV['KEYS_PASSPHRASE'] environment variable\n" unless passphrase
    files = ['.chef/deployuser.pem', '.chef/data_bag_secret.pem']
    files.each do |file|
      File.open("#{file}.enc", 'w') {|fh|
        puts "encrypting #{file} as #{file}.enc"
        fh.print(AESCrypt.encrypt(File.read(file), passphrase))
      }
    end
  end

  desc "decrypts the needed keys for pushing to the chef server"
  task :decrypt do
    passphrase = ENV['KEYS_PASSPHRASE']
    raise "\nMissing ENV['KEYS_PASSPHRASE'] environment variable\n" unless passphrase
    files = ['.chef/deployuser.pem', '.chef/data_bag_secret.pem']
    files.each do |file|
      File.open(file, 'w') {|fh|
        puts "decrypting #{file}.enc as #{file}"
        fh.puts(AESCrypt.decrypt(File.read("#{file}.enc"), passphrase))
      }
    end
  end
end #  namespace - keys

~~~

This provides the following rake tasks:

  * `rake keys:encrypt` to encrypt in place
  * `rake keys:decrypt` to decrypt in place

As you can see here, we're saving 2 keys: the user's key and the Data Bag key

All of the encryption is done with the passphrase provided by the environment variable `KEYS_PASSPHRASE`. This is what we set on Codeship to be able to decrypt the keys.

So to finish up the repo setup you just need to do the following on your command line:

~~~
export KEYS_PASSPHRASE=some_unique_passphrase
rake keys:encrypt
git add Rakefile Gemfile ./chef/*.enc
git commit -m'add required files for CI'
git push
~~~

(or something like that) :smile:


## Wire Up Codeship

With all this prep work done, Codeship will be a breeze.

First thing's first. Create a new project and connect to your repo. (duh)

### Setup the Test Tab:

Use the technology **"I want to create my own custom commands"**

In the box labeled **"Modify your Setup Commands"** you need these commands:

~~~
bundle install
bundle exec rake keys:decrypt
~~~

*note: wrap your commands in `bundle exec` to ensure you have the gems from the bundle install*

In the box labeled **"Modify your Test Commands"** you need your test commands:

~~~
bundle exec rake lint
~~~

So it should look like [this](https://www.evernote.com/shard/s6/sh/6f3bfb55-05b3-45eb-aae0-6749df206b3b/8533865904eb3d80fdd63f89d2881523/res/4cce1b3d-b855-46c0-a478-4fb5c288a475/skitch.png)

### Setup the Deployment Tab

On this tab you want to choose to use the **"Custom Script"**

And in there you want to run all the commands to upload to the chef server. In our case here, its just the databag upload task:

~~~
bundle exec rake databag:upload:all
~~~

It should look like [this](https://www.evernote.com/shard/s6/sh/c6e2e105-c8dd-4d48-a51f-00ac075d936c/228808a13cca66fa96d0e03ab64a39e6/res/c10d1978-840c-4a4b-919e-bae58d7a75e2/skitch.png)


### Setup the Environment Tab

We're almost done.. We just need to provide Codeship with the passphrase to decrypt the keys with.

So on the Environment Tab you need to put your ENV variables:

~~~
KEYS_PASSPHRASE=some_unique_passphrase
USER=deployuser
~~~

*note: the USER env variable is used in our `knife.rb` to set the user to use: `user = ENV["USER"]`*

It should look like [this](https://www.evernote.com/shard/s6/sh/c0d93cf0-40ff-4a80-bf4b-64b1dc865279/c3be32fce033fefdeabfa78f09516413/deep/0/Codeship---Hosted-continuous-integration-and-deployment.-Built-for-the-cloud..png)


**BOOM! YOU'RE DONE!**

Now anytime a pull request is opened, Codeship will lint the code for you and report that status on Github. And anytime someone merges into master, Codeship will push data bag changes directly to the chef server.

## Conclusion

This is just scratching the surface of how to get CI/CD working for you with chef. In this guide we have done the bare minimum to get things working. In practice you would want to expand your tests beyond simple foodcritic linting and you would want to expand your deployment to update everything (not just data bags), but I hope this serves as a guide for how to get started down the road of Continuous Deployment of your Chef Repo using Codeship.

*Bonus! :: your team can make changes to Chef without having to setup a user on the Chef server or understand how to use Knife!*
