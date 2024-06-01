---
title: "I wish Golang had generic method feature"
date: 2024-06-01T18:04:39+03:30
---

Golang is my favorite language; I started my career with it, learned most backend concepts and topics with Golang,
and implemented many things from scratch with Golang. But there's one thing that really annoy me.

### Generic on methods
Honestly, why? I mean, you can implement the generic concept on functions, but why not on methods? They are the same as functions. 
For example, you can create functions like this:
```go
func SomeFunc[T any](t T){}
```
But you can't create methods like this:
```go
type A string
func (a A) SomeAnotherFunc[T any](t T){}
```
Even though they are the same thing, you can't create methods like this.

I contributed to the [gofiber](https://github.com/gofiber/fiber) project, And I implemented some generic functions([#2758](https://github.com/gofiber/fiber/issues/2758)).
I implemented functions like `QueryInt` that performed conversion tasks *(before generic were introduced in the go1.18)* .
After generic feature was released, I added new implementation of new converting data types with the help of generic in Fiber.

Now if you want recieve somethings like url query in diffrenet data types, you have two ways:
1. Use functions implemented in Fiber to convert values, like the Query function:
```go
value, err := fiber.Query[uint64](ctx, "id")
 ```
<small>In this case, you should pass the ctx value to the Query function to receive your value in the desired data type.</small> 

2. Use convertor functions by your own:
```go
  valueString := ctx.Query("id")
  value, err := strconv.ParseUint(valueString)
```
<small>In this case, you recieve the value and store it into valueString variable; Then pass it into ParseUint function to convert this data type.</small>

So, think about it, how much better and easy if just use one of the methods of `Ctx` struct and recieve the value with your favorite data type. 
```go
value, err := ctx.Query[uint64]("id")
``` 

It's funny to think about how much effort they put into adding generics without breaking things, yet they didn't add generics to methods.
