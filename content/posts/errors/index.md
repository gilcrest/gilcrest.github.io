---
title: "REST API Error Handling in Go"
date: 2021-06-03T09:06:25-04:00
description: An approach to REST API error handling in the Go programming language.
menu:
  sidebar:
    name: REST API Error Handling in Go
    weight: 1
---

Handling errors is really important in Go. Errors are first class citizens and there are many different approaches for handling them. Initially I started off basing my error handling almost entirely on a [blog post from Rob Pike](https://commandcenter.blogspot.com/2017/12/error-handling-in-upspin.html) and created a carve-out from his code to meet my needs. It served me well for a long time, but found over time I wanted a way to easily get a stacktrace of the error, which led me to Dave Cheney's [https://github.com/pkg/errors](https://github.com/pkg/errors) package. I now use a combination of the two. The implementation below is sourced from my [go-api-basic](https://github.com/gilcrest/go-api-basic) repo, indeed, this post will be folded into its [README](https://github.com/gilcrest/go-api-basic#errors) as well.

## Requirements

My requirements for errors from a REST API perspective are the following:

- Requests for users who are *not* properly authenticated should return a `401 Unauthorized` error with a `WWW-Authenticate` response header and an empty response body.
- Requests for users who are authenticated, but do not have permission to access the resource, should return a `403 Forbidden` error with an empty response body.
- All requests which are due to a client error (invalid data, malformed JSON, etc.) should return a `400 Bad Request` and a response body which looks similar to the following:

```json
{
    "error": {
        "kind": "input_validation_error",
        "param": "director",
        "message": "director is required"
    }
}
```

- All requests which incur errors as a result of an internal server or database error should return a `500 Internal Server Error` and not leak any information about the database or internal systems to the client. These errors should return a response body which looks like the following:

```json
{
    "error": {
        "kind": "internal_error",
        "message": "internal server error - please contact support"
    }
}
```

All errors should return a `Request-Id` response header with a unique request id that can be used for debugging to find the corresponding error in logs.

## Implementation

All errors should be raised using custom errors from the [errs](https://github.com/gilcrest/go-api-basic/tree/main/domain/errs) package. The three custom errors correspond directly to the requirements above. 

### Unauthenticated Errors

```go
type UnauthenticatedError struct {
    // WWWAuthenticateRealm is a description of the protected area.
    // If no realm is specified, "DefaultRealm" will be used as realm
    WWWAuthenticateRealm string

    // The underlying error that triggered this one, if any.
    Err error
}
```

The [spec](https://tools.ietf.org/html/rfc7235#section-3.1) for `401 Unauthorized` calls for a `WWW-Authenticate` response header along with a `realm`. The realm should be set when creating an Unauthenticated error. The `errs.Unauthenticated` function initializes an `UnauthenticatedError`.

```go
func Unauthenticated(realm string, err error) *UnauthenticatedError {
    return &UnauthenticatedError{WWWAuthenticateRealm: realm, Err: err}
}
```

### Unauthorized Errors

Error handling throughout `go-api-basic` always creates an error using the `E` function from the `errs` package as seen below. `errs.E`, is derived from Rob Pike's package (but has been changed a lot). The `errs.E` function call is [variadic](https://en.wikipedia.org/wiki/Variadic) and can take several different types to form the custom `Error`struct.

 ```go
// Error is the type that implements the error interface.
// It contains a number of fields, each of different type.
// An Error value may leave some values unset.
type Error struct {
    // User is the username of the user attempting the operation.
    User UserName
    // Kind is the class of error, such as permission failure,
    // or "Other" if its class is unknown or irrelevant.
    Kind Kind
    // Param represents the parameter related to the error.
    Param Parameter
    // Code is a human-readable, short representation of the error
    Code Code
    // The underlying error that triggered this one, if any.
    Err error
}
```

Here is a simple example of creating an `error` using `errs.E`:

```go
err := errs.E("seems we have an error here")
```

When a string is sent, an error will be created using the `errors.New` function from `github.com/pkg/errors` and added to the `Err` element of the struct, which allows a stacktrace to be generated later on if need be. In the above example, `User`, `Kind`, `Param` and `Code` would all remain unset.

You can set any of the custom error values that you like, for example:

```go
func (m *Movie) SetReleased(r string) (*Movie, error) {
    t, err := time.Parse(time.RFC3339, r)
    if err != nil {
        return nil, errs.E(errs.Validation,
            errs.Code("invalid_date_format"),
            errs.Parameter("release_date"),
            err)
    }
    m.Released = t
    return m, nil
}
```

Above, we used `errs.Validation` to set the `errs.Kind` as Validation. Valid error `Kind` are:

```go
// Kinds of errors.
//
// The values of the error kinds are common between both
// clients and servers. Do not reorder this list or remove
// any items since that will change their values.
// New items must be added only to the end.
const (
    Other           Kind = iota // Unclassified error. This value is not printed in the error message.
    Invalid                     // Invalid operation for this type of item.
    Permission                  // Permission denied.
    IO                          // External I/O error such as network failure.
    Exist                       // Item already exists.
    NotExist                    // Item does not exist.
    Private                     // Information withheld.
    Internal                    // Internal error or inconsistency.
    BrokenLink                  // Link target does not exist.
    Database                    // Error from database.
    Validation                  // Input validation error.
    Unanticipated               // Unanticipated error.
    InvalidRequest              // Invalid Request
    Unauthenticated             // User did not properly authenticate
    Unauthorized                // User is not authorized for the resource
)
```

`errs.Code` represents a short code to respond to the client with for error handling based on codes (if you choose to do this) and is any string you want to pass.

`errs.Parameter` represents the parameter that is being validated or has problems, etc.

In addition, instead of passing a string and creating a new error inside the `errs.E` function, I am just passing in the error received from the `time.Parse` function and inside `errs.E` the error is added to `Err` using `errors.WithStack` from the `github.com/pkg/errors` package so that the stacktrace can be obtained later if needed.

There are a few helpers in the `errs` package as well, namely the `errs.MissingField` function which can be used when validating missing input on a field. This idea comes from [this Mat Ryer post](https://medium.com/@matryer/patterns-for-decoding-and-validating-input-in-go-data-apis-152291ac7372) and is pretty handy.

Here is an example in practice:

```go
// IsValid performs validation of the struct
func (m *Movie) IsValid() error {
    switch {
    case m.Title == "":
        return errs.E(errs.Validation, errs.Parameter("title"), errs.MissingField("title"))
```

The error message for the above would read **title is required**

There is also `errs.InputUnwanted` which is meant to be used when a field is populated with a value when it is not supposed to be.

#### Error Flow

Errors at their initial point of failure should always start with `errs.E`, but as they move up the call stack, `errs.E` does not need to be used. Errors should just be passed on up, like the following:

```go
func inner() error {
    return errs.E("seems we have an error here")
}

func middle() error {
    err := inner()
    if err != nil {
        return err
    }
    return nil
}

func outer() error {
    err := middle()
    if err != nil {
        return err
    }
    return nil
}

```

In the above example, the error is created in the `inner` function - `middle` and `outer` return the error as is typical in Go.

At the top of the program flow for each service is the handler. In each handler, if any error occurs from any function/method calls, they are sent through the `errs.HTTPErrorResponse` function along with the `http.ResponseWriter` and a `zerolog.Logger`.

For example:

```go
// Call the NewMovie method for struct initialization
m, err := movie.NewMovie(uuid.New(), extlID, u)
if err != nil {
    errs.HTTPErrorResponse(w, logger, err)
    return
}
```

`errs.HTTPErrorResponse` takes the custom `Error` created by `errs.E` and writes the HTTP response body as JSON as well as logs the error. If `zerolog.ErrorStackMarshaler` is set to log error stacks (more about this later), the logger will log the full error stack, which can be super helpful when trying to identify issues. When the above error is returned to the client using the `errs.HTTPErrorResponse` function in each of the handlers, the response body JSON looks like the following:

```json
{
    "error": {
        "kind": "input_validation_error",
        "code": "invalid_date_format",
        "param": "release_date",
        "message": "parsing time \"1984a-03-02T00:00:00Z\" as \"2006-01-02T15:04:05Z07:00\": cannot parse \"a-03-02T00:00:00Z\" as \"-\""
    }
}
```

and the error log looks like (I cut off parts of the stack for brevity):

```json
{
    "level": "error",
    "ip": "127.0.0.1",
    "user_agent": "PostmanRuntime/7.26.8",
    "request_id": "bvol0mtnf4q269hl3ra0",
    "stack": [{
        "func": "E",
        "line": "172",
        "source": "errs.go"
    }, {
        "func": "(*Movie).SetReleased",
        "line": "76",
        "source": "movie.go"
    }, {
        "func": "(*MovieController).CreateMovie",
        "line": "139",
        "source": "create.go"
    }, {
    ...
    }],
    "error": "parsing time \"1984a-03-02T00:00:00Z\" as \"2006-01-02T15:04:05Z07:00\": cannot parse \"a-03-02T00:00:00Z\" as \"-\"",
    "HTTPStatusCode": 400,
    "Kind": "input_validation_error",
    "Parameter": "release_date",
    "Code": "invalid_date_format",
    "time": 1609650267,
    "severity": "ERROR",
    "message": "Response Error Sent"
}
```

> Note: `E` will usually be at the top of the stack as it is where the `errors.New` or `errors.WithStack` functions are being called. If you prefer not to see this, you can call `errors.New` or `errors.WithStack` as part of the `errs.E` call, for example:

```go
err := errs.E(errors.New("seems we have an error here"))
```

Illustration by [Pixeltrue](https://icons8.com/illustrations/author/5ec7b0e101d0360016f3d1b3) from [Ouch!](https://icons8.com/illustrations)
