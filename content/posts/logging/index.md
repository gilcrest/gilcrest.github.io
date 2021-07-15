---
title: "Logging in go-api-basic"
date: 2021-07-13T07:30:00-04:00
description: Logging in go-api-basic
menu:
  sidebar:
    name: Logging in go-api-basic
    weight: 1
---

## Thoughts on Logging

I've gone through a few logging phases in my career. I logged *everything*. It was great, until the logs became meaningless because there were too many. I logged only errors, but in some cases that was not enough. Now I'm somewhere in the middle. I do believe in logging all errors and detailed how I do that in [my last post](https://dangillis.dev/posts/errors/#typical-error-response). There are also times when it's helpful to have some debugging logs in place. As an example, if you've ever tried to build a google doc using their APIs, then you may know that it can be helpful to have some tidbits of information about the document index position along your code/document path. In cases like this, I will leave debug logs in place in order to be able to pull the thread on them later if need be. Often times, I may deploy a change with some logs in place until I've seen enough data in action in production and then back out the logs in the next change when I'm satisfied. In general, I try to minimize the amount of logging in my code and only put logs in where it counts.

## Logging in go-api-basic with zerolog

[go-api-basic](https://github.com/gilcrest/go-api-basic) uses the [zerolog](https://github.com/rs/zerolog) library from [Olivier Poitrey](https://github.com/rs). I'm not gonna lie - initially, I chose this library because Olivier uses it at [Netflix](https://github.com/Netflix?language=go) and that was enough for me. Over time though, I've really come to love this logging library and Olivier is a great maintainer. I spent a lot of time a couple of years ago helping with the README and a number of other things in the library along the way. I've learned a lot about Go and open source just from these interactions. If I use a library, I generally try to give back in some way to it, but often times through these interactions, I get back more in return, actually.

The mechanics for using `zerolog` are straightforward and are well documented in the library's [README](https://github.com/rs/zerolog#readme). `zerolog` takes an `io.Writer` as input to create a new logger; for simplicity in `go-api-basic`, I use `os.Stdout`.

## Setting Logger State on Startup

When starting `go-api-basic`, there are several flags which setup the logger:

| Flag Name       | Description | Environment Variable | Default |
| --------------- | ----------- | -------------------- | ------- |
| log-level       | zerolog logging level (debug, info, etc.) | LOG_LEVEL | debug |
| log-level-min   | sets the minimum accepted logging level | LOG_LEVEL_MIN | debug |
| log-error-stack | If true, log full error stacktrace, else just log error | LOG_ERROR_STACK | false |

---

> `go-api-basic` uses the [ff](https://github.com/peterbourgon/ff) library from [Peter Bourgon](https://peter.bourgon.org), which allows for using either flags or environment variables. Click [here](https://github.com/gilcrest/go-api-basic#command-line-flags) if you want more info on this setup. Going forward, we'll assume you've chosen flags.

The `log-level` flag sets the Global logging level for your `zerolog.Logger`.

**zerolog** allows for logging at the following levels (from highest to lowest):

* panic (`zerolog.PanicLevel`, 5)
* fatal (`zerolog.FatalLevel`, 4)
* error (`zerolog.ErrorLevel`, 3)
* warn (`zerolog.WarnLevel`, 2)
* info (`zerolog.InfoLevel`, 1)
* debug (`zerolog.DebugLevel`, 0)
* trace (`zerolog.TraceLevel`, -1)

The `log-level-min` flag sets the minimum accepted logging level, which means, for example, if you set the minimum level to error, the only logs that will be sent to your chosen output will be those that are greater than or equal to error (`error`, `fatal` and `panic`).

The `log-error-stack` boolean flag tells whether to log stack traces for each error. If `true`, the `zerolog.ErrorStackMarshaler` will be set to `pkgerrors.MarshalStack` which means, for errors raised using the [github.com/pkg/errors](https://github.com/pkg/errors) package, the error stack trace will be captured and printed along with the log. All errors raised in `go-api-basic` are raised using `github.com/pkg/errors`.

## Reading and Modifying Logger State

You can retrieve and update the state of these flags using the `{{base_url}}/api/v1/logger` endpoint.

To retrieve the current logger state use a `GET` request:

```bash
curl --location --request GET 'http://127.0.0.1:8080/api/v1/logger' \
--header 'Authorization: Bearer <REPLACE WITH ACCESS TOKEN>'
```

and the response will look something like:

```json
{
    "logger_minimum_level": "debug",
    "global_log_level": "error",
    "log_error_stack": false
}
```

In order to update the logger state use a `PUT` request:

```bash
curl --location --request PUT 'http://127.0.0.1:8080/api/v1/logger' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer <REPLACE WITH ACCESS TOKEN>' \
--data-raw '{
    "global_log_level": "debug",
    "log_error_stack": "true"
}'
```

and the response will look something like:

```json
{
    "logger_minimum_level": "debug",
    "global_log_level": "debug",
    "log_error_stack": true
}
```

The `PUT` response is the same as the `GET` response, but with updated values. In the examples above, I used a scenario where the logger state started with the global logging level (`global_log_level`) at error and error stack tracing (`log_error_stack`) set to false. The `PUT` request then updates the logger state, setting the global logging level to `debug` and the error stack tracing. You might do something like this if you are debugging an issue and need to see debug logs or error stacks to help with that.

## Logger Setup in Handlers

After `zerolog.Logger` is initialized in `main.go`, it is injected into a `handler.Middleware` struct.

```go
// Middleware are the collection of app middleware handlers
type Middleware struct {
    Logger               zerolog.Logger
    AccessTokenConverter auth.AccessTokenConverter
    Authorizer           auth.Authorizer
}
```

 The `handler.Middleware` struct is then injected into the `handler.Routes` function (along with the app handlers and gorilla Mux router), which is responsible for registering routes and corresponding middleware/handlers to the given gorilla/mux router.

For each route registered to the handler, upon execution, the `zerolog.Logger` is then added to the request context through the `Middleware.LoggerChain` method for subsequent use with pre-populated fields, including the request method, url, status, size, duration, remote IP, user agent, referer. A unique `Request ID` is also added to the logger, context and response headers.

```go
func (mw Middleware) LoggerChain() alice.Chain {
    ac := alice.New(hlog.NewHandler(mw.Logger),
        hlog.AccessHandler(func(r *http.Request, status, size int, duration time.Duration) {
            hlog.FromRequest(r).Info().
                Str("method", r.Method).
                Stringer("url", r.URL).
                Int("status", status).
                Int("size", size).
                Dur("duration", duration).
                Msg("request logged")
        }),
        hlog.RemoteAddrHandler("remote_ip"),
        hlog.UserAgentHandler("user_agent"),
        hlog.RefererHandler("referer"),
        hlog.RequestIDHandler("request_id", "Request-Id"),
    )

    return ac
}
```

For every request, you'll get a request log that looks something like the following:

```json
{
    "level": "info",
    "remote_ip": "127.0.0.1",
    "user_agent": "PostmanRuntime/7.28.0",
    "request_id": "c3npn8ea0brt0m3scvq0",
    "method": "POST",
    "url": "/api/v1/movies",
    "status": 401,
    "size": 0,
    "duration": 392.254496,
    "time": 1626315682,
    "severity": "INFO",
    "message": "request logged"
}
```

All error logs will have the same request metadata, including `request_id`. The `Request-Id` is also sent back as part of the error response as a response header, allowing you to link the two. An error log will look somethiing like the following:

```json
{
    "level": "error",
    "remote_ip": "127.0.0.1",
    "user_agent": "PostmanRuntime/7.28.0",
    "request_id": "c3nppj6a0brt1dho9e2g",
    "error": "googleapi: Error 401: Request is missing required authentication credential. Expected OAuth 2 access token, login cookie or other valid authentication credential. See https://developers.google.com/identity/sign-in/web/devconsole-project., unauthorized",
    "http_statuscode": 401,
    "realm": "go-api-basic",
    "time": 1626315981,
    "severity": "ERROR",
    "message": "Unauthenticated Request"
}
```

> The above error log demonstrates a log for an error with stack trace turned off.

If the logger is to be used beyond the scope of the handler, I always pull the logger from the request context in the handler and send it as a parameter to any inner calls. The Logger is added only to the request context to capture request related fields with the Logger and be able to pass the initialized logger and middleware handlers easier to the app/route handler. Additional use of the logger should be directly called out in function/method signatures so there are no surprises. All logs from the logger passed down get the benefit of the request metadata though, which is great IMHO!

---

{{< tweet 1411167609983229954 >}}

[Peter Bourgon](https://peter.bourgon.org) recently tweeted the above about `github.com/pkg/errors`. He's not wrong. In general the standard library is always the way to go. For years I only used the standard library for raising errors and would annotate each error with the function name in order to capture a pseudo stack trace exactly like Rob Pike does in [this post](https://commandcenter.blogspot.com/2017/12/error-handling-in-upspin.html) (the `errs.Op` field). I believe in annotating and logging every error - not everyone does. It can cause noise in your logs, but in my experience, it's helpful. After years of trying to annotate every error though, I found I made too many mistakes. I kept forgetting to log the function name or I would misspell it, etc. I switched to use `github.com/pkg/errors` because of it's stack trace capability as well as the integration `zerolog` has with it. Stack trace for errors can certainly be overkill though, hence the ability to turn it on/off with a service for tactical usage can come in handy.

![My log does not judge.](log.gif)

> No actual logs were harmed in this post.
