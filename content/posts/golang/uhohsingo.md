+++
title = "Uh-Oh’s in Go — Slice of Pointers"
description = "Tripping hazards in Go"
tags = [
    "golang",
]
categories = [
    "tutorial"
]
date = 2018-05-23
author = "Nitish Malhotra"
+++

![xkcd](https://imgs.xkcd.com/comics/pointers.png)

Go makes dealing with slices and other primitive data structures a complete cake walk. For those coming from the daunting world of pointers, in C and C++, working in a Go is a blessing (most of the time). For those that have adopted go after working in languages like JS/Python, there is not much of a difference, apart from the syntax.

However more often than not a JS/Python developer or any Go newbie would trip on pointers. Most likely it would be in the scenario presented below.

---

## Scenario

Consider the case where you need to load a slice of string pointers, `[]*string{}` with some data.

Let’s see some code -

```go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    // Declare a list/slice of string pointers
    listOfNumberStrings := []*string{}

    // Forward declaration of the variable we will store the data in before adding to slice
    var numberString string

    // Loop from 0 to 9
    for i := 0; i < 10; i++ {

        // Format the string with '#' prefixed to the number
        numberString = fmt.SPrintf("#%s", strconv.Itoa(i))

        // Add number string to the slice
        listOfNumberStrings = append(listOfNumberStrings, &numberString)
    }

    for _, n := range listOfNumberStrings {
        fmt.Printf("%s\n", *n)
    }
}
```

The code sample above, generates numbers from 0 to 9. For each number (`int`), we convert it, into its string representation using `strconv.Itoa` . Next we prefix the generated number string with the `#` symbol. And finally append it to the targeted slice.

Let’s look at the output from running that code snippet

```console
➜  sample go run main.go
#9
#9
#9
#9
#9
#9
#9
#9
#9
#9
```

> What happened here ???
> Why do I only see the final number `#9` being printed ??? I am pretty sure I added the other numbers to the list! Let’s add some debug to the sample program.

```go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    // Declare a list/slice of string pointers
    listOfNumberStrings := []*string{}

    // Forward declaration of the variable we will store the data in before adding to slice
    var numberString string

    // Loop from 0 to 9
    for i := 0; i < 10; i++ {
        // Format the string with '#' prefixed to the number
        numberString = fmt.Sprintf("#%s", strconv.Itoa(i))
                fmt.Printf("Adding number %s to the slice\n", numberString)
        // Add number string to the slice
        listOfNumberStrings = append(listOfNumberStrings, &numberString)
    }

    for _, n := range listOfNumberStrings {
        fmt.Printf("%s\n", *n)
    }
}
```

Output from the run with debug -

```console
sample go run main.go
Adding number #0 to the slice
Adding number #1 to the slice
Adding number #2 to the slice
Adding number #3 to the slice
Adding number #4 to the slice
Adding number #5 to the slice
Adding number #6 to the slice
Adding number #7 to the slice
Adding number #8 to the slice
Adding number #9 to the slice
```

> I see them being added … *scratching head*
> WHY IS THIS HAPPENING TO ME !!!????
> $@#! My Life !!

---

> **Easy there cowboy ... Let’s look at what really happened, shall we …**

```go
var numberString string
```

What happens here is that `numberString` gets allocated on the stack, at let’s say, memory address `0x3AF1D234`.

![numberString on the stack at address 0x3AF1D234](https://miro.medium.com/max/520/1*15Rbt3dzl-U5BI57d1alSA.png)
*numberString on the stack at address 0x3AF1D234*

```go
for i := 0; i < 10; i++ {
    numberString = fmt.Sprintf("#%s", strconv.Itoa(i))
    listOfNumberStrings = append(listOfNumberStrings, &numberString)
}
```

What we do now is we loop through from 0 to 9.

**First iteration `[i = 0]`**

Here we generate the string `"#0"` and store it in the variable `numberString`.

![numberString stored at 0x3AF1D234 with content "#0"](https://miro.medium.com/max/604/1*0ti6HW7R57n057yHltMk9Q.png)
*`numberString` stored at `0x3AF1D234` with content `"#0"`*

Next we take the address of `numberString` (`&numberString`), which equates to `0x3AF1D234` , and append it to the `listOfNumberStrings` slice.

> listOfNumberStrings now looks like :

![listOfNumberStrings slice with value 0x3AF1D234](https://miro.medium.com/max/382/1*lqhnW94y7EYO66gcAScLkA.png)
*`listOfNumberStrings` slice with value `0x3AF1D234`*

**Second iteration `[i = 1]`**

> We follow the same steps as above.

Here we generate a string `"#1"` and store it in the same variable `numberString`.

![numberString stored at 0x3AF1D234 with content "#1"](https://miro.medium.com/max/604/1*Repod_CioITvZXA0QDGKqw.png)

*`numberString` stored at `0x3AF1D234` with content `"#1"`*

Next we take the address of `numberString` (`&numberString`), which equates to `0x3AF1D234` , and append it to the `listOfNumberStrings` slice.

> listOfNumberStrings now looks like:

![Two items in the slice BOTH with values 0x3AF1D234.](https://miro.medium.com/max/782/1*TXkH3liojhp49agMFMkWqg.png)
*Two items in the slice BOTH with values 0x3AF1D234.*

Hopefully it is now starting to become more obvious to you.

The slice now has two members. But both the members, at index `0` and `1`, store the value `0x3AF1D234` (memory address of `numberString`) .

However, keep in mind, that the last string stored in `numberString`, at the end of the second iteration, was `“#1”`.

This above steps repeat till we reach the last iterations.

The final string stored in `numberString`, at the end of the last iteration, was `“#9”`.

---

Alright so lets look at what happens when we try to print the content of each element stored in the slice, by de-referencing it (the pointer) using the `*` operator.

```go
for _, n := range listOfNumberStrings {
    fmt.Printf("%s\n", *n)
}
```

Since each element of the slice stores the value `0x3AF1D234` (as seen in the examples above) , de-referencing the element at that would give you the value stored at that memory address.

From the final iteration, we know that the last value to be stored was `“#9”`. Hence, the output looks like shown below.

The last string stored in the variable `numberString` => `"#9"`.

```console
➜  sample go run main.go
#9
#9
#9
#9
#9
#9
#9
#9
#9
#9
```

## Solution

There is a pretty easy way to resolve the issue. It would require you to modify the location of where you declare your variable `numberString`.

```go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    listOfNumberStrings := []*string{}

    for i := 0; i < 10; i++ {
        var numberString string
        numberString = fmt.Sprintf("#%s", strconv.Itoa(i))
        listOfNumberStrings = append(listOfNumberStrings, &numberString)
    }

    for _, n := range listOfNumberStrings {
        fmt.Printf("%s\n", *n)
    }

    return
}
```

We now declare the variable inside our `for` loop.

What this accomplishes is, that at every iteration through the loop we are forced to re-declare the variable `numberString` on the stack, thereby giving it a new unique memory address.

We then update the variable with the generated string and append its address into the slice. The result is that every element of the slice, now stores a unique memory address.

The output from running the above snippet would be -

```console
➜  sample go run main.go
#0
#1
#2
#3
#4
#5
#6
#7
#8
#9
```

---

I hope this post would help someone out there. I came up with the idea to write this post based on my experience with a Junior engineer at my workplace, where he hit this exact scenario and seemed to be completed baffled by it. It reminded me of the time I fell into the same trap, while working in C at my previous job, when I was a Junior engineer.

> PS : If you come from the wonderful world of pointers in C/C++ … Honestly, you have all made (and learned from) this mistake already ;)!

*Orginally published on [medium](https://medium.com/codezillas/uh-ohs-in-go-slice-of-pointers-c0a30669feee)*

{{ template "_internal/disqus.html" . }}
