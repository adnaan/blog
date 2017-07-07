---
ID: 13
post_title: Exploring Dependency Injection in Go
author: adnaan
post_excerpt: ""
layout: post
permalink: http://adnaan.badr.in/?p=13
published: false
---

## Introduction
As a clean programming practice, the theory of dependency injection is quite old and simple across several languages: 

> A dependency is passed to an object as an argument rather than the object creating or finding it.

It plainly means that the dependencies of an object are provided to it as it’s initial state. 
This is in contrast with using globals for dependencies wherein the same global resource is shared across multiple objects.

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

Instead `PostService` explicity depends on the `user.Service` resource. While initializing the `PostService` object, we would be `injecting` it's dependencies. 

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

## Designing for Injection

### Construction over initialization.
### Naming

## Testing/Mocking

## Refractoring Code

## From Monolith to Microservices

## Object Graph Builders

## Conclusion

