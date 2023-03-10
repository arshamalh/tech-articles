---
title: "Trying to shut down a gin server... (goroutines and channels tips)"
datePublished: Fri Mar 10 2023 04:15:42 GMT+0000 (Coordinated Universal Time)
cuid: clf20ywtn00010al4amz8gd3f
slug: trying-to-shut-down-a-gin-server
tags: go, channels, gin, goroutines, gin-gonic

---

Sometimes I write about specific real-world problems I faced at work in the past.  
In this article, I was trying to gracefully shut down my gin server and start it again through the application itself, without re-running my application. I learned some precious things that I'd really love to share with you.

## So, What's the problem?

Running a server, using `gin.Run()` or `http.ListenAndServe()`, will block the goroutine if it's in the main process, so it will block the main goroutine:

```go
func main() {
  ginApp := gin.Default()
  ginApp.Run(":80") // This code is blocking
  // next lines will never run because of blocking code.
  FuncForStoppingGinServer() // This function can't be used.
}
```

## Use another goroutine

We can simply block another goroutine and help the main goroutine to remain free.

```go
func main() {
  ginApp := gin.Default()
  go ginApp.Run(":80") // This code is not blocking anymore.
  FuncForStoppingGinServer() // This function will run now.
}
```

But, what should we do inside `FuncForStoppingGinServer()`??

Nothing! The answer is that gin itself doesn't have any function for shutting down its server! But before getting into that, let's try another wrong way!!

## Just kill it!

Are you thinking about stopping your gin server using goroutines and channels? (forget about handling active connections for now)

```go
var ginApp *gin.Engine

func main() {
  // Make an engine
  ginApp = gin.Default()

  // Make a stop channel
  stopChan := make(chan struct{})

  go func() {
    ginApp.Run(":80") // blocking process
    fmt.Println(">> 1 <<")
    for range stopChan {
      return
    }
  }()
  fmt.Println(">> 2 <<")
  // terminate gin server after 3 seconds
  time.Sleep(time.Second * 3)
  stopChan <- struct{}{}
  fmt.Println(">> 3 <<")
}
```

This sounds to be a good approach, but what do you think is wrong here?

Which numbers will be printed? (1, 2, 3)?

As I mentioned before, [`ginApp.Run`](http://ginApp.Run)`(":80")` is a blocking process and it blocks its goroutine, so we will send it to the channel `stopChan`, BUT, we will never be able to **read** from the `stopChan` using our `for range loop`, because that came after a blocking process and "&gt;&gt; 1 &lt;&lt;" will not be printed.

And as our channel is not buffered, the sender which is the main goroutine also will be blocked until somebody read from that! exactly before printing "&gt;&gt; 3 &lt;&lt;" and it will not be printed too.

"&gt;&gt; 2 &lt;&lt;" will be printed anyway 😄

**Additional fun fact:** If we make our channel buffered (size 1 is enough for this case), the main goroutine will send to it and go ahead, then it will complete its work and close, whenever the main goroutine job is done, the whole program exists! and any other goroutine will die with it, just make this change and see the result:

```go
stopChan := make(chan struct{}, 1)
```

I also should mention that `chan struct{}` as the channel type and empty struct `struct{}{}` will consume no memory at all, and it's known as a signal only channel.

I finally realized there is no way to solve this problem this way: [Is there a way to stop a long blocking function?](https://stackoverflow.com/a/20429950/12972198)

You can also learn more about stopping goroutines in this Question: [How to stop a goroutine](https://stackoverflow.com/questions/6807590/how-to-stop-a-goroutine), And I'll cover more examples later. (you can follow me to get noticed)

## Back into the problem!

So using Goroutines didn't solve our problem, How about trying `http.Server.Shutdown()` method?

```go
var ginApp *gin.Engine

func main() {
  // Make an engine
  ginApp = gin.Default()

  // Make a http server
  httpServer := &http.Server{
    Addr:    ":80",
    Handler: ginApp,
  }

  // Launch http server in a separate goroutine
  go httpServer.ListenAndServe()

  // Stop the server
  time.Sleep(time.Second * 5)
  fmt.Println("We're going to stop gin server")
  httpServer.Shutdown(context.Background())
}
```

It may be helpful to read this question: [Graceful stop of gin server](https://stackoverflow.com/questions/65686384/graceful-stop-of-gin-server)

## It works, but what is still wrong?

Now, we are shutting down our server inside of our program, but how can we start it again? Maybe by running `ListenAndServe()` again?

As I asked in this question: [How to stop and start gin server multiple times in one run](https://stackoverflow.com/questions/73563023/how-to-stop-and-start-gin-server-multiple-times-in-one-run) I'm trying to implement something like this semi-code:

```go
srv := NewServer()
srv.Start()
srv.Stop()
srv.Start()
srv.Stop()
srv.Start()
```

Let's do it with a little refactoring:

```go
type Server struct {
  httpServer *http.Server
}

func main() {
  srv := NewHTTPServer()
  srv.Start()
  time.Sleep(time.Second * 2)
  srv.Stop()
  time.Sleep(time.Second * 2)
  srv.Start()

  select {} // Simulate other processes
}

func NewHTTPServer() *Server {
  return &Server{
    httpServer: &http.Server{
      Addr:    ":80",
      Handler: gin.Default(),
    },
  }
}

func (s *Server) Start() {
  go func() {
    if err := s.httpServer.ListenAndServe(); err != nil {
      fmt.Println(">>>", err)
    }
  }()
}

func (s *Server) Stop() {
  s.httpServer.Shutdown(context.Background())
}
```

Run this code and see what happens! Output:

```go
...Gin logs...
>>> http: Server closed
>>> http: Server closed
```

What? we called `srv.Stop()` just once, but our server closed twice! Why? According to `.Shutdown()` functions docs:

> Once Shutdown has been called on a server, it may not be reused; future calls to methods such as Serve will return ErrServerClosed.

So, it's officially not working!

But there is still one little change, we can make the server struct again from scratch, every time we start it.

Add a `serve()` function to the last code and change the main function, other codes are the same.

```go
func main() {
  srv := serve()
  time.Sleep(time.Second * 2)
  srv.Stop()
  time.Sleep(time.Second * 2)
  srv = serve()
  srv.Start()

  select {} // Simulate other processes
}

func serve() *Server {
  srv := NewHTTPServer()
  // Register handlers or other stuff
  srv.Start()
  return srv
}
```

This is something that we want, but you can make the code cleaner.

## Other solutions

In my case, I was able to use Fiber instead, it fits way much better in my projects, but this may not be the solution you are looking for, Also you can have a look at this page: [Graceful restart or stop](https://chenyitian.gitbooks.io/gin-web-framework/content/docs/38.html)

## Final quote

This article had a different feeling for me, it was an up-and-down path with some tips and some not working approaches, I hope you feel the same and I'll be happy to hear your thoughts.