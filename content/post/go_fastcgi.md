+++
date = "2015-10-25"
title = "Go: FastCGI, Nginx"
slug = "go-fastcgi"
+++

[FastCGI](http://golang.org/pkg/net/http/fcgi/) under Go is a bit different than say a PHP fcgi. In PHP fcgi, a [new process](http://wiki.nginx.org/FcgiExample) is spawned external to the web-server while in Go, a new goroutine is launched to handle the incoming connections. So when we put fcgi under nginx for Go, nginx is effectively just a pass-thorough proxy.

In your nginx conf file under /etc/nginx/sites-available, add:

    location ~ /app.* {
        fastcgi_pass   127.0.0.1:8082;
         #proxy_pass  127.0.0.1:8082;
     }

The above tells nginx to redirect requests to 127.0.0.1/app. If you don't need fcgi, just use the "proxy_pass" for standard http. Also don't forget to create a symlink to the  sites-available/myconfig to sites-enabled and reload nginx.

Go fcgi pseudo code:

    listener, e := net.Listen("tcp", addr)
    if e != nil {
    	//do something
    }
    err = fcgi.Serve(listener, app.Handlers)
