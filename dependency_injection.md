---
post_title: 'Exploring Dependency Injection in Go'
layout: post_type_probably_post
published: false
---

As a clean programming practice, the theory of dependency injection is quite old and simple across several languages: 

> A dependency should be passed to a client as an argument instead of the client building or finding one.

It plainly means that the dependencies of a client are provided to it as it’s initial state. 
This is in contrast with using globals for dependencies wherein the same global resource is shared across multiple clients.

```go
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
