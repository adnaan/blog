---
ID: 13
post_title: Exploring Dependency Injection in Go
author: adnaan
post_excerpt: ""
layout: post
permalink: >
  http://adnaan.badr.in/2017/07/15/exploring-dependency-injection-in-go/
published: true
post_date: 2017-07-15 03:42:44
---
<h2>Introduction</h2>
There is lot of material available about the pros and cons of Dependency Injection. This post is less about the pattern itself and more about its implementation design and it's side-effects w.r.t <code>Go</code>.

Let's setup a context for <code>Go</code> users :

As a clean programming practice, the theory of dependency injection is quite simple across several languages:
<blockquote>A dependency is passed to an object as an argument than the object creating or finding it.</blockquote>
It plainly means that the dependencies of an object are passed to it as it’s initial state.
This is in contrast with using globals as dependencies wherein the same global resource is shared across multiple objects. It also means the object doesn't self-initialize its dependencies.
<pre><code class="go">// Using a global dependency in PostService

var userService user.Service

func Init() {
userService = user.NewService()
}

func GetComments() []Comment {
var comments []Comment
comments = append(comments, userService.GetComments())
return comments
}
</code></pre>
Instead <code>PostService</code> could explicitly depend on the <code>user.Service</code> resource. While initializing the <code>PostService</code> object, we would be <code>injecting</code> it's dependencies.
<pre><code class="go">// Using dependency injection

type PostService struct {
UserService user.Service
}

func (p *PostService) GetComments() []Comment {
var comments []Comment
comments = append(comments, p.UserService.GetComments())
return comments
}

func main(){
postService := &amp;PostService{UserService:user.NewService()
...
}
</code></pre>
<h2>Designing for Dependency Injection</h2>
<h3>Goals:</h3>
<ol>
 	<li>No Global State: No package level variables, no package level func init. Refer: <a href="https://peter.bourgon.org/blog/2017/06/09/theory-of-modern-go.html">theory of modern go</a></li>
 	<li>Support Multi-Mode Services: Configure a service to enable/disable capabilities.</li>
 	<li>Better Testability: Easy testing/mocking.</li>
 	<li>Better Refractor-ability: Design for code refractors.</li>
 	<li>Better Readability: Explicit and readable dependency graph.</li>
</ol>
First, let's look at a more involved example :
<pre><code class="go">
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
AuthService: authService2
... // less dependencies
}

}
</code></pre>
Though the concept of DI itself is simple, it takes a bit of premeditated design to avoid a convoluted injection code.

In the above example, where <code>user.Service</code> has multiple modes, the struct <code>user.Service{...}</code> initialization does not make clear what dependents where required to make that happen.

One can imagine this initialization pattern to get even more messy with changing requirements. Service usability and behavior could change based on:
<ol>
 	<li>The service has multiple modes or capabilities. e.g: <code>limitedaccess</code>, <code>maintenance</code> , <code>readonly</code> etc.</li>
 	<li>The service has a commonly used <code>default</code> mode.</li>
 	<li>The service adds/removes dependencies over time(i.e capabilities).</li>
</ol>
The following conventions could be adopted to leverage the power of DI while side-stepping <code>a few</code> of it's pitfalls:
<h4><strong><em>1. Use interfaces</em></strong></h4>
To support multi-mode services(with a general <code>default</code> mode), declaring interfaces is the way to go. Declaring interfaces allows us to have multiple implementations of the same service. When used as a dependency, clients don't need to change the contract when a different mode of the service is passed.This is also regularly useful for testing .
<pre><code class="go">type AppHandler struct {
UserService user.Service
// ... more services.
}
</code></pre>
Here, <code>AppHandler</code> will accept any service(mode) which satisfies the <code>user.Service</code> interface. We will see how it's done, further ahead.
<h4><strong><em>2. Prefer construction over initialization</em></strong></h4>
In the previous example, we used struct initializers to inject dependencies. While this is good enough for a few dependencies, it gets hard to manage them once the service starts adding new capabilities. It also reduces readability and requires prior implicit knowledge of the dependencies of a service.

Moreover, initializing a service using <code>user.Service{}</code> limits our ability to enable/disable and extend service capabilities. I would recommend <code>constructor functions</code> over <code>struct initializers</code> to eliminate these problems. Let's see.

