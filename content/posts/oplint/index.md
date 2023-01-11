---
title: "oplint: The op linter"
date: 2023-01-11T10:35:00-04:00
description: Linter to ensure `op` is correct for every function.
menu:
  sidebar:
    name: oplint
    weight: 20
---

I made a new linter called [oplint](https://github.com/gilcrest/oplint)! This was a really fun thing to do as I had never ventured into static analysis before. I created this tool to help me understand where an error originates without using the [pkg/errors](https://github.com/pkg/errors) error stack functionality. The `pkg/errors` project has been archived and it seems the Go community has largely moved on to just wrapping errors using the standard library. I am moving away from using `pkg/errors` to use only the standard library as well, and am trying to make sure I can trace an error source in a standardized way.

One approach is to prepend an error with the function name where the error occurred. As the error goes up the call chain, additional function names can be added while wrapping the original error. Brandon Schurman does a great job explaining this concept in the *Wrapping Errors* section of his [Effective Error Handling in Golang](https://earthly.dev/blog/golang-errors/) blog post.

For this pattern, it can be handy to define an `op` constant (short for *operation*) to capture the function name, particularly if you have multiple error return possibilities in one function (rather than copy/paste). The problem I've had using the `op` constant pattern for a while, is that I would often not be able to trust my own results as I invariably had copy/pasted something and forgotten to update the op value to match the function name.

This tool makes it so I can scan through my code easily and know I got each op correct.

Let's start by looking at a simple example of a function returning an error:

```go
package opdemo

import "fmt"

// IsEven returns an error if the number given is not even
func IsEven(n int) error {
    const op = "opdemo/IsEven"

    if n%2 != 0 {
        return fmt.Errorf("%s: %d is not even", op, n)
    }
    return nil
}
```

The error returned is formatted with a helpful locator:
> `opdemo/IsEven: 3 is not even`

The `op` format for a typical function is:

`package name` + `"/"` + `function name`
> e.g. `const op = "opdemo/IsEven"`

If the function has a value or pointer receiver:

```go
package opdemo

import "fmt"

type Number int

// IsEven returns an error if the number is not even
func (n Number) IsEven() error {
    const op = "opdemo/Number.IsEven"

    if n%2 != 0 {
        return fmt.Errorf("%s: %d is not even", op, n)
    }
    return nil
}
```

then the format should be:

`package name` + `"/"` + `type name` + `"."` + `function name`
> e.g. `const op = "opdemo/number.IsEven"`

Adding the `op` to errors adds context which can be critical to understanding your errors, particularly when wrapping errors and sending through a long chain of function calls.

## oplint command

The `oplint` command will scan the body of every function (including those with function receivers) in a given file or directory/package. If a constant is defined as either `op` or `Op`, it will check to see if the value given for said constant matches the function name which envelops it. If the value does not match, `oplint` will report the mismatch and the position.

```sh
$ cat foo.go
package opdemo

import (
    "fmt"
)

type yoda string

// Do does nothing really but returns an error
func (s *yoda) Do() error {
    const op = "opdemo/yoda.Try"

    return fmt.Errorf("%s: There is no try", op)
}

$ oplint foo.go
/Users/gilcrest/opdemo/foo.go:11:8: op constant value (opdemo/yoda.Try) does not match function name (opdemo/yoda.Do)
```

Click on the diagnostic from `oplint` to be taken directly to the spot of the value mismatch and correct it using the diagnostic message.

### oplint -missing flag

Optionally, `oplint` can also report functions that return an error, but do not have a constant named `op` defined.

```sh
$ cat foo.go
package testdata

import (
    "errors"
    "fmt"
)

func hello() error {
    return errors.New("some error message")
}

$ oplint -missing foo.go
/Users/gilcrest/opdemo/foo.go:8:1: testdata/hello returns an error but does not define an op constant
```

> The default value is false for the `-missing` flag. If it is not present, these checks will not be run.

## Next Steps

I am going to refactor the error handling in my [diygoapi project](https://github.com/gilcrest/diygoapi) and I will use this tool to aid in that effort. I will add another post describing the changes I made for errors then.

## Acknowledgements

I started using this `op` pattern after reading Rob Pike's [article on Error handling in Upspin](https://commandcenter.blogspot.com/2017/12/error-handling-in-upspin.html) back in 2017. Every so often, I go back and read this post, it's one of my favorites.

-----

Cat linting image by [oddstranger](https://www.flickr.com/people/49863366@N02) from Tokyo, Japan, [CC BY-SA 2.0](https://creativecommons.org/licenses/by-sa/2.0), via Wikimedia Commons
