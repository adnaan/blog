---
ID: 13
post_title: Exploring Dependency Injection in Go
author: adnaan
post_excerpt: ""
layout: post
permalink: http://adnaan.badr.in/?p=13
published: true
---

## Introduction

A lot has been said about the pros and cons of Dependency Injection. This post is less about the pattern itself and more about it's implementation design and it's side-effects w.r.t `Go`

Let's define a context for `Go` users :

As a clean programming practice, the theory of dependency injection is quite simple across several languages: 

> A dependency is passed to an object as an argument rather than the object creating or finding it.

It plainly means that the dependencies of an object are provided to it as it’s initial state. 
This is in contrast with using globals as dependencies wherein the same global resource is shared across multiple objects. It also means the object doesn't self-initialize it's dependencies.

```go
// Using a global dependency in PostService

var userService user.Service

func Init() {
 userService = user.NewService()
}

func GetComments() []Comment {
 var comments []Comment
 comments = append(comments, userService.GetComments())
 return comments
}
```

Instead `PostService` could explicitly depend on the `user.Service` resource. While initializing the `PostService` object, we would be `injecting` it's dependencies. 

```go
// Using dependency injection

type PostService struct {
	UserService user.Service
}

func (p *PostService) GetComments() []Comment {
	var comments []Comment
	comments = append(comments, p.UserService.GetComments())
	return comments
}

func main(){
 postService := &PostService{UserService:user.NewService()
 ...
 }
```


## Designing for Dependency Injection

### Goals: 