Possible approaches to construct a service object using functions:
<ol>
 	<li><code>New</code>. e.g: <code>NewReadOnlyService(...)</code>, <code>NewPrivilegedService(...)</code></li>
 	<li><code>Config</code> : <code>type Config struct{}...</code>. <code>NewService(c *Config)</code>. <code>NewService(nil)</code> <code>nil</code> would mean a default <code>Config</code></li>
 	<li>Variadic <code>Config</code>. <code>NewService(c ...Config)</code>.</li>
 	<li>Functional Options: <code>NewService(options ...func(Service)</code></li>
</ol>
The above ideas have been sourced from the excellent post by Dave Cheney. Would highly recommend a nice, slow read: <a href="https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis.">https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis.</a> Actually I will be right here, while you do that.

The <code>functional options</code> pattern is reasonable enough. But still doesn't scale if you have a large number of dependencies and multiple modes. Consider the following:
<pre><code class="go">// using the functional option pattern
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

</code></pre>
We end up with a situation where again, there are many  arguments passed into the <code>New</code> function. Also it doesn't make a lot of sense to have to declare <code>default</code> behavior as a mode.

Ideally, for a service we would want a <code>default</code> behavior which we can override to create a new mode.
<h5>Functional Config Options</h5>
So, instead we use a combo of <code>functional options</code> and <code>Config</code> patterns to get around this complexity.
<pre><code class="go">// NewService configures the service
func NewService(defaultConfig Config, configOptions ...func(*Config)) Service {
for _, option := range configOptions {
option(&amp;defaultConfig)
}
return &amp;service{c: &amp;defaultConfig}
}
</code></pre>
Note that we didn't pass <code>*Config</code> as <code>defaultConfig</code> as we wouldn't want the caller to hold the reference to it. Additional attributes/overrides to the <code>defaultConfig</code> is done via <code>configOptions</code>.

The following snippets of code condenses the ideas we have talked about:
<pre><code class="go">// user/user.go
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
Msession *mgo.Session
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
option(&amp;defaultConfig)
}
return &amp;service{c: &amp;defaultConfig}
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

</code></pre>
<pre><code class="go">//main.go
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
msession := &amp;mgo.Session{} //dummy
defaultConfig := &amp;user.Config{Msession: msession}
redisPool := &amp;redis.Pool{} //dummy
myUserService := user.NewService(defaultConfig)
// override default behavior
myPrivilegedUserService := user.NewService(defaultConfig, user.PrivilegedMode(redisPool))
appHandler := &amp;AppHandler{UserService: myUserService}
appHandler2 := &amp;AppHandler{UserService: myPrivilegedUserService}

// If necessaryimplement functional config options for AppHandler too
// ...
// register appHandler to the http server.

}
</code></pre>
<h2>Testing/Mocking</h2>
Easier testing/mocking is a benefit which just falls out of implementing the dependency injection pattern.

Since we already have a <code>Service</code> interface, writing a mock implementation is a no-brainer. Our test code would now look something like this.
<pre><code class="go">//user/mock/user.go
package mock

import "github.com/adnaan/talks/dependency_injection_july2017/user"

// configurable service which implements the Service interface
type service struct {
c *user.Config
}

// NewService configures the service
func NewService(defaultConfig user.Config, configOptions ...func(*user.Config)) user.Service {
for _, option := range configOptions {
option(&amp;defaultConfig)
}
return &amp;service{c: &amp;defaultConfig}
}
...
</code></pre>
<pre><code class="go">// main_test.go

package main

import (
"testing"

"github.com/adnaan/talks/dependency_injection_july2017/user/mock"
)

func TestService(t *testing.T) {

mockService := mock.NewService(mockConfig)
}
...
</code></pre>
<h2>Refactoring Code</h2>
A sorted out dependency graph makes splitting our code into new services trivial.
<pre><code class="go">//service1/main.go
msession := &amp;mgo.Session{} //dummy
defaultConfig := user.Config{Msession: msession}
redisPool := &amp;redis.Pool{} //dummy
myUserService := user.NewService(defaultConfig)
appHandler := &amp;AppHandler{UserService: myUserService}
...
</code></pre>
<pre><code class="go">//service2/main.go
msession := &amp;mgo.Session{} //dummy
defaultConfig := user.Config{Msession: msession}
redisPool := &amp;redis.Pool{} //dummy
myPrivilegedUserService := user.NewService(defaultConfig, user.PrivilegedMode(redisPool))
appHandler := &amp;AppHandler{UserService: myPrivilegedUserService}
...
</code></pre>
<h2>Dependency Graph Builders</h2>
There are various other ways to build a sensible dependency graph. Packages like <a href="https://github.com/facebookgo/inject">facebookgo/inject</a>, <a href="https://github.com/codegangsta/inject">codegangsta/inject</a> are available to help. Unfortunately, I haven't tried them yet.
<h2>Takeaway</h2>
Here is a tl;dr for the reader:
<pre><code>Pass dependencies to a service implementation as functional config options:

New(defaultConfig Config, configOptions ...func(*Config))Service.

where Service is an interface.
</code></pre>
Dependency Injection in other languages consider more factors specific to the ecosystem. It's important to evaluate which of the patterns emanating from those factors are applicable in Go. Hopefully I have presented a strong case for a sensible dependency injection pattern in Go.