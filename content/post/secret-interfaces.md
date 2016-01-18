+++
date = "2016-01-08T19:45:02-07:00"
title = "\"Secret\" interfaces in go"
description = "An interesting discovery from the std lib"
tags = ["go"]
+++

I recently was looking for a way for an http server to detect if the client had disconnected. The idea being the server does not need to continue doing expensive work if the client has already stopped listening.

I was quickly pointed at the documentation for [http.CloseNotifier](https://golang.org/pkg/net/http/#CloseNotifier):

```
type CloseNotifier interface {
        // CloseNotify returns a channel that receives a single value
        // when the client connection has gone away.
        CloseNotify() <-chan bool
}
```

My first thought was "Great! How do I get one?" Well, it turns out I already have one:

> The CloseNotifier interface is implemented by ResponseWriters which allow detecting when the underlying connection has gone away.

When I make an http handler of the format `func handleSomething(w http.ResponseWriter, r *http.Request)`, there is a good chance the `w` there already implements
the `CloseNotifier` interface, and I just have to reach out and grab it:

```
notifier, ok := w.(http.CloseNotifier)
if !ok {
  log.Println("Doesn't implement notify")    
  return
} 
select{
  case <- notifier.CloseNotify():
    log.Println("Client hung up")
    cancelWorkInProgress()
  case data := <- realWork()
    w.Write(data)    
}
```

That is a bit contrived, and real usages are a bit more complicated (you may use the channel inside a [context](https://blog.golang.org/context) 
object passed down your call stack for example), but I was really amazed that this insanely useful feature was already implemented by a type 
I already was using and I didn't 
even know it! Why is this not exposed more prominently, and why isn't `CloseNotify` on `http.ResponseWriter` itself? Two good reasons:

1. It requires a particular level of expertise to use correctly. I'm not suggesting that the standard library needs to hide complex concepts from people
entirely, but cancellation can be tricky to get right, and it might not be best to point people right at it in such a core interface.
2. Not every implementation of `ResponseWriter` has the reponsibilility, or even the ability, to implement the `CloseNoifier` interface. An in-process
response recorder may not even have a remote client. A gzip middleware may have replaced the original ResponseWriter with a different one, 
and may not have preserved the reference to the original client. It may not be possible to provide a notification given the particular implementation you are using.
By not putting `CloseNotify` on the main interface, you are not forcing anybody to lie about their capabilities.

The http package's public contract is essentially: "A ResponseWriter may or may not be able to tell you if the connection closes. If it can, it will implement `CloseNotifier`, but there
is no guarantee any given `ResponseWriter` will." 

My title may be a bit misleading. There is nothing "Secret" about `http.CloseNotifier`, but it has some properties I think are interesting:

- It is never directly passed in or returned from any api in the package.
- Only unexported types implement it.
- It provides features not generally required for the "90% use case".
- You have to read the docs to even know it exists.

This has the benefits of keeping this functionality "out of the way" in an already large package, 
but still availible if really needed. 

### where else can we use stuff like this?

I'd love to see more examples of this type of design pattern in the wild. I used it recently when working on a data access package for our redis server. The top level data access interface is strictly domain oriented:

```
type DataAccess interface{
	GetUser(id int) ()*User,error)
	CreateUser(*User) (error)
}
``` 
There is nothing in the interface that tells you what the underlying storage is, nor should there be. There are however a few cases where 
I may want to access the redis instance directly (integration tests are a good case. Complex, one-time data migrations another.) So my implementation type (which is not exported) also implements 

`type Connector interface {GetConnection() redis.Conn}`

so that special cases like this can still access the underlying data source. There are strong comments to discourage such access, and it is likely to break if you dpend on it too much, but at least the capabilities are there.

Got any other examples? 