1. No Global State: No package level variables, no package level func init. Refer: [theory of modern go](https://peter.bourgon.org/blog/2017/06/09/theory-of-modern-go.html)
2. Support Multi-Mode Services: Configure a service to enable/disable capabilities.
3. Better Testability: Easy testing/mocking.
4. Better Refractor-ability: Design for code refractors. 
5. Better Readability: Explicit and readable dependency graph.

First, let's look at a more involved example : 

```go

func main(){
	asClient := aerospike.NewClient(...)
	kafkaProducer := sarama.NewAsyncProducer(...)
	authService := auth.Service{
		Key: "xxx",
		Host: "jjj"
	}

	... // more services

	userService := user.Service{
		AsClient : asClient,
		KafkaProducer: kafkaProducer,
		AuthService: authService
		... // more dependencies
	}
   
   authService2 := auth.Service {
     Key: "yyy",
     Host: "kkk"
   }
	userServiceLimitedAccess := user.Service{
		AsClient : asClient,
		KafkaProducer: kafkaProducer,
		AuthService: authService
		... // less dependencies
	}

}
```

Though the concept of DI itself is simple, it takes a bit of premeditated design to avoid a convoluted injection code.

In the above example, where  `user.Service` has multiple modes, the struct `user.Service{...}` initialization does not make clear what dependents where required to make that happen.

One can imagine this initialization pattern to get even more messy with changing requirements:

1. The service has multiple modes or capabilities. e.g: `limitedaccess`, `maintenance` , `readonly` etc. Each mode could have additional or fewer dependencies. 
2. The service has a commonly used `default` mode.
3. The service adds/removes dependencies over time(i.e capabilities). 

The following conventions could be adopted to leverage the power of DI while side-stepping `a few` of it's pitfalls:

#### **_1. Use interfaces_** 

To support multi-mode services(with a general `default` mode), declaring interfaces is the way to go. Declaring interfaces allows us to have multiple implementations of the same service. When used as a dependency, clients don't need to change the contract when a different mode of the service is passed.This is also regularly useful for testing .

```go
type AppHandler struct {
	UserService user.Service
	// ... more services.
}
```

Here, `AppHandler` will accept any service(mode) which satisfies the `user.Service` interface. We will see how it's done, further ahead.

#### **_2. Prefer construction over initialization_**

In the previous example, we used struct initializers to inject dependencies. While this is good enough for a small number of dependencies, it gets hard to manage them once the service starts adding new capabilities. It also reduces readability and requires prior implicit knowledge of the dependencies of a service.

Moreover, initializing a service using `user.Service{}` limits our ability to enable/disable and extend service capabilities. I would recommend `constructor functions` over `struct initializers` to eliminate these problems. Let's see.

Possible approaches to construct a service object using functions:

1. `New`. e.g: `NewReadOnlyService(...)`, `NewPrivilegedService(...)`
2. `Config` : `type Config struct{}...`. `NewService(c *Config)`. `NewService(nil)` would mean a default `Config`
3. Variadic `Config`. `NewService(c ...Config)`.
4. Functional Options: `NewService(options ...func(Service)`

The above ideas have been  sourced from the excellent post by Dave Cheney. Would highly recommend a nice, slow read: https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis. Actually I will be right here, while you do that.

The `functional options` pattern is reasonable enough. But still doesn't really scale if you have a large number of dependencies and multiple modes. Consider the following:

```go
  // using the functional option pattern
  myUserService := user.NewService(
  msession *mgo.Session,
  ..., // more default dependencies
  user.PrivilegedMode(redisPool *redis.Pool),
  ..., // more modes
  )
  
 // or 
 
  myUserService := user.NewService(
  user.DefaultMode(msession,...,),// more default dependencies
  user.PrivilegedMode(redisPool),
  ..., // more modes
  )
  
```

We end up with a situation where again, there are a large number of arguments to be passed into the `New` function. Also it doesn't make a lot of sense to have to declare `default` behavior as a mode.

Ideally, for a service we would want a `default` behavior which we can override to achieve a mode.

##### Functional Config Options

So, instead we use a combo of  `functional options` and  `Config` patterns to get around this complexity.

```go
// NewService configures the service
func NewService(defaultConfig Config, configOptions ...func(*Config)) Service {
	for _, option := range configOptions {
		option(&defaultConfig)
	}
	return &service{c: &defaultConfig}
}
```

Note that we didn't pass `*Config` as `defaultConfig` as we wouldn't want the caller to hold the reference to it. Additional attributes/overrides to the `defaultConfig` is done via `configOptions`.

The following snippets of code condenses the ideas we have talked about:

```go
// user/user.go
package user

import (
	"fmt"

	"github.com/garyburd/redigo/redis"
	mgo "gopkg.in/mgo.v2"
)

// Service interface
type Service interface {
	// invoked from a http request and returns the role of the user.
	HandleGetRoleRequest(uid string) string
	HandleGetAccessKey(uid string) (string, error)
}

// Config for the service
type Config struct {
	RedisPool *redis.Pool
	Msession  *mgo.Session
}

// configurable service which implements the Service interface
type service struct {
	c *Config
}

func (s *service) HandleGetRoleRequest(uid string) string {
	return getRole(s.c.Msession.Copy(), uid)
}

func (s *service) HandleGetAccessKey(uid string) (string, error) {
	if s.c.RedisPool == nil {
		return "", fmt.Errorf("redis pool is not initalized")
	}
	return getAccessKey(s.c.RedisPool.Get(), uid)
}

// NewService configures the service
func NewService(defaultConfig Config, configOptions ...func(*Config)) Service {
	for _, option := range configOptions {
		option(&defaultConfig)
	}
	return &service{c: &defaultConfig}
}

func PrivilegedMode(RedisPool *redis.Pool) func(*Config) {
	return func(c *Config) {
		c.RedisPool = RedisPool
	}
}

func getRole(session *mgo.Session, uid string) string {
	defer session.Close()
	// fetch role
	return "admin"
}

func getAccessKey(conn redis.Conn, uid string) (string, error) {
	defer conn.Close()
	//fetch accesskey
	return "xyz123", nil
}

```

```go
//main.go
package main

import (
	"github.com/adnaan/talks/dependency_injection_july2017/user"
	"github.com/garyburd/redigo/redis"
	"gopkg.in/mgo.v2"
)

type AppHandler struct {
	UserService user.Service
	// ... more services.
}

func main() {
	msession := &mgo.Session{} //dummy
	defaultConfig := &user.Config{Msession: msession}
	redisPool := &redis.Pool{} //dummy
	myUserService := user.NewService(defaultConfig)
	// override default behavior
	myPrivilegedUserService := user.NewService(defaultConfig, user.PrivilegedMode(redisPool))
	appHandler := &AppHandler{UserService: myUserService}
	appHandler2 := &AppHandler{UserService: myPrivilegedUserService}

	// If necessaryimplement functional config options for AppHandler too
	// ...
	// register appHandler to the http server.

}
```


## Testing/Mocking

Easier testing/mocking is a benefit which just falls out of implementing the discussed dependency injection pattern.

Since we already have a `Service` interface, writing a mock implementation is a no-brainer. Our test code would now look something like this.
```go
//user/mock/user.go
package mock

import "github.com/adnaan/talks/dependency_injection_july2017/user"

// configurable service which implements the Service interface
type service struct {
	c *user.Config
}

// NewService configures the service
func NewService(defaultConfig user.Config, configOptions ...func(*user.Config)) user.Service {
	for _, option := range configOptions {
		option(&defaultConfig)
	}
	return &service{c: &defaultConfig}
}
...
```

```go
// main_test.go

package main

import (
	"testing"

	"github.com/adnaan/talks/dependency_injection_july2017/user/mock"
)

func TestService(t *testing.T) {

	mockService := mock.NewService(mockConfig)
}
...
```

## Refactoring Code
Splitting our code into multiple services or even binaries becomes trivial too. 

```go
   //service1/main.go
	msession := &mgo.Session{} //dummy
	defaultConfig := user.Config{Msession: msession}
	redisPool := &redis.Pool{} //dummy
	myUserService := user.NewService(defaultConfig)
    appHandler := &AppHandler{UserService: myUserService}
    ...
```

```go
    //service2/main.go
	msession := &mgo.Session{} //dummy
	defaultConfig := user.Config{Msession: msession}
	redisPool := &redis.Pool{} //dummy
    myPrivilegedUserService := user.NewService(defaultConfig, user.PrivilegedMode(redisPool))
    appHandler := &AppHandler{UserService: myPrivilegedUserService}
    ...
```

## Dependency Graph Builders

There are various other ways to build a sensible dependency graph. Packages like [facebookgo/inject](https://github.com/facebookgo/inject), [codegangsta/inject](https://github.com/codegangsta/inject) are available to help. Unfortunately, I haven't had a chance to try them yet.

## Takeaway

Here is a one liner for the reader:

```
Pass dependencies to a service implementation as functional config options: New(defaultConfig Config, configOptions ...func(*Config) )
```

Dependency Injection in other languages consider more factors specific to the ecosystem. It's important to evaluate which of the patterns emanating from those factors can be directly mapped to Go. Hopefully I have presented a strong case for a sensible dependency injection pattern in Go.
