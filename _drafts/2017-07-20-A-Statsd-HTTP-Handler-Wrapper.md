---
ID: 78
post_title: A Statsd HTTP Handler Wrapper
author: adnaan
post_excerpt: ""
layout: post
permalink: http://adnaan.badr.in/?p=78
published: false
---
Statsd is a simple and effective tool to trace app metrics: http request latency, throughput, runtime metrics etc. Using the  package [alexcesaro/statsd.v2](https://godoc.org/gopkg.in/alexcesaro/statsd.v2), tracking response time of a request is a one liner:


```go

func main(){
	r := chi.NewRouter()
	r.Get("/",handleHome)
}

func handleHome(w http.ResponseWriter, r *http.Request) {
	defer c.NewTiming().Send(&quot;homepage.response_time&quot;)
	defer c.Increment("foo.counter")
	time.Sleep(time.Millisecond * 1000)
	w.WriteHeader(http.StatusOK)
	w.Write([]byte("OK"))
}

```

But this gets cumbersome if you have more than a couple of handlers and would want to track other metrics too.

A http handler wrapper is an idiomatic approach to overcome this. I have written a package to that effect: [statsdwrap](https://github.com/adnaan/statsdwrap). It's a tiny package for the [chi](https://github.com/go-chi/chi) router. 

Usage:

  ```go
  r := chi.NewRouter()
  statsdClient, _ := statsd.New(
	statsd.Prefix("myapp"),
	statsd.Address("localhost:8125"),
  )
  wrap := statsdwrap.NewChi("user_service", statsdClient)
  handleHome := func(w http.ResponseWriter, r *http.Request) {
	  time.Sleep(time.Millisecond * 1000)
	  w.WriteHeader(http.StatusOK)
	  w.Write([]byte("OK"))
  }
  r.Get(wrap.HandlerFunc("home", "/", handleHome))
```


By default  the wrapper sends the following metrics for http: `response_time`,
`count` and `status<HTTPStatusCode>.count` Since the code is trivial I would recommend folks to copy it and modify to suit their own purposes. I would try to extend this to other popular routers on a later date if it makes sense. Here's the single file which constitutes the package:  

```go
// Package statsdwrap exposes wrappers for http.Handler and http.HandlerFunc which send
// metrics to statsd.
// Usage:
// 		r := chi.NewRouter()
// 		statsdClient, _ := statsd.New(
// 		statsd.Prefix(&quot;myapp&quot;),
// 		statsd.Address(&quot;localhost:8125&quot;),
// 		)
// 		wrap := statsdwrap.NewChi(&quot;user_service&quot;, statsdClient)
// 		handleHome := func(w http.ResponseWriter, r *http.Request) {
// 			time.Sleep(time.Millisecond * 1000)
// 			w.WriteHeader(http.StatusOK)
// 			w.Write([]byte(&quot;OK&quot;))
// 		}
// 		r.Get(wrap.HandlerFunc(&quot;home&quot;, &quot;/&quot;, handleHome))
package statsdwrap

import (
	&quot;bytes&quot;
	&quot;fmt&quot;
	&quot;net/http&quot;
	&quot;strings&quot;

	&quot;github.com/pressly/chi/middleware&quot;
	&quot;gopkg.in/alexcesaro/statsd.v2&quot;
)

// HandlerWrapper ...
type HandlerWrapper interface {
	Handler(routeName string, pattern string, handler http.Handler) (string, http.Handler)
	HandlerFunc(routeName string, pattern string, handlerFunc http.HandlerFunc) (string, http.HandlerFunc)
}

// HTTPTxn a single http transaction record
type HTTPTxn interface {
	Write(status int)
	End()
}

// NewChi statsd wrapper client. Usage: NewChi(&quot;acme&quot;,statsdClient). The wrapper sends the metrics: response_time,
// count and status&lt;HTTPStatusCode&gt;.count. e.g. :
// acme.home.response_time where home is the route name
// acme.home.count
// acme.home.status404.count
func NewChi(prefix string, statsdClient *statsd.Client) HandlerWrapper {
	var cloneStatsdClient *statsd.Client
	if prefix == &quot;&quot; {
		cloneStatsdClient = statsdClient.Clone()
	} else {
		cloneStatsdClient = statsdClient.Clone(statsd.Prefix(prefix))
	}
	return &amp;defaultWrapper{
		client: cloneStatsdClient,
	}

}

// defaultWrapper statsd client
type defaultWrapper struct {
	client *statsd.Client
}

// startTransaction ...
func (d *defaultWrapper) startTransaction(name string, w middleware.WrapResponseWriter, r *http.Request) HTTPTxn {
	entry := &amp;httpTxn{
		name:               name,
		timing:             d.client.NewTiming(),
		responseTimeBucket: strings.Join([]string{name, &quot;response_time&quot;}, &quot;.&quot;),
		hitsBucket:         strings.Join([]string{name, &quot;count&quot;}, &quot;.&quot;),
		defaultWrapper:     d,
		ww:                 w,
		request:            r,
		buf:                &amp;bytes.Buffer{},
	}

	return entry
}

type httpTxn struct {
	name               string
	responseTimeBucket string
	hitsBucket         string

	timing statsd.Timing
	*defaultWrapper
	ww      middleware.WrapResponseWriter
	request *http.Request
	buf     *bytes.Buffer
}

func (d *httpTxn) Write(status int) {
	d.timing.Send(d.responseTimeBucket)
	httpStatusBucket := fmt.Sprintf(&quot;%s.http%d&quot;, d.name, d.ww.Status())
	d.client.Increment(httpStatusBucket)
	d.client.Increment(d.hitsBucket)
}

func (d *httpTxn) End() {
	d.Write(d.ww.Status())
}

// Handler ...
func (d *defaultWrapper) Handler(routeName string, pattern string, handler http.Handler) (string, http.Handler) {
	return pattern, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		ww := middleware.NewWrapResponseWriter(w, r.ProtoMajor)
		txn := d.startTransaction(routeName, ww, r)
		defer txn.End()

		handler.ServeHTTP(ww, r)
	})
}

// HandlerFunc ...
func (d *defaultWrapper) HandlerFunc(routeName string, pattern string, handlerFunc http.HandlerFunc) (string, http.HandlerFunc) {
	p, h := d.Handler(routeName, pattern, handlerFunc)
	return p, func(w http.ResponseWriter, r *http.Request) { h.ServeHTTP(w, r) }
}
```

We can use the above pattern to write wrappers for different routers, metrics, logging etc. Use this with something like the [go-runtime-metrics](https://github.com/bmhatfield/go-runtime-metrics) package and you have got a nice instrumentation going.