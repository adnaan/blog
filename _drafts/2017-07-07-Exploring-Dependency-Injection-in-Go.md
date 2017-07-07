---
ID: 13
post_title: Exploring Dependency Injection in Go
author: adnaan
post_excerpt: ""
layout: post
permalink: http://adnaan.badr.in/?p=13
published: false
---
As a clean programming practice, the theory of dependency injection is quite old and simple across several languages: 

> A dependency is passed to a client as an argument rather than the client building or finding it.

It plainly means that the dependencies of a client are provided to it as it’s initial state. 
This is in contrast with using globals for dependencies wherein the same global resource is shared across multiple clients.

```
// Using a global dependency. 

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
