---
layout: post
title:  SSL Cert Generation for Amazon ELB
date:   2015-01-31 12:00:00
categories: SSL
preview: After doing a security audit of our webapp, we discovered that our SSL certs and settings needed some updating. One of the most major things was that the certs where generated using SHA1 which is dangerously weak and becomes more so every day.
tags: [SSL, AWS, ELB, Amazon, Cloud, Security]
og_type: article
og_description: After doing a security audit of our webapp, we discovered that our SSL certs and settings needed some updating. One of the most major things was that the certs where generated using SHA1 which is dangerously weak and becomes more so every day.
disqus_id: 11
---

After doing a security audit of our webapp, we discovered that our SSL certs and settings needed some updating. One of the most major things was that the certs where generated using SHA1 which is dangerously weak and becomes more so every day. (more on that [here](https://konklone.com/post/why-google-is-hurrying-the-web-to-kill-sha-1)) So to fix this, we needed to re-issue our certificates using the more secure SHA2 algorithm.

You can check your site right now using [www.ssllabs.com](http://www.ssllabs.com/ssltest)... here's our score:

<img class="img-fluid" src="/img/ssllabs_b.png"/>

SSL cert generation can be a bit daunting for the uninitiated but if you use Linux or a Mac, I assure you we can get through it here if you follow these simple steps. :smile:

### Setup

We're going to use OpenSSL to do all of this. I'm doing this on my Mac but you can use Linux if you like. If you have a Mac with XCode installed it will have this already. (If you're using Linux, install the latest OpenSSL package using the appropriate package manager... `apt-get install openssl` or `yum install openssl`)

Verify you have the `openssl command`:


    [07:13:06]heijmans@HolyCrap:~$ which openssl
    /usr/bin/openssl
    [07:13:12]heijmans@HolyCrap:~$

## First we need a Key

You always need a key for a lock right? Otherwise it wouldn't be a lock now would it? The same is true for SSL. (there's a lock in the browser window with SSL..  I know, I know... terrible joke.)

We need to use OpenSSL to generate an RSA key with a length of 2048 bits like this:


    openssl genrsa -out your_website_name.key 2048

    # Change "your_website_name" to an appropriate name for your key


Whamo! you have a key:


    [07:17:01]heijmans@HolyCrap:~/demo$ openssl genrsa -out your_website_name.key 2048
    Generating RSA private key, 2048 bit long modulus
    ....................................+++
    ...+++
    e is 65537 (0x10001)


**DO NOT SHARE THIS KEY WITH ANYONE! IT IS YOUR IDENTITY FOR THE CERT WE GENERATE LATER!**

## Create the CSR using your key

The next step is a Certificate Signing Request (CSR). You will need the CSR to give to your SSL provider (in our case, GeoTrust) so that they can generate your certificate that references your private key. (Think of this as a special public key to that private key we just made that has information about your website/company that will be embedded in your certificate)

This command will ask you a series of questions to get information about the website that will be embedded in your key.

   * **Country Name** (2 letter code) - Your Country (ie: US)
   * **State or Province Name** (full name) - Your State (ie: California)
   * **Locality Name** (eg, city) - Your City (ie: San Francisco)
   * **Organization Name** - Your Company Name (ie: My Awesome Company)
   * **Organizational Unit Name** - Your department (ie: IT) (I usually leave this blank)
   * **Common Name** - The FQDN of your Website (ie: www.your-website.com) (**This is where the cert will be used!**)
   * **Email Address** - The email address for administrating your cert (ie: admin@your-website.com)
   * **A challenge password** - A password used to authenticate you by the Cert Issuer if the need arises (ie: revoking the key)
   * **An optional company name** - Used for identification with the password (I usually leave this blank)

**IMPORTANT! Don't lose the challenge password! You will need it if your key ever gets compromised!**

We do this using OpenSSL:


    openssl req -new -sha256 -key your_website_name.key -out your_website_name.csr

    # Again, change "your_website_name" to an appropriate name for your CSR


Here we used the key we generated in the first step to output a CSR

## Give your CSR to the Certificate Issuer

You will then upload or paste the contents of the CSR on your issuer's website specifying that you want SHA2. (if its even an option anymore).

**Important! This is the step that you specify that you want SHA2**

We use GeoTrust RapidSSL so we just have to paste the contents of the CSR in a textbox on their site.

It should look something like this:


    -----BEGIN CERTIFICATE REQUEST-----
    MIIDADCCAegCAQAwgZ0xCzAJBgNVBAYTAlVTMRMwEQYDVQQIEwpDYWxpZm9ybmlh
    MRYwFAYDVQQHEw1TYW4gRnJhbmNpc2NvMRswGQYDVQQKExJNeSBBd2Vzb21lIENv
    bXBhbnkxHTAbBgNVBAMTFHd3dy55b3VyLXdlYnNpdGUuY29tMSUwIwYJKoZIhvcN
    AQkBFhZhZG1pbkB5b3VyLXdlYnNpdGUuY29tMIIBIjANBgkqhkiG9w0BAQEFAAOC
    AQ8AMIIBCgKCAQEA2V0dHAlFyLI/XaAfqmtdyq2K+iv/wEJycVYVTZ2a1X0Y2i+8
    8dhXg3y1f79wCFpqRWdccjQG2HesKOxzvrnrPfCc93f7k2kuRDzivNacnOsMw19W
    bVEcXU38qxnNMtJOqyYzl5Yh1VBojyTvuec3g/21IxWQGPV7QsZ6OAMkw09V8H/U
    CCT2F+JkFPkP/QHt+xyrGhj+CVAfLGmYUjAwQIB/Kz9xAZdILOSEiS/lf+rPurzg
    XU3i6ZHMTICog9a5k05XBU2gzbh8otTG6/czRn2hAqUTLqeNBb98YXBLEj0D8gC/
    Nl124155fffaxcfZTMrQp1iJ9Z66JUx0kWxPXwIDAQABoB0wGwYJKoZIhvcNAQkH
    MQ4TDGFiYzEyMzMyMWNiYTANBgkqhkiG9w0BAQsFAAOCAQEAT1nR6EHMNOeASrog
    QFavjMK6bI4IM5Dc3m7kUS2V7dbpinH8No2MawvLxyyDDYIp7de2Lk4LzFN2Vmjj
    e7lr8yYC5ZoX6GbuvXEznnpcd7QWvoQxJ/ISWY2fgR8S9EFYiMaQ1oLh8aAWepHX
    AcFMUKc9HF8yWS/xIaVC8WaXBM9YnM5Ich9/KPAF9iQ3e/f0wltSmTpwL9rcYuA/
    kbaxBJRbzdYhN9JSAqhYidplcAQK4YUGtivkVoWxgoD9q0camegbF3AXuzrH1M6j
    xI2Za6PXlBy7TX1d0k9W1d924B3E5qxZiVQ7cBQes9EEF/bJVyKkyNsjF1LrU00G
    INsRUQ==
    -----END CERTIFICATE REQUEST-----


## Get the Certificate from the Issuer

Depending on your Certificate Authority/Issuer and the type of cert you request, this can take a few days or happen instantly.

You should receive 2 certs from your issuer. One is your actual Certificate and the other is what's known as the Intermediate Cert or bundle or root cert. You only need the first for SSL to work but you need both to have a proper chain setup. (A lot of people mess this up with ELB). SSL will work without the chain but to have a proper SSL secured website, you need to include the path back to the "RootCA" for your cert so that every certificate in the chain can be verified. That's what the bundle is for. It includes the public keys of the all the certs in your chain and can be provided with the cert negotiation request for an SSL session.

If you don't have your chain setup properly you will see this error from SSL Labs:

<img src="/img/chain.png" class="img-fluid"/>

*Note: if the issuer gives you the option, you will need the X.509 version of the certificates for the next steps*

## Generate PEM files for Amazon

So, here's where it gets fun... Your certs will have been issued as `.crt` files, but Amazon wants all the keys to be PEM encoded. For some reason, lots of people have trouble with this step, but you won't because you're using this guide. :smile:

We need to encode both certificates AND the private key from step 1:


    openssl rsa -in your_website_name.key -text > your_website_name.key.pem
    openssl x509 -inform PEM -in your_website_name.crt > your_website_name.crt.pem
    openssl x509 -inform PEM -in your_website_name_root.crt > your_website_name_root.crt.pem

    # replace the names... of course


Almost there.. now we just need to upload these to amazon's ELB


## Upload your keys into your ELB

Paste the contents of the 3 PEM files you just created into their respective text boxes:

<img class="img-fluid" src="/img/elb.png"/>

Click save and you're done!


## SAVE ALL THESE FILES IN A SAFE PLACE!

Make sure that you save all these files we created and the challenge password you used in a safe place. Chances are you will never need them again but if you do, its a real bummer to not have them. (I like to keep these in a folder labeled with the domain name in the company google drive)

You should have:

  * `your_website_name.crt`
  * `your_website_name.crt.pem`
  * `your_website_name_root.crt`
  * `your_website_name_root.crt.pem`
  * `your_website_name.csr`
  * `your_website_name.key`
  * `your_website_name.key.pem`



## Oh, and don't forget to check your awesomeness on ssllabs

<img src="/img/ssllab_yay.png" class="img-fluid"/>
