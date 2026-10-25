---
title: "Go - The Difference Between http.HandleFunc and http.HandlerFunc"
author: Bumgu
date: 2025-01-07
categories: 
    - Developer
tags: 
    - Go
    - http
slug: "http_handleFunc_handlerFunc"
---
### 1. http.HandleFunc

* `http.HandleFunc` is a **function**.
* It registers a handler to process requests for a particular path.
* The registered function follows the `func(http.ResponseWriter, *http.Request)` signature.

#### 1-1. Usage example
```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
)

type Response struct {
	Message		string  `json:"message"`
}

func helloHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json") // Set the response header
	w.WriteHeader(http.StatusOK)    // Set the status code in the header

	response := Response {
		Message:	"hello world",
	}

	if err := json.NewEncoder(w).Encode(response); err != nil {
		fmt.Errorf("failed to encode json reponse", err)
	}
}

func main() {
	http.HandleFunc("/", helloHandler) // Register the handler for the '/' route

	fmt.Println("Server start on port :8080")

	if err := http.ListenAndServe(":8080", nil); err != nil {
		panic("failed to start server")		
	}
}
```
![](/images/post/17-go-http/1.png)

### 2. http.HandlerFunc
* `http.HandlerFunc` is a **type**.
* It is defined as a function type that implements the `http.Handler` interface.
* This type can act as an `http.Handler` through its `ServeHTTP` method.

#### 2-1. Usage example
```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
)

type Response struct {
	Message		string  `json:"message"`
}

func helloHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json") // Set the response header
	w.WriteHeader(http.StatusOK)    // Set the status code in the header

	response := Response {
		Message:	"hello world",
	}

	if err := json.NewEncoder(w).Encode(response); err != nil {
		fmt.Errorf("failed to encode json reponse", err)
	}
}

func main() {
	handler := http.HandlerFunc(helloHandler) // Convert the function to a handler
	http.Handle("/", handler) // Register the handler

	fmt.Println("Server start on port :8080")

	if err := http.ListenAndServe(":8080", nil); err != nil {
		panic("failed to start server")
	}
}
```
![](/images/post/17-go-http/2.png)

### 3. Differences
Examples 1-1 and 2-1 produce exactly the same result. We know they differ in that one is a type and the other is a function, but how do they differ in use, and when should each be used?
First, `http.HandlerFunc` converts a function into an `http.Handler`. Therefore, it can be used when a handler is explicitly required or when using middleware.

### 3-1. Middleware example
```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
)

type Response struct {
	Message		string  `json:"message"`
}

// Logging middleware
func loggingMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Printf("Request: %s %s\n", r.Method, r.URL.Path) // Print the log
		next.ServeHTTP(w, r) // Call the next handler
	})
}

// Default handler
func helloHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json") // Set the response header
	w.WriteHeader(http.StatusOK)    // Set the status code in the header

	response := Response {
		Message:	"hello Middleware",
	}

	if err := json.NewEncoder(w).Encode(response); err != nil {
		fmt.Errorf("failed to encode json reponse", err)
	}
}

func main() {
	// Wrap the default handler with middleware
	http.Handle("/", loggingMiddleware(http.HandlerFunc(helloHandler)))
	fmt.Println("server start on port :8080")

	if err := http.ListenAndServe(":8080", nil); err != nil {
		panic("failed to start server")
	}
}
```

![](/images/post/17-go-http/3.png)

Looking at the terminal at the bottom of the screen on the left, when we visit `:8080/`, the middleware runs and prints `r.Method` and `r.URL.Path`.

The reason `logginMiddleware()` takes a `Handler` argument is that `handler.ServeHTTP()` is defined in the `Handler` interface.
```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

### 4. Summary of the differences

|Characteristic | http.HandlerFunc | http.HandleFunc|
|--|--|--|
|Role | Type (converts a function to `http.Handler`) | Function (connects a route and handler)|
|Purpose | Use as a handler or connect as middleware | Simply register a handler for a path|
|Internal behavior | Processes requests through `ServeHTTP` method | Internally converts to `HandlerFunc`|
|Usage | Use directly as a handler | Connect a path and handler function|
|Example | `http.Handle("/path", HandlerFunc)` | `http.HandleFunc("/path", handlerFunc)`|

### 4. Conclusion

Recently, I've been trying various things with Go.
I've deliberately written some scripts needed at work in Go, and ported an old toy project to Go as well...
It is certainly a very compelling language.

Thank you for reading.
