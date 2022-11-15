---
author: "Narcissus"
title: "Golang调试器Delve学习"
date: "2022-11-13"
lastmod: "2022-11-13"
description: "学习非常适合Golang的调试器Delve"
tags: ["Golang"]
categories: ["Golang"]
password: ""
---

Go语言支持GDB、LLDB、Delve等调试器，其中Delve是专门为Go语言设计开发的调试器，本身也是采用Golang实现，下面学习Delve。

## 1. 安装

[Delve](https://github.com/go-delve/delve)可以使用如下方式进行安装：

```go
go install github.com/go-delve/delve/cmd/dlv@latest
```

> 注意：此方式安装必须要求Go版本为1.16及以上。

安装完成后，GOBIN目录下将会有dlv的可执行文件。

## 2. 基础使用

### 2.1 命令

使用`dlv debug file.go`即可启动调试，输入`help`即可查询可用命令。

```
Running the program:
    call ------------------------ Resumes process, injecting a function call (EXPERIMENTAL!!!)
    continue (alias: c) --------- Run until breakpoint or program termination.
    next (alias: n) ------------- Step over to next source line.
    rebuild --------------------- Rebuild the target executable and restarts it. It does not work if the executable was not built by delve.
    restart (alias: r) ---------- Restart process.
    step (alias: s) ------------- Single step through program.
    step-instruction (alias: si)  Single step a single cpu instruction.
    stepout (alias: so) --------- Step out of the current function.

Manipulating breakpoints:
    break (alias: b) ------- Sets a breakpoint.
    breakpoints (alias: bp)  Print out info for active breakpoints.
    clear ------------------ Deletes breakpoint.
    clearall --------------- Deletes multiple breakpoints.
    condition (alias: cond)  Set breakpoint condition.
    on --------------------- Executes a command when a breakpoint is hit.
    toggle ----------------- Toggles on or off a breakpoint.
    trace (alias: t) ------- Set tracepoint.
    watch ------------------ Set watchpoint.

Viewing program variables and memory:
    args ----------------- Print function arguments.
    display -------------- Print value of an expression every time the program stops.
    examinemem (alias: x)  Examine raw memory at the given address.
    locals --------------- Print local variables.
    print (alias: p) ----- Evaluate an expression.
    regs ----------------- Print contents of CPU registers.
    set ------------------ Changes the value of a variable.
    vars ----------------- Print package variables.
    whatis --------------- Prints type of an expression.

Listing and switching between threads and goroutines:
    goroutine (alias: gr) -- Shows or changes current goroutine
    goroutines (alias: grs)  List program goroutines.
    thread (alias: tr) ----- Switch to the specified thread.
    threads ---------------- Print out info for every traced thread.

Viewing the call stack and selecting frames:
    deferred --------- Executes command in the context of a deferred call.
    down ------------- Move the current frame down.
    frame ------------ Set the current frame, or execute command on a different frame.
    stack (alias: bt)  Print stack trace.
    up --------------- Move the current frame up.

Other commands:
    config --------------------- Changes configuration parameters.
    disassemble (alias: disass)  Disassembler.
    dump ----------------------- Creates a core dump from the current process state
    edit (alias: ed) ----------- Open where you are in $DELVE_EDITOR or $EDITOR
    exit (alias: quit | q) ----- Exit the debugger.
    funcs ---------------------- Print list of functions.
    help (alias: h) ------------ Prints the help message.
    libraries ------------------ List loaded dynamic libraries
    list (alias: ls | l) ------- Show source code.
    source --------------------- Executes a file containing a list of delve commands
    sources -------------------- Print list of source files.
    transcript ----------------- Appends command output to a file.
    types ---------------------- Print list of types
```

### 2.2 标准输入的调试方式

因为这方调试方式并不是主流，所以暂不支持优雅的方式进行调试，可以采用如下方式：

1. 启动一个Shell输入如下命令调试相关文件

```shell
dlv debug --headless --listen=<:XYZ> <file.go>
```

> listen后面就是监听的端口，然后就可以在该终端进行标准输入

2. 启动另一个端口，输入如下命令进行正常调试即可

```shell
dlv connect :XYZ
```

**第一个终端的输入将进入第二个，最后的输入将在第一个终端。**

