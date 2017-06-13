---
layout: post
title:  "Name:Value Encryption Utility using PL/SQL"
date:   2014-12-17 01:05:00 -0400
categories: plsql
---
I recently ran into an issue where, as part of a webservice call I was making using the Oracle APEX_WEB_SERVICE API, I needed to pass the application id and password in the soap header, but I did not want those sensitive data elements to be exposed in my pl/sql code in cleartext.  I decided I wanted to encrypt the data and retrieve it at run time as a function call.  I tried to find a way to do this via Stack Overflow, Google, etc. but was unable to find a very simple way to do this.... 

I found many great posts on how to do encryption, but I was looking for a dead easy way to encrypt and persist random pieces of data and be able to retrieve it when necessary just as easily...  I decided to build the solution myself!  I have put a project up on GitHub, that I named Name:Value Encryption or NVE.  You can find the code on Github at [https://github.com/gilcrest/nve][nve-gh]

This is definitely v1 of the code and I'd love for the community to participate and give me pull requests to improve this!  My first mistake may be calling this Name:Value instead of Key:Value - not sure which is more accepted at this point...

[nve-gh]:   https://github.com/gilcrest/nve