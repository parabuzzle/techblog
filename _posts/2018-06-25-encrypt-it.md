---
layout: post
title:  Encrypt It!
date:   2018-06-25 12:00:00
categories: Encryption
disqus_id: 20
---

Today I was presented with a common problem: I have a private key file I must distribute to my team members securely. How do I do it? My first thought is that I could email it to them all, but our email is retained for quite a long time and isn't a secure mechanism regardless. Then I thought I'd use a file share, but that seemed less ideal given that access wasn't controlled by me directly and who knows who could end up getting access one day. 

Then it hit me! Why not just encrypt it before I send it and they can decrypt it on the other end? That's exactly what I did. Now, there are many ways to encrypt something and some are better than others (asymmetric encryption is better than symmetric... look that up on your own). I decided that what would work best for me was AES-256 encryption because using a symmetrical encryption would allow me to use a password that I could verbally give out to decrypt it rather than dealing with collecting public keys for this particular case. Simple password protection was all that was needed here. So AES it was!

Here's how to quickly encrypt using AES encryption on the command line:

~~~
openssl aes-256-cbc -e -in secret.key -out secret.key.enc
~~~

This will prompt you for a password to use and on successfully typing the password twice it will write out the `secret.key.enc`.

Now you can attach the `secret.key.enc` file to your email and know that its (somewhat) protected.

All you have to do is share the password you used with the recipient(s) and they can decrypt it on their end with this command:

~~~
openssl aes-256-cbc -d -in secret.key.end -out secret.key
~~~ 

Boom! now they have the `secret.key` file and it was encrypted in transit!

Go forth and Encrypt!