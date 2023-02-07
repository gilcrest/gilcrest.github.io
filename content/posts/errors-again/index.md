---
title: "REST API Error Handling in Go - Part 2"
date: 2023-02-06T10:35:00-04:00
description: An updated approach to REST API error handling in the Go programming language.
menu:
  sidebar:
    name: REST API Error Handling in Go - Part 2
    weight: 19
---

I have recently updated the way I handle errors in Go. In some ways, I've come full circle back to where I started, in others, I think I've evolved. When I initially wrote my errors post in June of 2021, I had come up with a hybrid of Rob Pike's [upspin errors](https://commandcenter.blogspot.com/2017/12/error-handling-in-upspin.html) and Dave Cheney's [https://github.com/pkg/errors](https://github.com/pkg/errors) package. This hybrid works well, however, in December 2021 [https://github.com/pkg/errors](https://github.com/pkg/errors) was archived and, generally, the Go community seems to have opted for a simpler option of adding proper error context and wrapping errors. For instance, [https://earthly.dev/blog/golang-errors/](https://earthly.dev/blog/golang-errors/) was one of the more popular blog posts from 2022 and features this approach.

I decided I would return to error wrapping also, but wanted to standardize my approach similar to what Rob Pike did in his article using an `op` constant:

> In typical use, calls to errors.E will arise multiple times within a method, so we define a constant, conventionally called op, that will be passed to all E calls within the method:
>
>  func (s *Server) Delete(ref upspin.Reference) error {
>    const op errors.Op = "server.Delete"
>     ...
>
> Then through the method we use the constant to prefix each call (although the actual ordering of arguments is irrelevant, by convention op goes first):
>
> if err := authorize(user); err != nil {
>     return errors.E(op, user, errors.Permission, err)
>   }

Where I differ, is that I need to present the op stack in structured logging vs. the upspin approach of string formatting with nested indentation.

I created an OpStack function to recursively unwrap all errors wrapped inside a given error and pull out the `op` for each error, returning the full list of `ops` as a slice of string.

```go
// OpStack returns the op stack information for an error
func OpStack(err error) []string {
    type o struct {
        Op    string
        Order int
    }

    e := err
    i := 0
    var os []o

    // loop through all wrapped errors and add to struct
    // order will be from top to bottom of stack
    for errors.Unwrap(e) != nil {
        var errsError *Error
        if errors.As(e, &errsError) {
            if errsError.Op != "" {
                op := o{Op: string(errsError.Op), Order: i}
                os = append(os, op)
            }
        }
        e = errors.Unwrap(e)
        i++
    }

    // reverse the order of the stack (bottom to top)
    sort.Slice(os, func(i, j int) bool { return os[i].Order > os[j].Order })

    // pull out just the stack info, now in reversed order
    var ops []string
    for _, op := range os {
        ops = append(ops, op.Op)
    }

    return ops
}
```

I then use this function when logging errors from my API.

```go
        ops := OpStack(e)
        if len(ops) > 0 {
            j, _ := json.Marshal(ops)
            // log the error with the op stack
            lgr.Error().RawJSON("stack", j).Err(e.Err).
                Int("http_statuscode", httpStatusCode).
                Str("Kind", e.Kind.String()).
                Str("Parameter", string(e.Param)).
                Str("Code", string(e.Code)).
                Msg(errMsg)
```

This ends up producing a "pseudo stack" that is easy to look at and understand in structured logging.

```json
{
   "level": "error",
   "remote_ip": "127.0.0.1:60382",
   "user_agent": "PostmanRuntime/7.30.1",
   "request_id": "cfgihljuns2hhjb77tq0",
   "stack": [
      "diygoapi/Movie.IsValid",
      "service/MovieService.Create"
   ],
   "error": "title is required",
   "http_statuscode": 400,
   "Kind": "input validation error",
   "Parameter": "title",
   "Code": "",
   "time": 1675700438,
   "severity": "ERROR",
   "message": "error response sent to client"
}
```

I refactored my [diygoapi](https://github.com/gilcrest/diygoapi) project to now use this method by default. I have setup the project to still allow for error tracing using [https://github.com/pkg/errors](https://github.com/pkg/errors), but it's optional and needs to be set to do so on startup or using a service. I have updated the README details for errors [here](https://github.com/gilcrest/diygoapi#errors) and if you want to understand how to turn on the stack tracing using flags or environment variables, you can look [here](https://github.com/gilcrest/diygoapi#step-3---prepare-environment-2-options).

---
404 Error Illustration by [Pixeltrue](https://icons8.com/illustrations/author/5ec7b0e101d0360016f3d1b3) from [Ouch!](https://icons8.com/illustrations)
