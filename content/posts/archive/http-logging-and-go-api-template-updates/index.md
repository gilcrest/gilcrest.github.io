---
title: "HTTP Logging and Go API Template Updates"
date: 2018-10-13T07:10:00-04:00
description: HTTP Logging and Go API Template Updates
menu:
  sidebar:
    name: HTTP Logging and Go API Template Updates
    identifier: http-logging-and-go-api-template-updates
    parent: archive
    weight: 100
---

I built my first Go library! [httplog](https://github.com/gilcrest/httplog) provides middleware which logs http requests and responses along with a few other features I find useful. I thought it might be helpful for some. At the very least, I tried to document the library reasonably well, spending a lot of time developing the [README](https://github.com/gilcrest/httplog/blob/master/README.md) and ensuring the [GoDoc](https://godoc.org/github.com/gilcrest/httplog) was in good shape. I learned a lot through this exercise.

I also restructured my [Go API template repository](https://github.com/gilcrest/go-API-template) to try and shape it based on [Mat Ryer](https://medium.com/u/f25c357b8e4c?source=post_page-----3afba5c9d3a6--------------------------------)’s fantastic “[How I write Go HTTP services after seven years](https://medium.com/statuscode/how-i-write-go-http-services-after-seven-years-37c208122831)” post. I *think/hope* that I got the fundamentals right from the lessons he was trying to impart and have baked them into this template repo.

At a high level, using the template and httplog library, you can send a request that looks like this:

![request](/posts/archive/http-logging-and-go-api-template-updates/images/oldrequest.png)

and get a response body that looks like this:

![request](/posts/archive/http-logging-and-go-api-template-updates/images/oldresponse.png)

If turned on, the request and/or response are logged either to stdout or a PostgreSQL database (or both) and can then be used for investigative purposes, etc. You can also set options whether or not to log request/response headers and body. For instance, this example service takes password in the body, you should not log this and would turn off request body logging for this API.

As I mentioned above, I found the post from [Mat Ryer](https://medium.com/u/f25c357b8e4c?source=post_page-----3afba5c9d3a6--------------------------------) to be extremely compelling and completely restructured my [template](https://github.com/gilcrest/go-API-template). I now have a server struct, a la:

![request](/posts/archive/http-logging-and-go-api-template-updates/images/struct.png)

I have a [routes.go](https://github.com/gilcrest/go-API-template/blob/master/app/routes.go) file where all my routing lives…

![request](/posts/archive/http-logging-and-go-api-template-updates/images/routes.png)

and [my handlers hang off the server](https://github.com/gilcrest/go-API-template/blob/master/app/handleUser.go) with request and response structs defined within the handler…

![request](/posts/archive/http-logging-and-go-api-template-updates/images/handler.png)

I haven’t incorporated everything (still need to get to the testing stuff, etc.), but it’s a good start.

There’s a lot more happening in the [Go API template](https://github.com/gilcrest/go-API-template) (password hashing, graceful error handling, etc.) and I plan to bake even more into it as I continue to learn. For instance, this week’s release of Go Cloud’s [Wire](https://blog.golang.org/wire) makes me want to revisit my template again and add in dependency injection. I just added it to my list, actually….

Finally, I’ve been coding Go in a vacuum for a year as I don’t actually know anyone else who writes Go. I know it’s cliche, but I really do welcome feedback to help me learn. Thanks!
