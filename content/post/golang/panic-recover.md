---
title: "Panic Recover 笔记"
date: 2023-11-30T01:20:40+08:00
draft: false
tags: ["golang"]
categories: ["golang"]
author: "EilenC"
---

> Golang func使用的一些笔记

# Golang panic recover 笔记


## Panic 异常

Go的类型系统会在编译时捕获很多错误，但有些错误只能在运行时检查，如数组访问越界、空指针引用等。这些运行时错误会引起painc异常。

一般而言，当panic异常发生时，程序会中断运行，并立即执行在该goroutine（可以先理解成线程，在第8章会详细介绍）中被延迟的函数（defer 机制）。随后，程序崩溃并输出日志信息。日志信息包括panic value和函数调用的堆栈跟踪信息。panic value通常是某种错误信息。对于每个goroutine，日志信息中都会有与之相对的，发生panic时的函数调用堆栈跟踪信息。通常，我们不需要再次运行程序去定位问题，日志信息已经提供了足够的诊断依据。因此，在我们填写问题报告时，一般会将panic异常和日志信息一并记录。

不是所有的panic异常都来自运行时，直接调用内置的panic函数也会引发panic异常；panic函数接受任何值作为参数。当某些不应该发生的场景发生时，我们就应该调用panic。比如，当程序到达了某条逻辑上不可能到达的路径。



## Recover 捕获异常

在 Go 语言中，错误(error)被认为是一种可以预期的结果；而异常(panic)则是一种非预期的结果，发生异常可能表示程序中存在 BUG 或发生了其它不可控的问题。Go 语言推荐使用 recover 函数将内部异常转为错误处理，这使得用户可以真正的关心业务相关的错误处理。

如果在deferred函数中调用了内置函数recover，并且定义该defer语句的函数发生了panic异常，recover会使程序从panic中恢复，并返回panic value。导致panic异常的函数不会继续运行，但能正常返回。在未发生panic时调用recover，recover会返回nil。



## 常用示例

### recover

```go
defer func() {
    if err := recover(); err != nil {
        fmt.Printf("recover panic: %+v", err)
    }
}()
```



### panic

```go
panic("panic error")
```



### panic 没有 recover

code

```go
package main

import "fmt"

// main.go
func main() {
    fmt.Println(echo(10, 0))
    fmt.Println("end")
}

func echo(a, b int) int {
    v := a / b
    fmt.Println("echo a/b: ", v)
    return v
}
```

output

```powershell
GOROOT=D:\gosdk\go1.21.1 #gosetup
panic: runtime error: integer divide by zero    
                                                
goroutine 1 [running]:                          
main.echo(0x0?, 0xc00003bf30?)                  
        E:/work/awesomeProject/main.go:12 +0x98 
main.main()                                     
        E:/work/awesomeProject/main.go:7 +0x1e  

Process finished with the exit code 2
```

除法计算里0当除数触发`panic`,并且没有`recover`所以程序终止并退出了。



### panic 有 recover

#### 外部recover

code

```go
package main

import "fmt"

// main.go
func main() {
    defer func() {
        if err := recover(); err != nil {
            fmt.Printf("recover panic: %+v", err)
        }
    }()
    fmt.Println(echo(10, 0))
    fmt.Println("end")
}

func echo(a, b int) int {
    v := a / b
    fmt.Println("echo a/b: ", v)
    return v
}
```

output

```powershell
GOROOT=D:\gosdk\go1.21.1 #gosetup
recover panic: runtime error: integer divide by zero 
Process finished with the exit code 0
```

`echo`函数外部有recover触发panic后会影响程序后续运行 上方的 `fmt.Println("end")`没有执行。



#### 内部recover

code

```go
package main

import "fmt"

// main.go
func main() {
    fmt.Println(echo(10, 0))
    fmt.Println("end")
}

func echo(a, b int) int {
    defer func() {
        if err := recover(); err != nil {
            fmt.Printf("recover panic: %+v", err)
        }
    }()
    v := a / b
    fmt.Println("echo a/b: ", v)
    return v
}
```

output

```powershell
GOROOT=D:\gosdk\go1.21.1 #gosetup
recover panic: runtime error: integer divide by zero0 
end                                                   

Process finished with the exit code 0
```

`echo`函数内部有recover触发panic后会不影响程序后续运行。



#### 外部recover(协程)

code

```go
package main

import (
	"fmt"
	"time"
)

// main.go
func main() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Printf("recover panic: %+v", err)
		}
	}()
	go echo(10, 0)
	fmt.Println(echo(10, 5))
	fmt.Println("end")
	time.Sleep(1 * time.Minute)
}

func echo(a, b int) int {
	v := a / b
	fmt.Println("echo a/b: ", v)
	return v
}
```

output

```powershell
GOROOT=D:\gosdk\go1.21.1 #gosetup
echo a/b:  2
2                                                
panic: runtime error: integer divide by zero     
                                                 
goroutine 6 [running]:                           
main.echo(0x0?, 0x0?)                            
        E:/Mycode/awesomeProject/main.go:22 +0x98
created by main.main in goroutine 1              
        E:/Mycode/awesomeProject/main.go:15 +0x3b

Process finished with the exit code 2
```

`main`函数有recover触发并且使用`go`调用`echo` panic后recover捕获不到错误,`main`程序会异常退出。



#### 内部recover(协程)

code

```go
package main

import (
	"fmt"
	"time"
)

// main.go
func main() {
	go echo(10, 0)
	fmt.Println(echo(10, 5))
	fmt.Println("end")
	time.Sleep(1 * time.Minute)
}

func echo(a, b int) int {
	defer func() {
		if err := recover(); err != nil {
			fmt.Printf("recover panic: %+v", err)
		}
	}()
	v := a / b
	fmt.Println("echo a/b: ", v)
	return v
}
```

output

```powershell
GOROOT=D:\gosdk\go1.21.1 #gosetup
echo a/b:  2
2
end
recover panic: runtime error: integer divide by zero
```

`echo`函数有recover触发并且使用`go`调用`echo` panic后recover捕获错误,`main`程序会照常执行不会异常退出。



## 总结

golang 中的 defer 的使用可以总结为，如果 panic 和 recover 发生在同一个协程，那么 recover 是可以捕获的，如果 panic 和 recover 发生在不同的协程，那么 recover 是不可以捕获的。



## 引用文章与材料

- [Go语言高级编程-1.7 错误和异常](https://chai2010.cn/advanced-go-programming-book/ch1-basic/ch1-07-error-and-panic.html)
- [Go语言设计与实现-5.4 panic 和 recover](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-panic-recover/#54-panic-%e5%92%8c-recover)
- [Golang panic recover详解](https://m.haicoder.net/note/golang/golang-panic-recover.html)
