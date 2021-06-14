---
title: "HTTP JSON Error Responses in Go"
date: 2018-03-18T07:10:00-04:00
description: HTTP JSON Error Responses in Go
menu:
  sidebar:
    name: HTTP JSON Error Responses in Go
    identifier: http-json-error-responses
    parent: archive
    weight: 400
---

I like simple structured messages using JSON in error responses, similar to [Stripe](https://stripe.com/docs/api#errors), [Uber](https://developer.uber.com/docs/riders/guides/errors) and many others…

```json
{
    "error": {
        "type": "validation_failed",
        "message": "Username is a required field"
    }
}
```

In my last story, I wrote about HTTP Logging and in that I mentioned that I have used “chained” middleware using the Adapter pattern from [Mat Ryer](https://medium.com/u/f25c357b8e4c?source=post_page-----29fac10a7036--------------------------------)’s excellent [post](https://medium.com/@matryer/writing-middleware-in-golang-and-how-go-makes-it-so-much-fun-4375c1246e81). You’ll see that below, but also in addition, I’m wrapping my final true app handler (in this case, CreateUser) inside an ErrHandler type — eh.ErrHandler{Env: env, H: handler.CreateUser} (you’ll note I’m passing in a global environment type as well). I’m doing this based on another great article by [Matt Silverlock](https://github.com/elithrar) on his blog [here](http://blog.questionable.services/article/http-handler-error-handling-revisited/).

```go
package dispatch

import (
    "github.com/gilcrest/go-API-template/appUser/handler"
    "github.com/gilcrest/go-API-template/env"
    eh "github.com/gilcrest/go-API-template/server/errorHandler"
    "github.com/gilcrest/go-API-template/server/middleware"
    "github.com/gorilla/mux"
)

// Dispatch is a way of organizing routing to handlers (versioning as well)
func Dispatch(env *env.Env, rtr *mux.Router) *mux.Router {

    // initialize new instance of APIAudit
    audit := new(middleware.APIAudit)

    // match only POST requests on /api/appUser/create
    // This is the original (v1) version for the API and the response for this
    // will never change with versioning in order to maintain a stable contract
    rtr.Handle("/appUser", middleware.Adapt(eh.ErrHandler{Env: env, H: handler.CreateUser}, middleware.LogRequest(env, audit), middleware.LogResponse(env, audit))).
        Methods("POST").
        Headers("Content-Type", "application/json")

    // match only POST requests on /api/v1/appUser/create
    rtr.Handle("/v1/appUser", middleware.Adapt(eh.ErrHandler{Env: env, H: handler.CreateUser}, middleware.LogRequest(env, audit), middleware.LogResponse(env, audit))).
        Methods("POST").
        Headers("Content-Type", "application/json")

    return rtr
}
```

I’m using Matt’s article almost word for word for error handling, but made a few tweaks so that I could return a structured JSON response. The package is below:

```go
package errorHandler

import (
    "encoding/json"
    "errors"
    "net/http"

    "github.com/gilcrest/go-API-template/env"
)

// Error represents a handler error. It provides methods for a HTTP status
// code and embeds the built-in error interface.
type Error interface {
    error
    Status() int
    ErrType() string
}

// HTTPErr represents an error with an associated HTTP status code.
type HTTPErr struct {
    Code int
    Type string
    Err  error
}

// Allows HTTPErr to satisfy the error interface.
func (hse HTTPErr) Error() string {
    return hse.Err.Error()
}

// SetErr creates an error type and adds it to the struct
func (hse *HTTPErr) SetErr(s string) {
    hse.Err = errors.New(s)
}

// ErrType returns a string error type/code
func (hse HTTPErr) ErrType() string {
    return hse.Type
}

// Status Returns an HTTP status code.
func (hse HTTPErr) Status() int {
    return hse.Code
}

type errResponse struct {
    Error svcError `json:"error"`
}

type svcError struct {
    Type    string `json:"type"`
    Message string `json:"message"`
}

// The ErrHandler struct that takes a configured Env and a function matching
// our useful signature.
type ErrHandler struct {
    Env *env.Env
    H   func(e *env.Env, w http.ResponseWriter, r *http.Request) error
}

// ServeHTTP allows Handler type to satisfy the http.Handler interface
func (h ErrHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {

    // Get a new logger instance
    log := h.Env.Logger

    log.Debug().Msg("Start Handler.ServeHTTP")
    defer log.Debug().Msg("Finish Handler.ServeHTTP")

    err := h.H(h.Env, w, r)

    if err != nil {
        // We perform a "type switch" https://tour.golang.org/methods/16
        // to determine the interface value type
        switch e := err.(type) {
        // If the interface value is of type Error (not a typical error, but
        // the Error interface defined above), then
        case Error:
            // We can retrieve the status here and write out a specific
            // HTTP status code.
            log.Printf("HTTP %d - %s", e.Status(), e)

            er := errResponse{
                Error: svcError{
                    Type:    e.ErrType(),
                    Message: e.Error(),
                },
            }

            // Marshal errResponse struct to JSON for the response body
            errJSON, _ := json.MarshalIndent(er, "", "    ")

            http.Error(w, string(errJSON), e.Status())

        default:
            // Any error types we don't specifically look out for default
            // to serving a HTTP 500
            cd := http.StatusInternalServerError
            er := errResponse{
                Error: svcError{
                    Type:    "unknown_error",
                    Message: "Unexpected error - contact support",
                },
            }

            log.Error().Msgf("Unknown Error - HTTP %d - %s", cd, err.Error())

            // Marshal errResponse struct to JSON for the response body
            errJSON, _ := json.MarshalIndent(er, "", "    ")

            http.Error(w, string(errJSON), cd)
        }
    }

}
```

I renamed the StatusError struct to HTTPErr and added Type (string) as a field to it. I then added an ErrType() method to the Error interface and to the HTTPErr struct in order to be able to extract the Type field. I added a couple of structs to help form the JSON (errResponse and svcError) in the manner I wanted and then populate and marshal those structs into JSON within the type switch logic. That’s it, really — Matt did all the heavy lifting, I just added some bits that I thought were helpful for my purposes.

This whole thing makes error handling pretty nice though — given the wrapper logic, you’ll always return a pretty good looking error. When creating errors within your app, you don’t have to have every error take this form — you can return normal errors lower down in the code and, depending on how you organize your code, you can catch and form the HTTPErr at a very high level so you’re not having to deal with populating a cumbersome struct all the way throughout your code. The below basically assumes that the usr.Create method is doing validations and as such we can add the “validation_failed” type to all errors returned from this method.

```go
tx, err := usr.Create(ctx, env)
if err != nil {
    return errorHandler.HTTPErr{
        Code: http.StatusBadRequest,
        Type: "validation_failed",
        Err:  err,
    }
}
```

I also added a SetErr method to the HTTPErr struct, which allows you to initialize the struct with some default values and add the actual error on the fly. For instance, below as part of my super high level edit checks on my service inputs, I initialize the HTTPErr object at the beginning of my function and then perform my edit checks to allow for brevity in error creation.

```go
func newUser(ctx context.Context, env *env.Env, cur *createUserRequest) (*appUser.User, error) {
// declare a new instance of appUser.User
usr := new(appUser.User)
// initialize an errorHandler with the default Code and Type for
// service validations (Err is set to nil as it will be set later)
e := errorHandler.HTTPErr{
    Code: http.StatusBadRequest,
    Type: "validation_error",
    Err:  nil,
}
// for each field you can go through whatever validations you wish
// and use the SetErr method of the HTTPErr struct to add the proper
// error text
switch {
// Username is required
case cur.Username == "":
    e.SetErr("Username is a required field")
    return nil, e
// Username cannot be blah...
case cur.Username == "blah":
    e.SetErr("Username cannot be blah")
    return nil, e
// If we get through the switch, set the field
default:
    usr.Username = cur.Username
}
.....
```

That’s it for now — I should note that Rob Pike and Andrew Gerrand authored a great [post](https://commandcenter.blogspot.com/2017/12/error-handling-in-upspin.html) on error handling that is likely amazing. I need to ingest that and see how I can fold that into this as well…
