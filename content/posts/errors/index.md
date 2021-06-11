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

My requirements for REST API error handling are the following:

- Requests for users who are *not* properly ***authenticated*** should return a `401 Unauthorized` error with a `WWW-Authenticate` response header and an empty response body.
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

All errors should be raised using custom errors from the [domain/errs](https://github.com/gilcrest/go-api-basic/tree/main/domain/errs) package. The three custom errors correspond directly to the requirements above.

### Typical Errors

Typically, errors raised throughout [go-api-basic](https://github.com/gilcrest/go-api-basic) are the custom `errs.Error`, which looks like:

 ```go
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

These errors are raised using the `E` function from the [domain/errs](https://github.com/gilcrest/go-api-basic/tree/main/domain/errs) package. `errs.E` is taken from Rob Pike's [upspin errors package](https://github.com/upspin/upspin/tree/master/errors) (but has been changed based on my requirements). The `errs.E` function call is [variadic](https://en.wikipedia.org/wiki/Variadic) and can take several different types to form the custom `errs.Error` struct.

Here is a simple example of creating an `error` using `errs.E`:

```go
err := errs.E("seems we have an error here")
```

When a string is sent, an error will be created using the `errors.New` function from `github.com/pkg/errors` and added to the `Err` element of the struct, which allows retrieval of the error stacktrace later on. In the above example, `User`, `Kind`, `Param` and `Code` would all remain unset.

You can set any of these custom `errs.Error` fields that you like, for example:

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

Above, we used `errs.Validation` to set the `errs.Kind` as `Validation`. Valid error `Kind` are:

```go
const (
    Other           Kind = iota // Unclassified error. This value is not printed in the error message.
    Invalid                     // Invalid operation for this type of item.
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
)
```

`errs.Code` represents a short code to respond to the client with for error handling based on codes (if you choose to do this) and is any string you want to pass.

`errs.Parameter` represents the parameter that is being validated or has problems, etc.

> Note in the above example, instead of passing a string and creating a new error inside the `errs.E` function, I am directly passing the error returned by the `time.Parse` function to `errs.E`. The error is then added to the `Err` field using `errors.WithStack` from the `github.com/pkg/errors` package. This will enable stacktrace retrieval later as well.

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

### Typical Error Flow

As errors created with `errs.E` move up the call stack, they can just be returned, like the following:

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

> In the above example, the error is created in the `inner` function - `middle` and `outer` return the error as is typical in Go.

You can add additional context fields (`errs.Code`, `errs.Parameter`, `errs.Kind`) as the error moves up the stack, however, I try to add as much context as possible at the point of error origin and only do this in rare cases.

### Handler Flow

At the top of the program flow for each service is the app service handler (for example, [DefaultMovieHandlers.CreateMovie](https://github.com/gilcrest/go-api-basic/blob/main/handler/movieHandler.go)). In the app service handler, any error returned from any function or method is sent through the `errs.HTTPErrorResponse` function along with the `http.ResponseWriter` and a `zerolog.Logger`.

For example:

```go
// Call the NewMovie method for struct initialization
m, err := movie.NewMovie(uuid.New(), extlID, u)
if err != nil {
    errs.HTTPErrorResponse(w, logger, err)
    return
}
```

`errs.HTTPErrorResponse` takes the custom error (`errs.Error`, `errs.Unauthenticated` or `errs.UnauthorizedError`), writes the response to the given `http.ResponseWriter` and logs the error using the given `zerolog.Logger`.

> `return` must be called immediately after `errs.HTTPErrorResponse` to return the error to the client.

#### Typical Error Response

For the `errs.Error` type, `errs.HTTPErrorResponse` writes the HTTP response body as JSON using the `errs.ErrResponse` struct.

```go
// ErrResponse is used as the Response Body
type ErrResponse struct {
    Error ServiceError `json:"error"`
}

// ServiceError has fields for Service errors. All fields with no data will
// be omitted
type ServiceError struct {
    Kind    string `json:"kind,omitempty"`
    Code    string `json:"code,omitempty"`
    Param   string `json:"param,omitempty"`
    Message string `json:"message,omitempty"`
}
```

When the error is returned to the client, the response body JSON looks like the following:

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

In addition, the error is logged. If `zerolog.ErrorStackMarshaler` is set to log error stacks (more about this in a later post), the logger will log the full error stack, which can be super helpful when trying to identify issues.

The error log will look like the following (*I cut off parts of the stack for brevity*):

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

> Note: `E` will usually be at the top of the stack as it is where the `errors.New` or `errors.WithStack` functions are being called.

#### Internal or Database Error Response

There is logic within `errs.HTTPErrorResponse` to return a different response body if the `errs.Kind` is `Internal` or `Database`. As per the requirements, we should not leak the error message or any internal stack, etc. when an internal or database error occurs. If an error comes through and is an `errs.Error` with either of these error `Kind` or is unknown error type in any way, the response will look like the following:

```json
{
    "error": {
        "kind": "internal_error",
        "message": "internal server error - please contact support"
    }
}
```

---

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

The [spec](https://tools.ietf.org/html/rfc7235#section-3.1) for `401 Unauthorized` calls for a `WWW-Authenticate` response header along with a `realm`. The realm should be set when creating an Unauthenticated error. The `errs.NewUnauthenticatedError` function initializes an `UnauthenticatedError`.

> I generally like to follow the Go idiom for brevity in all things as much as possible, but for `Unauthenticated` vs. `Unauthorized` errors, it's confusing enough as it is already, I don't take any shortcuts.

```go
func NewUnauthenticatedError(realm string, err error) *UnauthenticatedError {
    return &UnauthenticatedError{WWWAuthenticateRealm: realm, Err: err}
}
```

#### Unauthenticated Error Flow

The `errs.Unauthenticated` error should only be raised at points of authentication. I will get into application flow in detail in later posts, but authentication for [go-api-basic](https://github.com/gilcrest/go-api-basic) happens in middleware handlers prior to calling the app handler for the given route.

- The `WWW-Authenticate` *realm* is set to the request context using the `DefaultRealmHandler` middleware in the [handlers package](https://github.com/gilcrest/go-api-basic/blob/main/handler/middleware.go) prior to attempting authentication.
- Next, the Oauth2 access token is retrieved from the `Authorization` http header using the `AccessTokenHandler` middleware. There are several access token validations in this middleware, if any are not successful, the `errs.Unauthenticated` error is returned using the realm set to the request context.
- Finally, if the access token is successfully retrieved, it is then converted to a `User` via the `GoogleAccessTokenConverter.Convert` method in the `gateway/authgateway` package. This method sends an outbound request to Google using their API; if any errors are returned, an `errs.Unauthenticated` error is returned.

> In general, I do not like to use `context.Context`, however, it is used in [go-api-basic](https://github.com/gilcrest/go-api-basic) to pass values between middlewares. The `WWW-Authenticate` *realm*, the Oauth2 access token and the calling user after authentication, all of which are `request-scoped` values, are all set to the request `context.Context`.

#### Unauthenticated Error Response

Per requirements, [go-api-basic](https://github.com/gilcrest/go-api-basic) does not return a response body when returning an **Unauthenticated** error. The error response from [cURL](https://curl.se/) looks like the following:

```bash
HTTP/1.1 401 Unauthorized
Request-Id: c30hkvua0brkj8qhk3e0
Www-Authenticate: Bearer realm="go-api-basic"
Date: Wed, 09 Jun 2021 19:46:07 GMT
Content-Length: 0
```

---

### Unauthorized Errors

```go
type UnauthorizedError struct {
    // The underlying error that triggered this one, if any.
    Err error
}
```

The `errs.NewUnauthorizedError` function initializes an `UnauthorizedError`.

#### Unauthorized Error Flow

The `errs.Unauthorized` error is raised when there is a permission issue for a user when attempting to access a resource. Currently, [go-api-basic](https://github.com/gilcrest/go-api-basic)'s placeholder authorization implementation `DefaultAuthorizer.Authorize` in the [domain/auth](https://github.com/gilcrest/go-api-basic/blob/main/domain/auth/auth.go) package performs rudimentary checks that a user has access to a resource. If the user does not have access, the `errs.Unauthorized` error is returned.

Per requirements, [go-api-basic](https://github.com/gilcrest/go-api-basic) does not return a response body when returning an **Unauthorized** error. The error response from [cURL](https://curl.se/) looks like the following:

```bash
HTTP/1.1 403 Forbidden
Request-Id: c30hp2ma0brkj8qhk3f0
Date: Wed, 09 Jun 2021 19:54:50 GMT
Content-Length: 0
```

---
404 Error Illustration by [Pixeltrue](https://icons8.com/illustrations/author/5ec7b0e101d0360016f3d1b3) from [Ouch!](https://icons8.com/illustrations)
