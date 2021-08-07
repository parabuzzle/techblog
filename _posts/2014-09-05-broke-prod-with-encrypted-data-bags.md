---
layout: post
title:  "How I Broke Prod with Encrypted Data Bags"
date:   2014-09-05 12:00:00
categories: Chef
preview: Chef has so many ways of working with data, some people call it "getting Chefed in the face". Well, I got "Chefed" in the ass last week with encrypted data bags
tags: [Tech, Chef, Startup, Failures, Fixes, Tips, Tricks]
og_type: article
og_description: Chef has so many ways of working with data, some people call it getting Chefed in the face. Well, I got Chefed in the ass last week with encrypted data bags.
disqus_id: 8
---

Chef has so many ways of working with data, some people call it *"getting Chefed in the face"*. Well, I got *"Chefed"* in the ass last week with encrypted data bags. Here's the story and the solution so that you don't have to lose as much hair as I did over the whole thing.


## Data Bag Basics

What is a Chef Data Bag? Its one (of many) ways to handle data specific to a node or set of nodes. In its simplist form its a JSON object or a collection of JSON objects.

This is an example of a Data Bag (for my user in production):

~~~ json
{
  "id": "mikeheijmans",
  "groups": [
    "awesome"
  ],
  "shell": "/bin/bash",
  "comment": "Mike Heijmans",
  "ssh_keys": [
    "1231231313.key1",
    "2213112413.key2"
  ]
}
~~~

The only required field here is `id` and is used to identify the Data Bag inside the code and on the chef server.

Everything else in the data bag can be accessed and used within your chef recipes and is most commonly used for setting variables in file templates or setting attributes for a node.

## Encrypted Data Bags

An encrypted data bag is exactly what it sounds like. Instead of the json being in plain text, you can encrypt the data on the chef server and decrypt it inside your recipes using the `secret-file` (usually a .pem) or `secret` (passphrase).

This is what a similar data bag looks like when its encrypted (similar because I don't want you trying to figure out the key you nefarious little hacker :smile:):

~~~ json
{
  "id": "my_encrypted_dbag",
  "data_item1": {
    "encrypted_data": "bV/6HaMMw0lX85Bmxpci6wtTgoIgufL6mwTp\nmdERiv8EcIh40+k5VPMNkSoeeQ==\n",
    "iv": "kffXFaiYhWxQh1npHsbYw==\n",
    "version": 1,
    "cipher": "aes-256-cbc"
  },
  "data_item2": {
    "encrypted_data": "BkifX++NPJZiRnEU76FYOYGy1HS4J0Wb9EXU36xAUaAAs0QgR1f3w9Iv0e\nPsrrSyQotTgk2niMNyHdfDguTqFlVsefm1W2tA9DV/BrI17T+xjPnN50M+xv\nu7i5o5Ik\n",
    "iv": "Ecjf4Y6KZVEFOGDTHJDw==\n",
    "version": 1,
    "cipher": "aes-256-cbc"
  }
}
~~~

This is decrypted in the recipes like this:

~~~ ruby
secret = Chef::EncryptedDataBagItem.load_secret(Chef::Config[:encrypted_data_bag_secret])
my_encrypted_dbag = Chef::EncryptedDataBagItem.load("my_data_bag", "my_encrypted_dbag", secret)
~~~

You can encrypt a Data Bag on upload by running:

~~~
knife data bag from file my_data_bag my_encrypted_dbag.json --secret-file .chef/my_secret.pem
~~~

If you save the encrypted version in the JSON file in your source... you would upload like this:

~~~
knife data bag from file my_data_bag my_encrypted_dbag.json
~~~

*note: this doesn't have the secret-file because its encrypted locally..*

## How To Break Chef in Production with Encrypted Data Bags

We have a setting in our our `.chef/knife.rb` file like this:

~~~ ruby
knife[:secret_file] = "#{current_dir}/my_secret.pem"
~~~

What this little one-line setting does is cause all data bag commands with knife to automatically have the <br/>
`--secret-file ./chef/my_secret.pem` appended.

At my previous job, we would use the repo as the source of truth and always upload all when making a change:

~~~
knife data bag from file -a
~~~

This uploads all the Data Bags in source.

... so do you see the problem here?

The Data Bag files in source where saved in the encrypted format... when I ran the "upload all" command, it encrypted the encrypted Data Bag and uploaded it. Double encrytped! dun Dun DUN! (yo dawg, I heard you like encryption..)

So now I have a few problems...

  1. chef-client runs are failing and destroying, who knows what, files
  1. I can't figure out how to get chef/knife to reverse the mess

*good news... chef failed fast and didn't break anything except itself*

## How To Reverse The Mess

When I run:

~~~
knife data bag show my_data_bag my_encrypted_dbag --secret-file ./chef/my_secret.pem
~~~

 I get:

~~~ json
{
  "id": "my_encrypted_dbag",
  "data_item1": {
    "encrypted_data": "bV/6HaMMw0lX85Bmxpci6wtTgoIgufL6mwTp\nmdERiv8EcIh40+k5VPMNkSoeeQ==\n",
    "iv": "kffXFaiYhWxQh1npHsbYw==\n",
    "version": 1,
    "cipher": "aes-256-cbc"
---snip---
~~~

This is exactly what the client is getting back when it decrypts the Data Bag and why everything is breaking.

The question here is: How do I decrypt the data and then upload the returned (single-encrypted) version so Chef works?

Here is what I did:

First, I commented out the auto encryption line from `.chef/knife.rb`:

~~~ ruby
# knife[:secret_file] = "#{current_dir}/my_secret.pem"
~~~

*this will prevent any confusion with what's getting encrypted and what's not*

Then I simply re-upload the single encrypted version in the Repo (remember the json is already saved encrypted):

~~~
knife data bag from file -a
~~~

This will overwrite the data on the Chef server with the single-encrypted version and fix everything. (and it did)

## Prevention

After doing a ton of research and even talking to a Chef consultant about this, I have come up with a few solutions to prevent this from happening in the future. (in order of my liking)

 1. Don't save the encrypted version in source (this is only an option if your repo is private of course)
 1. Don't automatically encrypt Data Bags on upload (requires more manual steps)
 1. Use a wrapper for all this encryption stuff, like [Chef Vault](https://docs.getchef.com/chef_vault.html)
 1. Don't use Data Bags

I am a fan of number 1 because it provides a simple way to maintain Data Bags, allows for CI/CD to handle your Data Bag changes, and it makes Data Bag changes review-able by your team, but every case is different and you need to find your own way through the weeds here.

I hope this helps you learn through my mistakes and prevents a bit of headache and hair loss in the future.


