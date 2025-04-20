---
title: "Go - Functional Options 패턴"
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
위와 같이 서버를 실행하는 함수가 있습니다.
`newServer()`를 호출할 때는 인자값으로 `maxConn`, `tls`, `id`등을 **필수**로 넘겨줘야 합니다.
지금은 3개밖에 안받으니 까먹을일 없겠지만 만약 인자가 10개, 20개라면 하나둘씩 까먹기 시작하고 실행에 실패할 것입니다.


### Opts 구조체
기존 `Server` 구조체에 가지고 있던 속성들을 `Opts`라는 구조체로 옮기고, `Server`는 `Opts`구조체를 가지겠습니다.

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

인자 3개를 넘겨주던 코드에서 `Opts`구조체 하나만 넘기는 코드로 바뀌긴 했지만, 결국 3개 속성의 값을 초기화 해줘야 함은 변함이 없습니다.
이제 `Functional Optsions` 패턴을 적용해보겠습니다.

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

`OptsFunc` 타입 : `*Opts` 를 인자로 받는 함수 시그니처
`defaultOpts()` : `Opts`의 **기본값** 을 초기화하고 반환함
`newServer()` : `OptsFunc`를 **여러개** 받을 수 있으며, `defaultOpts()`로 기본값 초기화 (넘겨지는 `OptsFunc`가 없으면 기본값을 사용함) 만약 받은 `OptsFunc`가 있다면 for문으로 그 값을 받아서 opts값을 초기화 후 리턴

for문 안에서는 인자로 받은 `OptsFunc` 만큼 루프를 도는데, `fn`은 `*Opts`를 인자로 받는 함수입니다. 그래서 `fn(&o)` 와 같이 사용하여 넘겨받은 `OptsFunc`로 값 초기화가 가능합니다.

현재 `main()`에서 `newServer()`에 아무런 값을 넘겨주지 않았지만 이대로 실행하면
![](https://velog.velcdn.com/images/kellyb9/post/4f7b5d6f-6af7-4c2b-905c-cb8fa36d3df0/image.png)

`defaultOpts()`에서 초기화해준 기본값이 설정됩니다.
이제 사용자 커스텀값을 넘겨줄 수 있도록 설정함수를 만들어봅시다.

### 사용자 설정 함수
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
`withTls()` : `tls`는 bool값이니 true, false만 있으므로 값을 받지 않고, 함수를 사용하면 true가 되도록 합니다. (사용하지 않으면 defaultOpts()에 초기화된 false 사용) `OptsFunc`와 타입 시그니처가 일치하기 때문에 `main()`에서 `newServer()` 호출 시 `withTLS(Opts{})`와 같이 호출하지 않고, `withTLS`와 같이 호출할 수 있습니다.

`withMaxConn()` : 사용자 설정 커넥션 값을 받고 `OptsFunc`를 리턴

`withID()` : 사용자 설정 아이디 값을 받고 `OptsFunc`를 리턴

모두 `OptsFunc`를 리턴하는 이유는 `newServer()`가 `OptsFunc`타입을 인자로 받기 때문에 그렇습니다. `withTLS`는 사용자 설정값을 받지 않기 때문에 `OptsFunc`시그니처와 일치하기 때문에 그대로 `newServer()`의 인자로 넘겨줄 수 있습니다.

### 결과
`withTLS, withMaxconn(200), withID("server1")`
![](https://velog.velcdn.com/images/kellyb9/post/61013c30-c99c-4d20-a518-b2dea20f8d9e/image.png)

`withMaxConn(50)`
![](https://velog.velcdn.com/images/kellyb9/post/299e7eb4-47b8-4b20-9a6a-87d73da8c29e/image.png)


이렇게 `OptsFunc` 를 받지 않으면 기본값을 사용하고, 
`OptsFunc`를 받으면 그 부분만 설정한 값으로 설정하는 `Functional Options` 패턴을 적용해봤습니다.


S3 api, Gorm 등을 사용할때 인스턴스를 초기화 하는과정에서 이와 같은 options를 받는걸 사용해봤는데 이렇게 적용해봄으로써 이 패턴의 이점을 알겠습니다.
특히 Go언어에서 라이브러리를 만들때 유용한 패턴인것 같습니다.

읽어주셔서 감사합니다!
