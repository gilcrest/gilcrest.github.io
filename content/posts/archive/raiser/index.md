---
title: "Raiser v0.5 - Committed to Github"
date: 2015-08-17T12:39:00-04:00
description: PL/SQL Error Raiser
menu:
  sidebar:
    name: Raiser
    identifier: raiser
    parent: archive
    weight: 900
---

I just released an early version of my "raiser" code for Oracle PL/SQL, leveraging OraOpenSource's logger utility.  I'd love to get anyone's input and help with it.  The Github project is [https://github.com/gilcrest/raiser][raiser]

As background, I've used the Quest Error Manager (QEM) in my codebase for years, but have recently switched primarily to using logger for most things. There are a few things that QEM does that logger does not...  In a highly distributed environment, you really need to be able to raise an exception to a user with a particular error message and unique identifier.  If an exception is thrown in one of my applications at 4 in the morning, I need that user to be able to contact support with a unique ID and I need to be able to get a full trace of that error for proper debugging and resolution.

Close to a year ago, I reached out to Martin D'Souza to ask if I could work with him to enhance the logger product to be able to achieve this type of implementation.  Martin, being a fantastic product owner, pushed back saying he didn't want logger to be a "raiser", but was soon to release a new version of logger that would have hooks in it (in the form of a plugin) that I could use to achieve my goals...  Great!  So, he did what he said he would - logger 3.0.0 and above has a really cool plugin facility that I've utilized to make a "raiser" out of logger.

Also, after reading Dan McGhan's excellent [post][McGhan-post] about using the APEX_JSON package for creating JSON content, I got the idea to encapsulate multiple fields into the Oracle exception text using JSON...  I had been struggling to figure out a good way to be able to give end users multiple data points with just the one SQLERRM output, but I think this works out nicely.

I plan to add a lot more in the coming weeks to the code - for instance, an example of using the code within an APEX error handle function to return a unique logger identifier to end users.  Someday, I'd also like to eventually work through some clients (java, javascript, python, etc.).

The first order of business is to fix the current getErrorDetails function to parse the JSON SQLERRM and how to work with it...

[raiser]:  https://github.com/gilcrest/raiser
[McGhan-post]:   https://jsao.io/2015/07/relational-to-json-with-apex_json/
