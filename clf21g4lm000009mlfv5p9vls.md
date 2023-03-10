---
title: "How to make a simple interactive shell in Go"
datePublished: Fri Mar 10 2023 04:29:05 GMT+0000 (Coordinated Universal Time)
cuid: clf21g4lm000009mlfv5p9vls
slug: making-a-simple-interactive-shell-in-go
tags: go

---

What is more exciting than making real-world applications? 🤔  
For me, it's teaching people how to make them again!

## Our Study

As I've spoken about my Dockeroller project earlier in my articles, it's a project that helps you control your docker daemon through different gates, such as Telegram or API.

I realized these gates should be easily configurable by the user, We should be able to turn some gates on or off, or change their tokens, ports, passwords, etc. One approach was using [Viper](https://github.com/spf13/viper) and dockeroller.yml file (implementing and teaching this approach is on my todo list)

Another approach can be an interactive shell! Let's do it!

> Completed code is available in my Github gists [(Link)](https://gist.github.com/arshamalh/819afbbe6fb2d31a8040096eb71f5de3).

## Welcome to the stage!

Try to be clean by separating things! In this case, I separated my code into different stages, such as:

* Welcome stage
    
* Help stage
    
* Gates stage
    
* Telegram
    
* API
    

Each stage has a message that we defined using constants at the beginning of the code, in bigger projects, they could be placed in for example a `msg` package.

We used backtick for multiline strings and we wrote constant type only once because all of them are in the same type.

```go
const (
	msg_welcome string = `
Welcome to Dockeroller!
It's sample multiline!
`

	msg_help = `
Dockeroller is an open-source project made for fun.
It's sample multiline!
`
```

## Do it forever!

The simplest form of interactive shells, use an infinite loop and it iterates until you break it, But don't worry, it's not bad at all, I'll tell you why.

In any iteration, we get something new from the user, and in that time, our app is not consuming any resources, After passing desired input to the shell, it'll process the result and go to the next iteration.  
Actually, our shell is constantly sleeping between iterations, it's waiting for us to pass it our input.

Have a look at this code:

```go
func main() {
  var stage int = 0
  for {
    switch stage {
    case 0:
      stage = StageWelcome()
    case 1:
      stage = StageHelp()
    case 2:
      stage = StageGates()
    case 11:
      stage = StageTelegram()
    case 12:
      stage = StageAPI()
    }
  }
}
```

Functions started with the word `StageXxx()` are our handlers, they handle what happens in (on 😂) a stage, we'll get into them soon.

Also, it may be clear that any stage is assigned to a number, but it could be assigned to anything else, but that may not be needed for a project this size.

**As a recap** We have a for loop, we iterate over it, and in every iteration, we check which stage we are on (using `switch`) and handle that stage.

Also, we update our stage after each iteration, but why? let's find out.

## Where would you like to go after?

It's time to update our welcome message!

```go
const (
	msg_welcome string = `
Welcome to Dockeroller!
Where would you like to go after? (choose only number)
1 - help
2 - gates
`
```

So we let the user decide which stage is its next place! and it's the reason we update the stage every time. Updating `stage` variables is a part of navigating between Stages.  
Each handler, according to its inputs, will decide where we would go next.

## Shell input & output

What is the point of this article without going into the terminal itself? This is just so easy, you may know how to use the `fmt.Print()` function, it Prints something on the terminal, **Without any default formatting**. There are some other variants of it such as `fmt.Println()` for printing with a line break and `fmt.Printf()` for printing with a specified format.

OK, there are a few more functions in `fmt` package, here we need `fmt.Scanln()` or `fmt.Scan()` (there is also a `fmt.Scanf()` for scanning according to a specified format)

Let's have a very simple example:

```go
package main

import "fmt"

func main() {
  var name string
  fmt.Scanln(&name)
  fmt.Println("hello", name)
}
```

> If you don't pass pointer to Scanln, it won't have any effect.

We can add extra print statements to make it more beautiful. We can also pass multiple pointers of different types to it, Scan will separate them by space and then populate them.

```go
package main

import "fmt"

func main() {
  var name string
  var age int
  fmt.Print("> ") // Just for beauty
  fmt.Scanln(&name, &age)
  fmt.Println("Hello", name, "you're", age, "years old.")
}
```

The output will be like:

![go interactive shell result](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ggjw9vs5oa9i50gporgl.png align="left")

## Basic Handlers

I think this should make sense now:

```go
// Welcome stage handler
func StageWelcome() int {
  fmt.Print(msg_welcome)
  return getStage()
}

// Help stage handler
func StageHelp() int {
  fmt.Print(msg_help)
  return getStage()
}

// Helper function for asking stage number and return it for next decisions
func getStage() (stage int) {
  fmt.Print("> ")
  fmt.Scanln(&stage)
  return
}
```

If you think it's not clear, I suggest you put these codes together now and then move on.

## Gates handlers

Welcome is a first-level stage,  
Help and Gates are second-level,  
Telegram & API are third-level and there can be more levels.

How can we realize it? well, there are many algorithms for it, but here, if we are in the Gates stage and we should go on with option 1 (telegram) or 2 (API) I add 10 to stage number, and in the main switch, telegram will be 11 and API will be 12. if we want to come back from this level, we will return 0 which means Welcome (stage) handler.

```go
func StageGates() int {
  fmt.Print(msg_gates)
  if stage := getStage(); stage != 0 {
    return stage + 10
  }
  return 0
}
```

## Telegram & API handlers

I just explain one of them, following the same approach.

```go
func StageTelegram() int {
  fmt.Print(msg_telegram)

  // Get a value (token) 
  var token string
  getInput("Token: ", &token)

  // Get another value (username) 
  var username string
  getInput("Username: @", &username)
  // Telegram usernames start with @,
  // Some users may include it, some may not,
  // So we helped users by including it.
  // And username variable won't have @ in it.

  fmt.Println("Username and token successfully sat!")
  return 0 // Return to the main menu after succefull config
}

// Helper function to print a message and get a value (by populating a pointer)
func getInput(msg string, value interface{}) {
  fmt.Print(msg)
  fmt.Scanln(value) // DO NOT include &, because it's a pointer when is passed to this function!
}
```

## Final quote

I hope you enjoyed this article, feel free to share your thoughts and critics with me. For more beautiful UIs, you can use this open-source package: [promptui](https://github.com/manifoldco/promptui)