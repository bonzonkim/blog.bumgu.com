---
title: "Go - Functional Options Pattern"
author: Bumgu
date: 2025-02-16
categories: 
    - Developer
tags: 
    - Go
slug: "go-functional-options"
---
```go
package main

import "fmt"

type Server struct {
	maxConn		int
	tls			bool
	id			string

}

func newServer(maxConn int, tls bool, id string) *Server {
	return &Server{
		maxConn:	maxConn,
		tls:		tls,
		id:			id,
	}
}

func main() {
	s := newServer(100, false, "server1")
	fmt.Println(s)
}
```
Here is a function that starts a server.
When calling `newServer()`, you must pass arguments such as `maxConn`, `tls`, and `id`.
There are only three for now, so it is hard to forget them, but with 10 or 20 arguments you may start forgetting some and the program will fail to run.

### Opts struct
Let's move the properties from the existing `Server` struct into a struct called `Opts`, and let `Server` contain an `Opts` struct.

```go
package main

import "fmt"


type Opts struct {
	maxConn		int
	tls			bool
	id			string
}

type Server struct {
	opts	Opts
}


func newServer(opts Opts) *Server {
	return &Server{
		opts:	opts,
	}
}

func main() {
	o := Opts {
		maxConn:	100,
		tls:		true,
		id:			"server1",
	}
	s := newServer(o)
	fmt.Println(s)
}
```
The code now passes a single `Opts` struct instead of three arguments, but we still need to initialize the values of all three properties.
Now let's apply the `Functional Options` pattern.

### defaultOpts
```go
package main

import "fmt"


type OptsFunc func(*Opts)

type Opts struct {
	maxConn		int
	tls			bool
	id			string
}

type Server struct {
	opts	Opts
}

func defaultOpts() Opts {
	return Opts {
		maxConn:	100,
		tls:		false,
		id:			"default-server",
	}
}


func newServer(optsFunc ...OptsFunc) *Server {
	o := defaultOpts()

	for _, fn := range optsFunc {
		fn(&o)
	}

	return &Server {
		opts:	o,
	}
}

func main() {
	s := newServer()
	fmt.Printf("%+v", s)
}
```
`OptsFunc` type: a function signature that takes `*Opts` as an argument
`defaultOpts()`: initializes and returns the **default values** for `Opts`
`newServer()`: accepts **multiple** `OptsFunc` values and initializes defaults using `defaultOpts()` (if no `OptsFunc` is passed, the defaults are used). If any `OptsFunc` values are provided, it applies them in a loop to initialize the options, then returns the result.

The loop iterates once for each `OptsFunc` passed as an argument. `fn` is a function that takes `*Opts`, so we can initialize values using the supplied `OptsFunc` by calling `fn(&o)`.

Although `main()` currently passes no values to `newServer()`, running it as-is gives us:
![](https://velog.velcdn.com/images/kellyb9/post/4f7b5d6f-6af7-4c2b-905c-cb8fa36d3df0/image.png)

The default values initialized in `defaultOpts()` are set.
Now let's create configuration functions so that users can pass custom values.

### User configuration functions
```go
func withTLS(opts *Opts) {
	opts.tls = true
}

func withMaxConn(conn int) OptsFunc {
	return func(opts *Opts) {
		opts.maxConn = conn
	}
}

func withID(id string) OptsFunc {
	return func(opts *Opts) {
		opts.id = id
	}
}
```
`withTLS()`: Since `tls` is a boolean and can only be true or false, it takes no value; using the function sets it to true. (If not used, the `false` initialized by `defaultOpts()` is used.) Since its signature matches `OptsFunc`, it can be passed to `newServer()` in `main()` as `withTLS`, without calling it like `withTLS(Opts{})`.

`withMaxConn()`: Takes a user-configured connection count and returns an `OptsFunc`.

`withID()`: Takes a user-configured ID and returns an `OptsFunc`.

They all return `OptsFunc` because `newServer()` takes arguments of that type. `withTLS` takes no user-provided value and its signature already matches `OptsFunc`, so it can be passed directly to `newServer()`.

### Results
`withTLS, withMaxconn(200), withID("server1")`
![](https://velog.velcdn.com/images/kellyb9/post/61013c30-c99c-4d20-a518-b2dea20f8d9e/image.png)

`withMaxConn(50)`
![](https://velog.velcdn.com/images/kellyb9/post/299e7eb4-47b8-4b20-9a6a-87d73da8c29e/image.png)

This demonstrates the `Functional Options` pattern: when no `OptsFunc` is provided, defaults are used; when one is provided, only the corresponding values are set to the supplied values.

I've used options like these when initializing instances with S3 APIs, Gorm, and similar tools. Trying the pattern out helped me understand its benefits.
It seems especially useful when building libraries in Go.

Thank you for reading!
