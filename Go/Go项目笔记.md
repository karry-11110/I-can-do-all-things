# TonyBaiGo语言第一课



### 1 跟踪函数调用链：理解代码更直观



#### 1.1  使用 defer 可以跟踪函数的执行过程

```GO
// trace.go
package main
  
func Trace(name string) func() {
    println("enter:", name)
    return func() {
        println("exit:", name)
    }
}

func foo() {
    defer Trace("foo")()
    bar()
}

func bar() {
    defer Trace("bar")()
}

func main() {
    defer Trace("main")()
    foo()
}
```

```GO
enter: main
enter: foo
enter: bar
exit: bar
exit: foo
exit: main
```

我们在每个函数的入口处都使用 defer 设置了一个 deferred 函数。根据第 23 讲中讲解的 defer 的运作机制，Go 会在 defer 设置 deferred 函数时对 defer 后面的表达式进行求值来确定defer函数。

我们以 foo 函数中的defer Trace("foo")()这行代码为例，Go 会对 defer 后面的表达式Trace("foo")()进行求值。**由于这个表达式包含一个函数调用Trace("foo")，所以这个函数会被执行**。上面的 Trace 函数只接受一个参数，˙这个参数代表函数名，Trace 会首先打印进入某函数的日志，比如：“enter: foo”。然后返回一个闭包函数，这个闭包函数一旦被执行，就会输出离开某函数的日志。**在 foo 函数中，这个由 Trace 函数返回的闭包函数就被设置为了 deferred 函数**，于是当 foo 函数返回后，这个闭包函数就会被执行，输出“exit: foo”的日志。

我们会发现这里还是有一些**“瑕疵”**，也就是离我们期望的“跟踪函数调用链”的实现还有一些不足之处。这里我列举了几点：

- 调用 Trace 时需手动显式传入要跟踪的函数名；

- 如果是并发应用，不同 Goroutine 中函数链跟踪混在一起无法分辨；

- 输出的跟踪结果缺少层次感，调用关系不易识别；

- 对要跟踪的函数，需手动调用 Trace 函数。

  

#### 1.2 自动获取所跟踪函数的函数名

一旦实现了自动获取函数名，所有支持函数调用链跟踪的函数都只需使用下面调用形式的 Trace 函数就可以了：

```GO
defer Trace()()
```

新版 Trace 函数的实现以及它的使用方法

```GO
func Trace() func() {
    pc, _, _, ok := runtime.Caller(1)
    if !ok {
        panic("not found caller")
    }

    fn := runtime.FuncForPC(pc)
    name := fn.Name()

    println("enter:", name)
    return func() { println("exit:", name) }
}

func foo() {
    defer Trace()()
    bar()
}

func bar() {
    defer Trace()()
}

func main() {
    defer Trace()()
    foo()
}
```

​	在这一版Trace函数中，我们通过runtime.Caller函数获得当**前Goroutine的函数调用栈**上的信息，runtime.Caller的参数标识是要获取的哪一个栈帧上的信息，当参数为0时，返回的是Caller函数的调用者的函数信息，在这里就是 Trace 函数。但我们需要的是 Trace 函数的调用者的信息，于是我们传入 1。Caller 函数有四个返回值：第一个返回值代表的是程序计数（pc）；第二个和第三个参数代表对应函数所在的源文件名以及所在行数，这里我们暂时不需要；最后一个参数代表是否能成功获取这些信息，如果获取失败，我们抛出 panic。接下来，我们通过 runtime.FuncForPC 函数和程序计数器（PC）得到被跟踪函数的函数名称。

```go
enter: main.main
enter: main.foo
enter: main.bar
exit: main.bar
exit: main.foo
exit: main.main
```



#### 1.3 增加 Goroutine 标识

上面的 Trace 函数在面对只有一个 Goroutine 的时候，还是可以支撑的，但当程序中并发运行多个 Goroutine 的时候，多个函数调用链的出入口信息输出就会混杂在一起，无法分辨。接下来我们还继续对 Trace 函数进行改造，让它支持多 Goroutine 函数调用链的跟踪。我们的方案就是在**输出的函数出入口信息时，带上一个在程序每次执行时能唯一区分 Goroutine 的 Goroutine ID**。

但Goroutine 也没有 ID 信息啊！的确如此，Go 核心团队为了避免**Goroutine ID 的滥用**，故意没有将 Goroutine ID 暴露给开发者。但在 Go 标准库的 h2_bundle.go 中，我们却发现了一个获取 Goroutine ID 的标准方法：

```go
// $GOROOT/src/net/http/h2_bundle.go
var http2goroutineSpace = []byte("goroutine ")

func http2curGoroutineID() uint64 {
    bp := http2littleBuf.Get().(*[]byte)
    defer http2littleBuf.Put(bp)
    b := *bp
    b = b[:runtime.Stack(b, false)]
    // Parse the 4707 out of "goroutine 4707 ["
    b = bytes.TrimPrefix(b, http2goroutineSpace)
    i := bytes.IndexByte(b, ' ')
    if i < 0 {
        panic(fmt.Sprintf("No space found in %q", b))
    }
    b = b[:i]
    n, err := http2parseUintBytes(b, 10, 64)
    if err != nil {
        panic(fmt.Sprintf("Failed to parse goroutine ID out of %q: %v", b, err))
    }
    return n
}
```

不过，由于 http2curGoroutineID 不是一个导出函数，我们无法直接使用。我们可以把它复制出来改造一下：

```go
// trace2/trace.go
var goroutineSpace = []byte("goroutine ")

func curGoroutineID() uint64 {
    b := make([]byte, 64)
    b = b[:runtime.Stack(b, false)]
    // Parse the 4707 out of "goroutine 4707 ["
    b = bytes.TrimPrefix(b, goroutineSpace)
    i := bytes.IndexByte(b, ' ')
    if i < 0 {
        panic(fmt.Sprintf("No space found in %q", b))
    }
    b = b[:i]
    n, err := strconv.ParseUint(string(b), 10, 64)
    if err != nil {
        panic(fmt.Sprintf("Failed to parse goroutine ID out of %q: %v", b, err))
    }
    return n
}
```

这里，我们改造了两个地方。一个地方是通过直接创建一个 byte 切片赋值给 b，替代原 http2curGoroutineID 函数中从一个 pool 池获取 byte 切片的方式，另外一个是使用 strconv.ParseUint 替代了原先的 http2parseUintBytes。改造后，我们就可以直接使用 curGoroutineID 函数来获取 Goroutine 的 ID 信息了。

在 Trace 函数中添加 Goroutine ID 信息的输出

```go
// trace2/trace.go
func Trace() func() {
    pc, _, _, ok := runtime.Caller(1)
    if !ok {
        panic("not found caller")
    }

    fn := runtime.FuncForPC(pc)
    name := fn.Name()

    gid := curGoroutineID()
    fmt.Printf("g[%05d]: enter: [%s]\n", gid, name)
    return func() { fmt.Printf("g[%05d]: exit: [%s]\n", gid, name) }
}
```

我们输出的 Goroutine ID 为 5 位数字，如果 ID 值不足 5 位，则左补零，这一切都是 Printf 函数的格式控制字符串“%05d”帮助我们实现的。这样对齐 Goroutine ID 的位数，为的是输出信息格式的一致性更好。如果你的 Go 程序中 Goroutine 的数量超过了 5 位数可以表示的数值范围，也可以自行调整控制字符串。

接下来，我们也要对示例进行一些调整，将这个程序由单 Goroutine 改为多 Goroutine 并发的，这样才能验证支持多 Goroutine 的新版 Trace 函数是否好用：

```go
// trace2/trace.go
func A1() {
    defer Trace()()
    B1()
}

func B1() {
    defer Trace()()
    C1()
}

func C1() {
    defer Trace()()
    D()
}

func D() {
    defer Trace()()
}

func A2() {
    defer Trace()()
    B2()
}
func B2() {
    defer Trace()()
    C2()
}
func C2() {
    defer Trace()()
    D()
}

func main() {
    var wg sync.WaitGroup
    wg.Add(1)
    go func() {
        A2()
        wg.Done()
    }()

    A1()
    wg.Wait()
}
```

新示例程序共有两个 Goroutine，main groutine 的调用链为A1 -> B1 -> C1 -> D，而另外一个 Goroutine 的函数调用链为A2 -> B2 -> C2 -> D。我们来看一下这个程序的执行结果是否和原代码中两个 Goroutine 的调用链一致：

```go
g[00001]: enter: [main.A1]
g[00001]: enter: [main.B1]
g[00018]: enter: [main.A2]
g[00001]: enter: [main.C1]
g[00001]: enter: [main.D]
g[00001]: exit: [main.D]
g[00001]: exit: [main.C1]
g[00001]: exit: [main.B1]
g[00001]: exit: [main.A1]
g[00018]: enter: [main.B2]
g[00018]: enter: [main.C2]
g[00018]: enter: [main.D]
g[00018]: exit: [main.D]
g[00018]: exit: [main.C2]
g[00018]: exit: [main.B2]
g[00018]: exit: [main.A2]
```

但由于 Go 运行时对 Goroutine 调度顺序的不确定性，各个 Goroutine 的输出还是会存在交织在一起的问题，这会给你查看某个 Goroutine 的函数调用链跟踪信息带来阻碍。这里我提供一个小技巧：**你可以将程序的输出重定向到一个本地文件中，然后通过 Goroutine ID 过滤出（可使用 grep 工具）你想查看的 groutine 的全部函数跟踪信息。**



#### 1.4 让输出的跟踪信息更具层次感

```go
// trace3/trace.go

func printTrace(id uint64, name, arrow string, indent int) {
    indents := ""
    for i := 0; i < indent; i++ {
        indents += "    "
    }
    fmt.Printf("g[%05d]:%s%s%s\n", id, indents, arrow, name)
}

var mu sync.Mutex
var m = make(map[uint64]int)

func Trace() func() {
    pc, _, _, ok := runtime.Caller(1)
    if !ok {
        panic("not found caller")
    }

    fn := runtime.FuncForPC(pc)
    name := fn.Name()
    gid := curGoroutineID()

    mu.Lock()
    indents := m[gid]    // 获取当前gid对应的缩进层次
    m[gid] = indents + 1 // 缩进层次+1后存入map
    mu.Unlock()
    printTrace(gid, name, "->", indents+1)
    return func() {
        mu.Lock()
        indents := m[gid]    // 获取当前gid对应的缩进层次
        m[gid] = indents - 1 // 缩进层次-1后存入map
        mu.Unlock()
        printTrace(gid, name, "<-", indents)
    }
}
```

我们使用了一个 map 类型变量 m 来保存每个 Goroutine 当前的缩进信息：m 的 key 为 Goroutine 的 ID，值为缩进的层次。然后，考虑到 Trace 函数可能在并发环境中运行，根据我们在第 16 讲中提到的“map 不支持并发写”的注意事项，我们增加了一个 sync.Mutex 实例 mu 用于同步对 m 的写操作。

在这一版的 Trace 函数实现中，我们也把输出出入口跟踪信息的操作提取到了一个独立的函数 printTrace 中，这个函数会根据传入的 Goroutine ID、函数名、箭头类型与缩进层次值，按预定的格式拼接跟踪信息并输出。

```go
g[00001]:    ->main.A1
g[00001]:        ->main.B1
g[00001]:            ->main.C1
g[00001]:                ->main.D
g[00001]:                <-main.D
g[00001]:            <-main.C1
g[00001]:        <-main.B1
g[00001]:    <-main.A1
g[00018]:    ->main.A2
g[00018]:        ->main.B2
g[00018]:            ->main.C2
g[00018]:                ->main.D
g[00018]:                <-main.D
g[00018]:            <-main.C2
g[00018]:        <-main.B2
g[00018]:    <-main.A2
```



#### 1.5 利用代码生成自动注入 Trace 函数

要实现向目标代码中的函数 / 方法自动注入 Trace 函数，我们首先要做的就是将上面 Trace 函数相关的代码打包到一个 module 中以方便其他 module 导入。

1. **将 Trace 函数放入一个独立的 module 中**

我们创建一个名为 instrument_trace 的目录，进入这个目录后，通过 go mod init 命令创建一个名为 instrument_trace 的 module,接下来，我们将最新版的 trace.go 放入到该目录下，将包名改为 trace，并仅保留 Trace 函数、Trace 使用的函数以及包级变量，其他函数一律删除掉。这样，一个独立的 trace 包就提取完毕了。

**作为 trace 包的作者，我们有义务告诉大家如何使用 trace 包。**在 Go 中，通常我们会用一个 example_test.go 文件来编写使用 trace 包的演示代码，下面就是我们为 trace 包提供的 example_test.go 文件：

```go
// instrument_trace/example_test.go
package trace_test
  
import (
    trace "github.com/bigwhite/instrument_trace"
)

func a() {
    defer trace.Trace()()
    b()
}

func b() {
    defer trace.Trace()()
    c()
}

func c() {
    defer trace.Trace()()
    d()
}

func d() {
    defer trace.Trace()()
}

func ExampleTrace() {
    a()
    // Output:
    // g[00001]:    ->github.com/bigwhite/instrument_trace_test.a
    // g[00001]:        ->github.com/bigwhite/instrument_trace_test.b
    // g[00001]:            ->github.com/bigwhite/instrument_trace_test.c
    // g[00001]:                ->github.com/bigwhite/instrument_trace_test.d
    // g[00001]:                <-github.com/bigwhite/instrument_trace_test.d
    // g[00001]:            <-github.com/bigwhite/instrument_trace_test.c
    // g[00001]:        <-github.com/bigwhite/instrument_trace_test.b
    // g[00001]:    <-github.com/bigwhite/instrument_trace_test.a
}
```

在 example_test.go 文件中，我们用 ExampleXXX 形式的函数表示一个示例，go test 命令会扫描 example_test.go 中的以 Example 为前缀的函数并执行这些函数。每个 ExampleXXX 函数需要包含预期的输出，就像上面 ExampleTrace 函数尾部那样，我们在一大段注释中提供这个函数执行后的预期输出，预期输出的内容从// Output:的下一行开始。go test 会将 ExampleTrace 的输出与预期输出对比，如果不一致，会报测试错误。从这一点，我们可以看出 example_test.go 也是 trace 包单元测试的一部分。

现在 Trace 函数已经被放入到独立的包中了，接下来我们就来看看如何将它自动注入到要跟踪的函数中去。

2. **自动注入 Trace 函数**

现在，我们在 instrument_trace module 下面增加一个命令行工具，这个工具可以以一个 Go 源文件为单位，自动向这个 Go 源文件中的所有函数注入 Trace 函数。

我们再根据带有可执行文件的 Go 项目布局，在 instrument_trace module 中增加 cmd/instrument 目录，这个工具的 main 包就放在这个目录下，而真正实现自动注入 Trace 函数的代码呢，被我们放在了 instrumenter 目录下

```go
$tree ./instrument_trace -F
./instrument_trace
├── Makefile
├── cmd/
│   └── instrument/
│       └── main.go  # instrument命令行工具的main包
├── example_test.go
├── go.mod
├── go.sum
├── instrumenter/    # 自动注入逻辑的相关结构
│   ├── ast/
│   │   └── ast.go
│   └── instrumenter.go
└── trace.go
```

main.go源码：

```go
//  instrument_trace/cmd/instrument/main.go

... ...

var (
    wrote bool
)

func init() {
    flag.BoolVar(&wrote, "w", false, "write result to (source) file instead of stdout")
}

func usage() {
    fmt.Println("instrument [-w] xxx.go")
    flag.PrintDefaults()
}

func main() {
    fmt.Println(os.Args)
    flag.Usage = usage
    flag.Parse() // 解析命令行参数

    if len(os.Args) < 2 { // 对命令行参数个数进行校验
        usage()
        return
    }

    var file string
    if len(os.Args) == 3 {
        file = os.Args[2]
    }

    if len(os.Args) == 2 {
        file = os.Args[1]
    }
    if filepath.Ext(file) != ".go" { // 对源文件扩展名进行校验
        usage()
        return
    }

    var ins instrumenter.Instrumenter // 声明instrumenter.Instrumenter接口类型变量
    
    // 创建以ast方式实现Instrumenter接口的ast.instrumenter实例
    ins = ast.New("github.com/bigwhite/instrument_trace", "trace", "Trace") 
    newSrc, err := ins.Instrument(file) // 向Go源文件所有函数注入Trace函数
    if err != nil {
        panic(err)
    }

    if newSrc == nil {
        // add nothing to the source file. no change
        fmt.Printf("no trace added for %s\n", file)
        return
    }

    if !wrote {
        fmt.Println(string(newSrc))  // 将生成的新代码内容输出到stdout上
        return
    }

    // 将生成的新代码内容写回原Go源文件
    if err = ioutil.WriteFile(file, newSrc, 0666); err != nil {
        fmt.Printf("write %s error: %v\n", file, err)
        return
    }
    fmt.Printf("instrument trace for %s ok\n", file)
}
```

作为命令行工具，instrument 使用标准库的 flag 包实现对命令行参数（这里是 -w）的解析，通过 os.Args 获取待注入的 Go 源文件路径。在完成对命令行参数个数与值的校验后，instrument 程序声明了一个 instrumenter.Instrumenter 接口类型变量 ins，然后创建了一个实现了 Instrumenter 接口类型的 ast.instrumenter 类型的实例，并赋值给变量 ins。

instrumenter.Instrumenter 接口类型的声明放在了 instrumenter/instrumenter.go 中：

```go
type Instrumenter interface {
    Instrument(string) ([]byte, error)
}
```

这里我们看到，这个接口类型的方法列表中只有一个方法 Instrument，**这个方法接受一个 Go 源文件路径，返回注入了 Trace 函数的新源文件内容以及一个 error 类型值，作为错误状态标识。我们之所以要抽象出一个接口类型，考虑的就是注入 Trace 函数的实现方法不一，为后续的扩展做好预留。**

在这个例子中，我们默认提供了一种自动注入 Trace 函数的实现，那就是 ast.instrumenter，它注入 Trace 的实现原理是这样的：

![img](markdown%E5%9B%BE%E7%89%87/5be0efb3920c39cd9f252af0b26e7ca8.jpg)

从原理图中我们可以清楚地看到，在这一实现方案中，我们先将传入的 Go 源码转换为抽象语法树。因为 Go 语言是开源编程语言，所以它的抽象语法树的操作包也和语言一起开放给了 Go 开发人员，我们可以基于 Go 标准库以及Go 实验工具库提供的 ast 相关包，快速地构建基于 AST 的应用，**这里的 ast.instrumenter 就是一个应用 AST 的典型例子**。

一旦我们通过 ast 相关包解析 Go 源码得到相应的抽象语法树后，我们便可以操作这棵语法树，并按我们的逻辑在语法树中注入我们的 Trace 函数，最后我们再将修改后的抽象语法树转换为 Go 源码，就完成了整个自动注入的工作了。

下面是 ast.instrumenter 的 Instructment 方法的代码：

```go
// instrument_trace/instrumenter/ast/ast.go

func (a instrumenter) Instrument(filename string) ([]byte, error) {
    fset := token.NewFileSet()
    curAST, err := parser.ParseFile(fset, filename, nil, parser.ParseComments) // 解析Go源码，得到AST
    if err != nil {
        return nil, fmt.Errorf("error parsing %s: %w", filename, err)
    }

    if !hasFuncDecl(curAST) { // 如果整个源码都不包含函数声明，则无需注入操作，直接返回。
        return nil, nil
    }

    // 在AST上添加包导入语句
    astutil.AddImport(fset, curAST, a.traceImport)

    // 向AST上的所有函数注入Trace函数
    a.addDeferTraceIntoFuncDecls(curAST)

    buf := &bytes.Buffer{}
    err = format.Node(buf, fset, curAST) // 将修改后的AST转换回Go源码
    if err != nil {
        return nil, fmt.Errorf("error formatting new code: %w", err)
    }
    return buf.Bytes(), nil // 返回转换后的Go源码
}
```

通过代码，我们看到 Instrument 方法的基本步骤与上面原理图大同小异。

- go/scanner包提供词法分析功能，将源代码转换为一系列的token，以供go/parser使用
- go/parser包提供语法分析功能，将这些token转换为AST

Instrument 首先通过 go/paser 的 ParserFile 函数对传入的 Go 源文件中的源码进行解析，并得到对应的抽象语法树 AST，然后向 AST 中导入 Trace 函数所在的包，并向这个 AST 的所有函数声明注入 Trace 函数调用。

实际的注入操作发生在 instrumenter 的 addDeferTraceIntoFuncDecls 方法中，我们来看一下这个方法的实现：

```go
// instrument_trace/instrumenter/ast/ast.go

func (a instrumenter) addDeferTraceIntoFuncDecls(f *ast.File) {
    for _, decl := range f.Decls { // 遍历所有声明语句
        fd, ok := decl.(*ast.FuncDecl) // 类型断言：是否为函数声明
        if ok { 
            // 如果是函数声明，则注入跟踪设施
            a.addDeferStmt(fd)
        }
    }
}
```

这个方法的逻辑十分清晰，就是遍历语法树上所有声明语句，如果是函数声明，就调用 instrumenter 的 addDeferStmt 方法进行注入，如果不是，就直接返回。addDeferStmt 方法的实现如下：

```go
// instrument_trace/instrumenter/ast/ast.go

func (a instrumenter) addDeferTraceIntoFuncDecls(f *ast.File) {
    for _, decl := range f.Decls { // 遍历所有声明语句
        fd, ok := decl.(*ast.FuncDecl) // 类型断言：是否为函数声明
        if ok { 
            // 如果是函数声明，则注入跟踪设施
            a.addDeferStmt(fd)
        }
    }
}
```

这个方法的逻辑十分清晰，就是遍历语法树上所有声明语句，如果是函数声明，就调用 instrumenter 的 addDeferStmt 方法进行注入，如果不是，就直接返回。addDeferStmt 方法的实现如下：

```go
// instrument_trace/instrumenter/ast/ast.go

func (a instrumenter) addDeferStmt(fd *ast.FuncDecl) (added bool) {
    stmts := fd.Body.List

    // 判断"defer trace.Trace()()"语句是否已经存在
    for _, stmt := range stmts {
        ds, ok := stmt.(*ast.DeferStmt)
        if !ok {
            // 如果不是defer语句，则继续for循环
            continue
        }

        // 如果是defer语句，则要进一步判断是否是defer trace.Trace()()
        ce, ok := ds.Call.Fun.(*ast.CallExpr)
        if !ok {
            continue
        }

        se, ok := ce.Fun.(*ast.SelectorExpr)
        if !ok {
            continue
        }

        x, ok := se.X.(*ast.Ident)
        if !ok {
            continue
        }
        if (x.Name == a.tracePkg) && (se.Sel.Name == a.traceFunc) {
            // defer trace.Trace()()已存在，返回
            return false
        }
    }

    // 没有找到"defer trace.Trace()()"，注入一个新的跟踪语句
    // 在AST上构造一个defer trace.Trace()()
    ds := &ast.DeferStmt{
        Call: &ast.CallExpr{
            Fun: &ast.CallExpr{
                Fun: &ast.SelectorExpr{
                    X: &ast.Ident{
                        Name: a.tracePkg,
                    },
                    Sel: &ast.Ident{
                        Name: a.traceFunc,
                    },
                },
            },
        },
    }

    newList := make([]ast.Stmt, len(stmts)+1)
    copy(newList[1:], stmts)
    newList[0] = ds // 注入新构造的defer语句
    fd.Body.List = newList
    return true
}
```

虽然 addDeferStmt 函数体略长，但逻辑也很清晰，就是先判断函数是否已经注入了 Trace，如果有，则略过；如果没有，就构造一个 Trace 语句节点，并将它插入到 AST 中。Instrument 的最后一步就是将注入 Trace 后的 AST 重新转换为 Go 代码，这就是我们期望得到的带有 Trace 特性的 Go 代码了。



> ### **利用instrument工具注入跟踪代码**

这里，我在 instrument_trace 项目的 examples 目录下建立了一个名为 demo 的项目，我们就来看看如何使用 instrument 工具为 demo 项目下的 demo.go 文件自动注入跟踪设施。demo.go 文件内容很简单：

```go
// instrument_trace/examples/demo/demo.go

package main

func foo() {
    bar()
}

func bar() {
}

func main() {
    foo()
}
```

我们首先构建一下 instrument_trace 下的 instrument 工具：

```go
$cd instrument_trace
$go build instrument_trace/cmd/instrument

```

接下来，我们使用 instrument 工具向 examples/demo/demo.go 源文件中的函数自动注入跟踪设施：

```go
$instrument -w  examples/demo/demo.go
```

注入后的 demo.go 文件变为了下面这个样子：

```go
package main
  
import "github.com/bigwhite/instrument_trace"

func foo() {
    defer trace.Trace()()
    bar()
}

func bar() {
    defer trace.Trace()()
}

func main() {
    defer trace.Trace()()
    foo()
}
```

此时，如果我们再对已注入 Trace 函数的 demo.go 执行一次 instrument 命令，由于 instrument 会判断 demo.go 各个函数已经注入了 Trace，demo.go 的内容将保持不变。

由于instrument_trace 并没有真正上传到 github.com 上，所以如果你要运行 demo.go，我们可以为它配置一个下面这样的 go.mod：

```go
module demo

go 1.17

require instrument_trace v1.0.0

replace instrument_trace v1.0.0 => ../../
```

```go
$go run demo.go
g[00001]:    ->main.main
g[00001]:        ->main.foo
g[00001]:            ->main.bar
g[00001]:            <-main.bar
g[00001]:        <-main.foo
g[00001]:    <-main.main
```



#### 1.6 小结及改进

关于这个实战项目，有两点注意事项要和你交代清楚：

- 第一，在代码中注入函数调用跟踪代码**仅适用于日常调试代码和阅读理解代码时使用**，被注入了跟踪设施的代码是不适合上生产环境的；
- 第二，我在这里使用到了 Go 核心团队不推荐使用的 Goroutine id，这也是由这个实战项目的性质所决定的。如果你的代码是上生产，我建议还是尽量听从 Go 核心团队的建议，**不要依赖 Goroutine ID。**

改进：

- 通过 instrument 命令行工具对 Go 源文件进行注入后，defer trace.Trace()() 就会成为 Go 源码的一部分被编译进最终的可执行文件中。我们在小结中也提到了，开启了 Trace 的代码不要上生产环境，这样我们在构建上生产的应用之前需要手工删掉这些 Trace 代码，操作起来十分繁琐易错。所以，这里我想请你为 Trace 增加一个开关功能，有了这个开关后，日常开发调试过程中编译出的程序中的 Trace 是起作用的，但为生产环境编译出的可执行程序中虽然也包含 Trace，但 Trace 不会真正起作用（提示：使用 build tag）



### 2 实现一个轻量级的线程池



#### 2.1 workerpool 的实现原理

workerpool 的工作逻辑通常都很简单，所以即便是用于生产环境的 workerpool 实现，代码规模也都在千行左右。

当然，workerpool 有很多种实现方式，这里为了更好地演示 Go 并发模型的应用模式，以及并发原语间的协作，我们采用完全基于 channel+select 的实现方案，不使用其他数据结构，也不使用 sync 包提供的各种同步结构，比如 Mutex、RWMutex，以及 Cond 等。

**workerpool 的实现主要分为三个部分：**

- pool 的创建与销毁；

- pool 中 worker（Goroutine）的管理；

- task 的提交与调度。

  

后两部分是 pool 的“精髓”所在，这两部分的原理我也用一张图表示了出来：

![img](markdown%E5%9B%BE%E7%89%87/d48ba3a204ca6e8961a4425573afa0fd.jpg)

**pool 对 worker 的管理：**

capacity 是 pool 的一个属性，代表整个 pool 中 worker 的最大容量。我们使用一个带缓冲的 channel：active，作为 worker 的“计数器”，这种 channel 使用模式就是上面讲到过的

当 active channel 可写时，我们就创建一个 worker，用于处理用户通过 Schedule 函数提交的待处理的请求。当 active channel 满了的时候，pool 就会停止 worker 的创建，直到某个 worker 因故退出，active channel 又空出一个位置时，pool 才会创建新的 worker 填补那个空位。

这张图里，我们把用户要提交给 workerpool 执行的请求抽象为一个 Task。Task 的提交与调度也很简单：Task 通过 Schedule 函数提交到一个 task channel 中，已经创建的 worker 将从这个 task channel 中读取 task 并执行。



#### 2.2 workerpool的一个最小可行实现

我们先建立 workerpool 目录作为实战项目的源码根目录，然后为这个项目创建 go module：

```go
$mkdir workerpool1
$cd workerpool1
```

我们创建 pool.go 作为 workpool 包的主要源码文件。在这个源码文件中，我们定义了 Pool 结构体类型，这个类型的实例代表一个 workerpool：

```go
type Pool struct {
    capacity int         // workerpool大小

    active chan struct{} // 对应上图中的active channel
    tasks  chan Task     // 对应上图中的task channel

    wg   sync.WaitGroup  // 用于在pool销毁时等待所有worker退出
    quit chan struct{}   // 用于通知各个worker退出的信号channel
}
```

workerpool 包对外主要提供三个 API，它们分别是：

- workerpool.New：用于创建一个 pool 类型实例，并将 pool 池的 worker 管理机制运行起来；
- workerpool.Free：用于销毁一个 pool 池，停掉所有 pool 池中的 worker；
- pool.Schedule：这是 pool 类型的一个导出方法，workerpool 包的用户通过该方法向 pool 池提交待执行的任务（Task）。

先来看看 workerpool.New 是如何创建一个 pool 实例的：

```go
func New(capacity int) *Pool {
    if capacity <= 0 {
        capacity = defaultCapacity
    }
    if capacity > maxCapacity { 
        capacity = maxCapacity
    } 

    p := &Pool{
        capacity: capacity,
        tasks:    make(chan Task),
        quit:     make(chan struct{}),
        active:   make(chan struct{}, capacity),
    }

    fmt.Printf("workerpool start\n")

    go p.run()

    return p
}
```

New 函数接受一个参数 capacity 用于指定 workerpool 池的容量，这个参数用于控制 workerpool 最多只能有 capacity 个 worker，共同处理用户提交的任务请求。函数开始处有一个对 capacity 参数的“防御性”校验，当用户传入不合理的值时，函数 New 会将它纠正为合理的值。

Pool 类型实例变量 p 完成初始化后，我们创建了一个新的 Goroutine，用于对 workerpool 进行管理，这个 Goroutine 执行的是 Pool 类型的 run 方法：

```go
func (p *Pool) run() { 
    idx := 0 

    for { 
        select { 
        case <-p.quit:
            return
        case p.active <- struct{}{}:
            // create a new worker
            idx++
            p.newWorker(idx)
        } 
    } 
}
```

run 方法内是一个无限循环，循环体中使用 select 监视 Pool 类型实例的两个 channel：quit 和 active。这种在 for 中使用 select 监视多个 channel 的实现，在 Go 代码中十分常见，是一种惯用法。

当接收到来自 quit channel 的退出“信号”时，这个 Goroutine 就会结束运行。而当 active channel 可写时，run 方法就会创建一个新的 worker Goroutine。 此外，为了方便在程序中区分各个 worker 输出的日志，我这里将一个从 1 开始的变量 idx 作为 worker 的编号，并把它以参数的形式传给创建 worker 的方法。

我们再将创建新的 worker goroutine 的职责，封装到一个名为 newWorker 的方法中：

```go
func (p *Pool) newWorker(i int) {
    p.wg.Add(1)
    go func() {
        defer func() {
            if err := recover(); err != nil {
                fmt.Printf("worker[%03d]: recover panic[%s] and exit\n", i, err)
                <-p.active
            }
            p.wg.Done()
        }()

        fmt.Printf("worker[%03d]: start\n", i)

        for {
            select {
            case <-p.quit:
                fmt.Printf("worker[%03d]: exit\n", i)
                <-p.active
                return
            case t := <-p.tasks:
                fmt.Printf("worker[%03d]: receive a task\n", i)
                t()
            }
        }
    }()
}
```

我们看到，在创建一个新的 worker goroutine 之前，newWorker 方法会先调用 p.wg.Add 方法将 WaitGroup 的等待计数加一。由于每个 worker 运行于一个独立的 Goroutine 中，newWorker 方法通过 go 关键字创建了一个新的 Goroutine 作为 worker。

新 worker 的核心，依然是一个基于 for-select 模式的循环语句，在循环体中，新 worker 通过 select 监视 quit 和 tasks 两个 channel。和前面的 run 方法一样，当接收到来自 quit channel 的退出“信号”时，这个 worker 就会结束运行。tasks channel 中放置的是用户通过 Schedule 方法提交的请求，新 worker 会从这个 channel 中获取最新的 Task 并运行这个 Task。

Task 是一个对用户提交的请求的抽象，它的本质就是一个函数类型:

```go
type Task func()
```

这样，用户通过 Schedule 方法实际上提交的是一个函数类型的实例。

在新 worker 中，为了防止用户提交的 task 抛出 panic，进而导致整个 workerpool 受到影响，我们在 worker 代码的开始处，使用了 defer+recover 对 panic 进行捕捉，捕捉后 worker 也是要退出的，于是我们还通过<-p.active更新了 worker 计数器。并且一旦 worker goroutine 退出，p.wg.Done 也需要被调用，这样可以减少 WaitGroup 的 Goroutine 等待数量。

我们再来看 workerpool 提供给用户提交请求的导出方法 Schedule：

```go
var ErrWorkerPoolFreed    = errors.New("workerpool freed")       // workerpool已终止运行

func (p *Pool) Schedule(t Task) error {
    select {
    case <-p.quit:
        return ErrWorkerPoolFreed
    case p.tasks <- t:
        return nil
    }
}
```

Schedule 方法的核心逻辑，是将传入的 Task 实例发送到 workerpool 的 tasks channel 中。但考虑到现在 workerpool 已经被销毁的状态，我们这里通过一个 select，检视 quit channel 是否有“信号”可读，如果有，就返回一个哨兵错误 ErrWorkerPoolFreed。如果没有，一旦 p.tasks 可写，提交的 Task 就会被写入 tasks channel，以供 pool 中的 worker 处理。

这里要注意的是，这里的 Pool 结构体中的 **tasks 是一个无缓冲的 channel，如果 pool 中 worker 数量已达上限**，而且 worker 都在处理 task 的状态，那么 Schedule 方法就会阻塞，直到有 worker 变为 idle 状态来读取 tasks channel，schedule 的调用阻塞才会解除。

至此，workerpool 的最小可行实现的主要逻辑都实现完了。我们来验证一下它是否能按照我们的预期逻辑运行。

现在我们建立一个使用 workerpool 的项目 demo1：

```go
$mkdir demo1
$cd demo1
$go mod init demo1
```

由于我们要引用本地的 module，所以我们需要手工修改一下 demo1 的 go.mod 文件，并利用 **replace 指示符将 demo1 对 workerpool （这里demo1和workerpool的概念上都是module，你的go mod里module叫什么这里就对应什么）的引用指向本地 workerpool1 路径**：

```go
module demo1

go 1.17

require github.com/bigwhite/workerpool v1.0.0

replace github.com/bigwhite/workerpool v1.0.0 => ../workerpool1
```

然后创建 demo1 的 main.go 文件，源码如下：

```go
package main
  
import (
    "time"
    "github.com/bigwhite/workerpool"
)

func main() {
    p := workerpool.New(5)

    for i := 0; i < 10; i++ {
        err := p.Schedule(func() {
            time.Sleep(time.Second * 3)
        })
        if err != nil {
            println("task: ", i, "err:", err)
        }
    }

    p.Free()
}
```

这个示例程序创建了一个 capacity 为 5 的 workerpool 实例，并连续向这个 workerpool 提交了 10 个 task，每个 task 的逻辑很简单，只是 Sleep 3 秒后就退出。main 函数在提交完任务后，调用 workerpool 的 Free 方法销毁 pool，pool 会等待所有 worker 执行完 task 后再退出。

```go
workerpool start
worker[005]: start
worker[005]: receive a task
worker[003]: start
worker[003]: receive a task
worker[004]: start
worker[004]: receive a task
worker[001]: start
worker[002]: start
worker[001]: receive a task
worker[002]: receive a task
worker[004]: receive a task
worker[005]: receive a task
worker[003]: receive a task
worker[002]: receive a task
worker[001]: receive a task
worker[001]: exit
worker[005]: exit
worker[002]: exit
worker[003]: exit
worker[004]: exit
workerpool freed
```

不过，目前的 workerpool 实现好比“铁板一块”，虽然我们可以通过 capacity 参数可以指定 workerpool 容量，但我们无法对 workerpool 的行为进行定制。

比如当 workerpool 中的 worker 数量已达上限，而且 worker 都在处理 task 时，用户调用 Schedule 方法将阻塞，如果用户不想阻塞在这里，以我们目前的实现是做不到的。

那我们可以怎么改进呢？我们可以尝试在上面实现的基础上，为 workerpool 添加功能选项（functional option）机制。

#### 2.3 添加功能选项机制

**功能选项机制，可以让某个包的用户可以根据自己的需求，通过设置不同功能选项来定制包的行为。**Go 语言中实现功能选项机制有多种方法，但 Go 社区目前使用最为广泛的一个方案，是 Go 语言之父 Rob Pike 在 2014 年在博文《自引用函数与选项设计》中论述的一种，这种方案也被后人称为“功能选项（functional option）”方案。

首先，我们将 workerpool1 目录拷贝一份形成 workerpool2 目录，我们将在这个目录下为 workerpool 包添加功能选项机制。

然后，我们在 workerpool2 目录下创建 option.go 文件，在这个文件中，我们定义用于代表功能选项的类型 Option：

```go
type Option func(*Pool)
```

我们看到，这个 Option 实质是一个接受 *Pool 类型参数的函数类型。那么如何运用这个 Option 类型呢？别急，马上你就会知道。现在我们先要做的是，明确给 workerpool 添加什么功能选项。这里我们为 workerpool 添加两个功能选项：Schedule 调用是否阻塞，以及是否预创建所有的 worker。

为了支持这两个功能选项，我们需要在 Pool 类型中增加两个 bool 类型的字段，字段的具体含义，我也在代码中注释了：

```g
type Pool struct {
    ... ...
    preAlloc bool // 是否在创建pool的时候就预创建workers，默认值为：false

    // 当pool满的情况下，新的Schedule调用是否阻塞当前goroutine。默认值：true
    // 如果block = false，则Schedule返回ErrNoWorkerAvailInPool
    block  bool
    ... ...
}
```

针对这两个字段，我们在 option.go 中添加两个功能选项，WithBlock 与 WithPreAllocWorkers：

```go'
func WithBlock(block bool) Option {
    return func(p *Pool) {
        p.block = block
    }
}

func WithPreAllocWorkers(preAlloc bool) Option {
    return func(p *Pool) {
        p.preAlloc = preAlloc
    }
}
```

我们看到，这两个功能选项实质上是两个返回闭包函数的函数。

为了支持将这两个 Option 传给 workerpool，我们还需要改造一下 workerpool 包的 New 函数，改造后的 New 函数代码如下：

```go
func New(capacity int, opts ...Option) *Pool {
    ... ...
    for _, opt := range opts {
        opt(p)
    }

    fmt.Printf("workerpool start(preAlloc=%t)\n", p.preAlloc)

    if p.preAlloc {
        // create all goroutines and send into works channel
        for i := 0; i < p.capacity; i++ {
            p.newWorker(i + 1)
            p.active <- struct{}{}
        }
    }

    go p.run()

    return p
}
```

新版 New 函数除了接受 capacity 参数之外，还在它的参数列表中增加了一个类型为 Option 的可变长参数 opts。在 New 函数体中，我们通过一个 for 循环，将传入的 Option 运用到 Pool 类型的实例上。

新版 New 函数还会根据 preAlloc 的值来判断是否预创建所有的 worker，如果需要，就调用 newWorker 方法把所有 worker 都创建出来。newWorker 的实现与上一版代码并没有什么差异，这里就不再详说了。

但由于 preAlloc 选项的加入，Pool 的 run 方法的实现有了变化，我们来看一下：

```go
 func (p *Pool) run() {
     idx := len(p.active)
 
     if !p.preAlloc {
     loop:
         for t := range p.tasks {
             p.returnTask(t)
             select {
             case <-p.quit:
                 return
             case p.active <- struct{}{}:
                 idx++
                 p.newWorker(idx)
             default:
                 break loop
             }
         }
     }
 
     for {
         select {
         case <-p.quit:
             return
         case p.active <- struct{}{}:
             // create a new worker
             idx++
             p.newWorker(idx)
         }
     }
 }
```

新版 run 方法在 preAlloc=false 时，会根据 tasks channel 的情况在适合的时候创建 worker（第 4 行~ 第 18 行)，直到 active channel 写满，才会进入到和第一版代码一样的调度逻辑中（第 20 行~ 第 29 行）。而且，提供给用户的 Schedule 函数也因 WithBlock 选项，有了一些变化：

```go
 func (p *Pool) Schedule(t Task) error {
     select {
     case <-p.quit:
         return ErrWorkerPoolFreed
     case p.tasks <- t:
         return nil
     default:
         if p.block {
             p.tasks <- t
             return nil
         }
         return ErrNoIdleWorkerInPool
     }
 }
```

Schedule 在 tasks chanel 无法写入的情况下，进入 default 分支。在 default 分支中，Schedule 根据 block 字段的值，决定究竟是继续阻塞在 tasks channel 上，还是返回 ErrNoIdleWorkerInPool 错误。和第一版 worker 代码一样，我们也来验证一下新增的功能选项是否好用。我们建立一个使用新版 workerpool 的项目 demo2，demo2 的 go.mod 与 demo1 的 go.mod 相似：

```go
module demo2

go 1.17

require github.com/bigwhite/workerpool v1.0.0

replace github.com/bigwhite/workerpool v1.0.0 => ../workerpool2
```

demo2的main.go文件如下：

```go
package main
  
import (
    "fmt"
    "time"

    "github.com/bigwhite/workerpool"
)

func main() {
    p := workerpool.New(5, workerpool.WithPreAllocWorkers(false), workerpool.WithBlock(false))

    time.Sleep(time.Second * 2)
    for i := 0; i < 10; i++ {
        err := p.Schedule(func() {
            time.Sleep(time.Second * 3)
        })
        if err != nil {
            fmt.Printf("task[%d]: error: %s\n", i, err.Error())
        }
    }

    p.Free()
}
```

在 demo2 中，我们使用 workerpool 包提供的功能选项，设置了我们期望的 workerpool 的运作行为，包括不要预创建 worker，以及不要阻塞 Schedule 调用。

考虑到 Goroutine 调度的次序的不确定性，这里我在创建 workerpool 与真正开始调用 Schedule 方法之间，做了一个 Sleep，尽量减少 Schedule 都返回失败的频率（但这仍然无法保证这种情况不会发生）。

运行 demo2，我们会得到这个结果：

```go
workerpool start(preAlloc=false)
task[1]: error: no idle worker in pool
worker[001]: start
task[2]: error: no idle worker in pool
task[4]: error: no idle worker in pool
task[5]: error: no idle worker in pool
task[6]: error: no idle worker in pool
task[7]: error: no idle worker in pool
task[8]: error: no idle worker in pool
task[9]: error: no idle worker in pool
worker[001]: receive a task
worker[002]: start
worker[002]: exit
worker[001]: receive a task
worker[001]: exit
workerpool freed(preAlloc=false)
```

不过，由于 Goroutine 调度的不确定性，这个结果仅仅是很多种结果的一种。我们看到，仅仅 001 这个 worker 收到了 task，其余的 worker 都因为 worker 尚未创建完毕，而返回了错误，而不是像 demo1 那样阻塞在 Schedule 调用上。



### 3 实现一个TCP服务

#### 3.1 什么是网络编程

什么是网络编程呢？网络编程的范围很大，因为我们熟知的网络是分层的，OSI 规定了七层参考模型，而实际上我们使用的主流网络模型实现，是 TCP/IP 模型，它只有四层：

![img](markdown%E5%9B%BE%E7%89%87/1339ef73cbd62791byy28821fd5ed926.jpg)通常来说，我们更多关注 OSI 网络模型中的传输层（四层）与应用层（七层），也就是 TCP/IP 网络模型中的最上面两层。

TCP/IP 网络模型，实现了两种传输层协议：TCP 和 UDP。TCP 是面向连接的流协议，为通信的两端提供稳定可靠的数据传输服务；而 UDP 则提供了一种无需建立连接就可以发送数据包的方法。两种协议各有擅长的应用场景。

我们日常开发中使用最多的是 TCP 协议。基于 TCP 协议，我们实现了各种各样的满足用户需求的应用层协议。比如，我们常用的 HTTP 协议就是应用层协议的一种，而且是使用得最广泛的一种。而基于 HTTP 的 Web 编程就是一种针对应用层的网络编程。我们还可以**基于传输层暴露给开发者的编程接口，实现应用层的自定义应用协议。**

这个传输层暴露给开发者的编程接口，究竟是什么呢？目前各大主流操作系统平台中，最常用的传输层暴露给用户的网络编程接口，就是套接字（socket)。**直接基于 socket 编程实现应用层通信业务，也是最常见的一种网络编程形式。**所以，这一节课，我们就使用一个基于 socket 网络编程的例子，我们先来看看这个例子对应的实际问题是什么。

#### 3.2 问题描述

我们面临的实际问题是这样的：**实现一个基于 TCP 的自定义应用层协议的通信服务端**。仅仅这一句话，你可能还不是很清楚，我们展开说明一下。我们的输入，是一个基于传输层自定义的应用层协议规范。由于 TCP 是面向连接的流协议传输机制，数据流本身没有明显的边界，这样定义协议时，就需要自行定义确定边界的方法，因此，基于 TCP 的自定义应用层协议通常有两种常见的定义模式：

- 二进制模式：采用长度字段标识独立数据包的边界。采用这种方式定义的常见协议包括 MQTT（物联网最常用的应用层协议之一）、SMPP（短信网关点对点接口协议）等；
- 文本模式：采用特定分隔符标识流中的数据包的边界，常见的包括 HTTP 协议等。

相比之下，二进制模式要比文本模式编码更紧凑也更高效，所以我们这个问题中的自定义协议也采用了二进制模式，协议规范内容如下图：![img](markdown%E5%9B%BE%E7%89%87/70b43197100a790f3a78db50997c1d21.jpg)



关于协议内容的分析，我们放到设计与实现的那一讲中再细说，这里我们再看一下使用这个协议的通信两端的通信流程：

![img](markdown%E5%9B%BE%E7%89%87/bbf6078436ff91207cf6232ce7a66b1e.jpg)

我们看到，这是一个典型的“请求 / 响应”通信模型。连接由客户端发起，建立连接后，客户端发起请求，服务端收到请求后处理并返回响应，就这样一个请求一个响应的进行下去，直到客户端主动断开连接为止。**而我们的任务，就是实现支持这个协议通信的服务端。**



#### 3.3 TCP Socket 编程模型

TCP Socket 诞生以来，它的编程模型，也就是网络 I/O 模型已几经演化。网络 I/O 模型定义的是应用线程与操作系统内核之间的交互行为模式。我们通常用**阻塞（Blocking）/非阻塞（Non-Blocking）**来描述网络 I/O 模型。

**阻塞 / 非阻塞，是以内核是否等数据全部就绪后，才返回（给发起系统调用的应用线程）来区分的。**如果内核一直等到全部数据就绪才返回，这种行为模式就称为**阻塞**。如果内核查看数据就绪状态后，即便没有就绪也立即返回错误（给发起系统调用的应用线程），那么这种行为模式则称为**非阻塞**。

常用的网络 I/O 模型包括下面这几种：

- 阻塞 I/O(Blocking I/O)

阻塞 I/O 是最常用的模型，这个模型下应用线程与内核之间的交互行为模式是这样的：![img](markdown%E5%9B%BE%E7%89%87/66e154f76d647a51b45fcfddf4697c70.jpg)

我们看到，在阻塞 I/O 模型下，当用户空间应用线程，向操作系统内核发起 I/O 请求后（一般为操作系统提供的 I/O 系列系统调用），内核会尝试执行这个 I/O 操作，并等所有数据就绪后，将数据从内核空间拷贝到用户空间，最后系统调用从内核空间返回。而在这个期间内，用户空间应用线程将阻塞在这个 I/O 系统调用上，无法进行后续处理，只能等待。

因此，在这样的模型下，一个线程仅能处理一个网络连接上的数据通信。即便连接上没有数据，线程也只能阻塞在对 Socket 的读操作上（以等待对端的数据）。虽然这个模型对应用整体来说是低效的，但对开发人员来说，这个模型却是最容易实现和使用的，所以，各大平台在默认情况下都将 Socket 设置为阻塞的。

- 非阻塞 I/O（Non-Blocking I/O）

非阻塞 I/O 模型下，应用线程与内核之间的交互行为模式是这样的：

![img](markdown%E5%9B%BE%E7%89%87/4c5e3980f756e03b9ca023185b91b5b3.jpg)



和阻塞 I/O 模型正相反，**在非阻塞模型下**，当用户空间线程向操作系统内核发起 I/O 请求后，内核会执行这个 I/O 操作，如果这个时候数据尚未就绪，就会立即将“未就绪”的状态以错误码形式（比如：EAGAIN/EWOULDBLOCK），返回给这次 I/O 系统调用的发起者。而后者就会根据系统调用的返回状态来决定下一步该怎么做。

在非阻塞模型下，位于用户空间的 I/O 请求发起者通常会通过轮询的方式，去一次次发起 I/O 请求，直到读到所需的数据为止。不过，这样的轮询是对 CPU 计算资源的极大浪费，因此，非阻塞 I/O 模型单独应用于实际生产的比例并不高。

- I/O 多路复用（I/O Multiplexing）

为了避免非阻塞 I/O 模型轮询对计算资源的浪费，同时也考虑到阻塞 I/O 模型的低效，开发人员首选的网络 I/O 模型，逐渐变成了建立在内核提供的多路复用函数 select/poll 等（以及性能更好的 epoll 等函数）基础上的 **I/O 多路复用模型**。这个模型下，应用线程与内核之间的交互行为模式如下图：![img](markdown%E5%9B%BE%E7%89%87/9389908c507b5efea24bf961b1d594b9.jpg)

从图中我们看到，在这种模型下，应用线程首先将需要进行 I/O 操作的 Socket，都添加到多路复用函数中（这里以 select 为例），然后阻塞，等待 select 系统调用返回。当内核发现有数据到达时，对应的 Socket 具备了通信条件，这时 select 函数返回。然后用户线程会针对这个 Socket 再次发起网络 I/O 请求，比如一个 read 操作。由于数据已就绪，这次网络 I/O 操作将得到预期的操作结果。我们看到，相比于阻塞模型一个线程只能处理一个 Socket 的低效，I/O 多路复用模型中，一个应用线程可以同时处理多个 Socket。同时，I/O 多路复用模型由内核实现可读 / 可写事件的通知，避免了非阻塞模型中轮询，带来的 CPU 计算资源浪费的问题。目前，主流网络服务器采用的都是“I/O 多路复用”模型，有的也结合了多线程。不过，I/O 多路复用模型在支持更多连接、提升 I/O 操作效率的同时，也给使用者带来了不小的复杂度，以至于后面出现了许多高性能的 I/O 多路复用框架，比如：libevent、libev、libuv等，以帮助开发者简化开发复杂性，降低心智负担。

那么，在这三种 socket 编程模型中，Go 语言使用的是哪一种呢？我们继续往下看。

#### 3.4 Go语言socket编程模型

Go 语言设计者考虑得更多的是 Gopher 的开发体验。前面我们也说过，阻塞 I/O 模型是对开发人员最友好的，也是心智负担最低的模型，而 I/O 多路复用的这种通过回调割裂执行流的模型，对开发人员来说还是过于复杂了，于是 **Go 选择了为开发人员提供阻塞 I/O 模型**，Gopher 只需在 Goroutine 中以最简单、最易用的“阻塞 I/O 模型”的方式，进行 Socket 操作就可以了。

再加上，Go 没有使用基于线程的并发模型，而是使用了开销更小的 Goroutine 作为基本执行单元，这让每个 Goroutine 处理一个 TCP 连接成为可能，并且在高并发下依旧表现出色。

不过，网络 I/O 操作都是系统调用，Goroutine 执行 I/O 操作的话，一旦阻塞在系统调用上，就会导致 M 也被阻塞，为了解决这个问题，Go 设计者将这个“复杂性”隐藏在 Go 运行时中，他们在运行时中实现了网络轮询器（netpoller)，netpoller 的作用，就是只阻塞执行网络 I/O 操作的 Goroutine，但不阻塞执行 Goroutine 的线程（也就是 M）。

这样一来，对于 Go 程序的用户层（相对于 Go 运行时层）来说，它眼中看到的 goroutine 采用了“阻塞 I/O 模型”进行网络 I/O 操作，Socket 都是“阻塞”的。

但实际上，这样的“假象”，是通过 Go 运行时中的 netpoller I/O 多路复用机制，“模拟”出来的，对应的、真实的底层操作系统 Socket，实际上是非阻塞的。只是运行时拦截了针对底层 Socket 的系统调用返回的错误码，并通过 netpoller 和 Goroutine 调度，让 Goroutine“阻塞”在用户层所看到的 Socket 描述符上。

比如：当用户层针对某个 Socket 描述符发起read操作时，如果这个 Socket 对应的连接上还没有数据，运行时就会将这个 Socket 描述符加入到 netpoller 中监听，同时发起此次读操作的 Goroutine 会被挂起。直到 Go 运行时收到这个 Socket 数据可读的通知，Go 运行时才会重新唤醒等待在这个 Socket 上准备读数据的那个 Goroutine。而这个过程，从 Goroutine 的视角来看，就像是 read 操作一直阻塞在那个Socket 描述符上一样。而且，Go 语言在网络轮询器（netpoller）中采用了 I/O 多路复用的模型。考虑到最常见的多路复用系统调用 select 有比较多的限制，比如：监听 Socket 的数量有上限（1024）、时间复杂度高，等等，Go 运行时选择了在不同操作系统上，使用操作系统各自实现的高性能多路复用函数，比如：Linux 上的 epoll、Windows 上的 iocp、FreeBSD/MacOS 上的 kqueue、Solaris 上的 event port 等，这样可以最大程度提高 netpoller 的调度和执行性能。



#### 3.5 基于socket的网络I/O操作

> ### socket 监听（listen）与接收连接（accept）

socket 编程的核心在于服务端，而服务端有着自己一套相对固定的套路：Listen+Accept。在这套固定套路的基础上，我们的服务端程序通常采用一个 Goroutine 处理一个连接，它的大致结构如下：

```go
 func handleConn(c net.Conn) {
     defer c.Close()
     for {
         // read from the connection
         // ... ...
         // write to the connection
         //... ...
     }
 }
 
 func main() {
     l, err := net.Listen("tcp", ":8888")
     if err != nil {
         fmt.Println("listen error:", err)
         return
     }
 
     for {
         c, err := l.Accept()
         if err != nil {
             fmt.Println("accept error:", err)
             break
         }
         // start a new goroutine to handle
         // the new connection.
         go handleConn(c)
     }
 }
```

在这个服务端程序中，我们在第 12 行使用了 net 包的 Listen 函数绑定（bind）服务器端口 8888，并将它转换为监听状态，Listen 返回成功后，这个服务会进入一个循环，并调用 net.Listener 的 Accept 方法接收新客户端连接。

在没有新连接的时候，这个服务会阻塞在 Accept 调用上，直到有客户端连接上来，Accept 方法将返回一个 net.Conn 实例。通过这个 net.Conn，我们可以和新连上的客户端进行通信。这个服务程序启动了一个新 Goroutine，并将 net.Conn 传给这个 Goroutine，这样这个 Goroutine 就专职负责处理与这个客户端的通信了。

而 net.Listen 函数很少报错，除非是监听的端口已经被占用，那样程序将输出类似这样的错误：

```go
bind: address already in use
```

当服务程序启动成功后，我们可以通过 netstat 命令，查看端口的监听情况：

```go
$netstat -an|grep 8888    
tcp46       0      0  *.8888                 *.*                    LISTEN     
```



> ### 向服务端建立 TCP 连接

一旦服务端按照上面的Listen + Accept结构成功启动，客户端便可以使用net.Dial或net.DialTimeout向服务端发起连接建立的请求：

```go
conn, err := net.Dial("tcp", "localhost:8888")
conn, err := net.DialTimeout("tcp", "localhost:8888", 2 * time.Second)
```

Dial 函数向服务端发起 TCP 连接，这个函数会一直阻塞，直到连接成功或失败后，才会返回。而 DialTimeout 带有超时机制，如果连接耗时大于超时时间，这个函数会返回超时错误。 对于客户端来说，连接的建立还可能会遇到几种特殊情形。

**第一种情况：网络不可达或对方服务未启动。**

如果传给Dial的服务端地址是网络不可达的，或者服务地址中端口对应的服务并没有启动，端口未被监听（Listen），Dial几乎会立即返回类似这样的错误：

```go
dial error: dial tcp :8888: getsockopt: connection refused
```

**第二种情况：对方服务的 listen backlog 队列满。**

当对方服务器很忙，瞬间有大量客户端尝试向服务端建立连接时，服务端可能会出现 listen backlog 队列满，接收连接（accept）不及时的情况，这就会导致客户端的Dial调用阻塞，直到服务端进行一次 accept，从 backlog 队列中腾出一个槽位，客户端的 Dial 才会返回成功。而且，不同操作系统下 backlog 队列的长度是不同的，在 macOS 下，这个默认值如下：

```go
$sysctl -a|grep kern.ipc.somaxconn
kern.ipc.somaxconn: 128
```

**第三种情况：若网络延迟较大，Dial 将阻塞并超时。**

如果网络延迟较大，TCP 连接的建立过程（三次握手）将更加艰难坎坷，会经历各种丢包，时间消耗自然也会更长，这种情况下，Dial函数会阻塞。如果经过长时间阻塞后依旧无法建立连接，那么Dial也会返回类似getsockopt: operation timed out的错误。在连接建立阶段，多数情况下Dial是可以满足需求的，即便是阻塞一小会儿也没事。但对于那些需要有严格的连接时间限定的 Go 应用，如果一定时间内没能成功建立连接，程序可能会需要执行一段“错误”处理逻辑，所以，这种情况下，我们使用DialTimeout函数更适合。

> ### 全双工通信

一旦客户端调用 Dial 成功，我们就在客户端与服务端之间建立起了一条全双工的通信通道。通信双方通过各自获得的 Socket，可以在向对方发送数据包的同时，接收来自对方的数据包。下图展示了系统层面对这条全双工通信通道的实现原理：![img](markdown%E5%9B%BE%E7%89%87/370808418c01bf37967cdb505848e3b0.jpg)

任何一方的操作系统，都会为已建立的连接分配一个发送缓冲区和一个接收缓冲区。以客户端为例，客户端会通过成功连接服务端后得到的 conn（封装了底层的 socket）向服务端发送数据包。这些数据包会先进入到己方的发送缓冲区中，之后，这些数据会被操作系统内核通过网络设备和链路，发到服务端的接收缓冲区中，服务端程序再通过代表客户端连接的 conn 读取服务端接收缓冲区中的数据，并处理。反之，服务端发向客户端的数据包也是先后经过服务端的发送缓冲区、客户端的接收缓冲区，最终到达客户端的应用的。理解了这个通信原理，我们再理解下面的 Socket 操作就容易许多了。

> ### socket读操作

连接建立起来后，我们就要在连接上进行读写以完成业务逻辑。我们前面说过，Go 运行时隐藏了 I/O 多路复用的复杂性。Go 语言使用者只需采用 **Goroutine+ 阻塞 I/O 模型**，就可以满足大部分场景需求。Dial 连接成功后，会返回一个 net.Conn 接口类型的变量值，这个接口变量的底层类型为一个 *TCPConn：

```go
//$GOROOT/src/net/tcpsock.go
type TCPConn struct {
    conn
}
```

TCPConn 内嵌了一个非导出类型：conn（封装了底层的 socket），因此，TCPConn“继承”了conn类型的Read和Write方法，后续通过Dial函数返回值调用的Read和Write方法都是 net.conn 的方法，它们分别代表了对 socket 的读和写。

接下来，我们先来通过几个场景来总结一下 Go 中从 socket 读取数据的行为特点

**首先是 Socket 中无数据的场景。**连接建立后，如果客户端未发送数据，服务端会阻塞在 Socket 的读操作上，这和前面提到的“阻塞 I/O 模型”的行为模式是一致的。执行该这个操作的 Goroutine 也会被挂起。Go 运行时会监视这个 Socket，直到它有数据读事件，才会重新调度这个 Socket 对应的 Goroutine 完成读操作。

**第二种情况是 Socket 中有部分数据**。如果 Socket 中有部分数据就绪，且数据数量小于一次读操作期望读出的数据长度，那么读操作将会成功读出这部分数据，并返回，而不是等待期望长度数据全部读取后，再返回。举个例子，服务端创建一个长度为 10 的切片作为接收数据的缓冲区，等待 Read 操作将读取的数据放入切片。当客户端在已经建立成功的连接上，成功写入两个字节的数据（比如：hi）后，服务端的 Read 方法将成功读取数据，并返回n=2，err=nil，而不是等收满 10 个字节后才返回。

**第三种情况是 Socket 中有足够数据**。如果连接上有数据，且数据长度大于等于一次Read操作期望读出的数据长度，那么Read将会成功读出这部分数据，并返回。这个情景是最符合我们对Read的期待的了。我们以上面的例子为例，当客户端在已经建立成功的连接上，成功写入 15 个字节的数据后，服务端进行第一次Read时，会用连接上的数据将我们传入的切片缓冲区（长度为 10）填满后返回：n = 10, err = nil。这个时候，内核缓冲区中还剩 5 个字节数据，当服务端再次调用Read方法时，就会把剩余数据全部读出。

**最后一种情况是设置读操作超时**。有些场合，对 socket 的读操作的阻塞时间有严格限制的，但由于 Go 使用的是阻塞 I/O 模型，如果没有可读数据，Read 操作会一直阻塞在对 Socket 的读操作上。这时，我们可以通过 net.Conn 提供的 SetReadDeadline 方法，设置读操作的超时时间，当超时后仍然没有数据可读的情况下，Read 操作会解除阻塞并返回超时错误，这就给 Read 方法的调用者提供了进行其他业务处理逻辑的机会。SetReadDeadline 方法接受一个绝对时间作为超时的 deadline，一旦通过这个方法设置了某个 socket 的 Read deadline，那么无论后续的 Read 操作是否超时，只要我们不重新设置 Deadline，那么后面与这个 socket 有关的所有读操作，都会返回超时失败错误。

下面是结合 SetReadDeadline 设置的服务端一般处理逻辑：

```go
func handleConn(c net.Conn) {
    defer c.Close()
    for {
        // read from the connection
        var buf = make([]byte, 128)
        c.SetReadDeadline(time.Now().Add(time.Second))
        n, err := c.Read(buf)
        if err != nil {
            log.Printf("conn read %d bytes,  error: %s", n, err)
            if nerr, ok := err.(net.Error); ok && nerr.Timeout() {
                // 进行其他业务逻辑的处理
                continue
            }
            return
        }
        log.Printf("read %d bytes, content is %s\n", n, string(buf[:n]))
    }
}
```

如果我们要取消超时设置，可以使用 SetReadDeadline（time.Time{}）实现。

> ### socket写操作

通过 net.Conn 实例的 Write 方法，我们可以将数据写入 Socket。当 Write 调用的返回值 n 的值，与预期要写入的数据长度相等，且 err = nil 时，我们就执行了一次成功的 Socket 写操作，这是我们在调用 Write 时遇到的最常见的情形。和 Socket 的读操作一些特殊情形相比，Socket 写操作遇到的特殊情形同样不少，我们也逐一看一下。

**第一种情况：写阻塞。**TCP 协议通信两方的操作系统内核，都会为这个连接保留数据缓冲区，调用 Write 向 Socket 写入数据，实际上是将数据写入到操作系统协议栈的数据缓冲区中。TCP 是全双工通信，因此每个方向都有独立的数据缓冲。当发送方将对方的接收缓冲区，以及自身的发送缓冲区都写满后，再调用 Write 方法就会出现阻塞的情况。

我们来看一个具体例子。这个例子的客户端代码如下：

```go
func main() {
    log.Println("begin dial...")
    conn, err := net.Dial("tcp", ":8888")
    if err != nil {
        log.Println("dial error:", err)
        return
    }
    defer conn.Close()
    log.Println("dial ok")

    data := make([]byte, 65536)
    var total int
    for {
        n, err := conn.Write(data)
        if err != nil {
            total += n
            log.Printf("write %d bytes, error:%s\n", n, err)
            break
        }
        total += n
        log.Printf("write %d bytes this time, %d bytes in total\n", n, total)
    }

    log.Printf("write %d bytes in total\n", total)
}
```

客户端每次调用 Write 方法向服务端写入 65536 个字节，并在 Write 方法返回后，输出此次 Write 的写入字节数和程序启动后写入的总字节数量。服务端的处理程序逻辑，我也摘录了主要部分，你可以看一下：

```go
... ...
func handleConn(c net.Conn) {
    defer c.Close()
    time.Sleep(time.Second * 10)
    for {
        // read from the connection
        time.Sleep(5 * time.Second)
        var buf = make([]byte, 60000)
        log.Println("start to read from conn")
        n, err := c.Read(buf)
        if err != nil {
            log.Printf("conn read %d bytes,  error: %s", n, err)
            if nerr, ok := err.(net.Error); ok && nerr.Timeout() {
                continue
            }
        }

        log.Printf("read %d bytes, content is %s\n", n, string(buf[:n]))
    }
}
... ...
```

我们可以看到，服务端在前 10 秒中并不读取数据，因此当客户端一直调用 Write 方法写入数据时，写到一定量后就会发生阻塞。你可以看一下客户端的执行输出：

```go
2022/01/14 14:57:33 begin dial...
2022/01/14 14:57:33 dial ok
2022/01/14 14:57:33 write 65536 bytes this time, 65536 bytes in total
... ...
2022/01/14 14:57:33 write 65536 bytes this time, 589824 bytes in total
2022/01/14 14:57:33 write 65536 bytes this time, 655360 bytes in total  <-- 之后，写操作将阻塞
```

后续当服务端每隔 5 秒进行一次读操作后，内核 socket 缓冲区腾出了空间，客户端就又可以写入了：

```go
服务端：

2022/01/14 15:07:01 accept a new connection
2022/01/14 15:07:16 start to read from conn
2022/01/14 15:07:16 read 60000 bytes, content is
2022/01/14 15:07:21 start to read from conn
2022/01/14 15:07:21 read 60000 bytes, content is
2022/01/14 15:07:26 start to read from conn
2022/01/14 15:07:26 read 60000 bytes, content is
....



客户端(得以继续写入)：

2022/01/14 15:07:01 write 65536 bytes this time, 720896 bytes in total
2022/01/14 15:07:06 write 65536 bytes this time, 786432 bytes in total
2022/01/14 15:07:16 write 65536 bytes this time, 851968 bytes in total
2022/01/14 15:07:16 write 65536 bytes this time, 917504 bytes in total
2022/01/14 15:07:27 write 65536 bytes this time, 983040 bytes in total
2022/01/14 15:07:27 write 65536 bytes this time, 1048576 bytes in total
.... ...
```

**第二种情况：写入部分数据**。Write 操作存在写入部分数据的情况，比如上面例子中，当客户端输出日志停留在“write 65536 bytes this time, 655360 bytes in total”时，我们杀掉服务端，这时我们就会看到客户端输出以下日志：

```go
...
2022/01/14 15:19:14 write 65536 bytes this time, 655360 bytes in total
2022/01/14 15:19:16 write 24108 bytes, error:write tcp 127.0.0.1:62245->127.0.0.1:8888: write: broken pipe
2022/01/14 15:19:16 write 679468 bytes in total
```

显然，Write并不是在 655360 这个地方阻塞的，而是后续又写入 24108 个字节后发生了阻塞，服务端 Socket 关闭后，我们看到客户端又写入 24108 字节后，才返回的broken pipe错误。由于这 24108 字节数据并未真正被服务端接收到，程序需要考虑妥善处理这些数据，以防数据丢失。

**第三种情况：写入超时。**如果我们非要给 Write 操作增加一个期限，可以调用 SetWriteDeadline 方法。比如，我们可以将上面例子中的客户端源码拷贝一份，然后在新客户端源码中的 Write 调用之前，增加一行超时时间设置代码：

```go
conn.SetWriteDeadline(time.Now().Add(time.Microsecond * 10))
```

然后先后启动服务端与新客户端，我们可以看到写入超时的情况下，Write 方法的返回结果：

```go
客户端输出：

2022/01/14 15:26:34 begin dial...
2022/01/14 15:26:34 dial ok
2022/01/14 15:26:34 write 65536 bytes this time, 65536 bytes in total
... ...
2022/01/14 15:26:34 write 65536 bytes this time, 655360 bytes in total
2022/01/14 15:26:34 write 24108 bytes, error:write tcp 127.0.0.1:62325->127.0.0.1:8888: i/o timeout
2022/01/14 15:26:34 write 679468 bytes in total
```

我们可以看到，在 Write 方法写入超时时，依旧存在数据部分写入（仅写入 24108 个字节）的情况。另外，和 SetReadDeadline 一样，只要我们通过 SetWriteDeadline 设置了写超时，那无论后续 Write 方法是否成功，如果不重新设置写超时或取消写超时，后续对 Socket 的写操作都将以超时失败告终。综合上面这些例子，虽然 Go 给我们提供了阻塞 I/O 的便利，但在调用Read和Write时，依旧要综合函数返回的n和err的结果以做出正确处理。

不过，前面说的 Socket 读与写都是限于单 Goroutine 下的操作，如果多个 Goroutine 并发读或写一个 socket 会发生什么呢？我们继续往下看。

> ### socket并发读写



Goroutine 的网络编程模型，决定了存在着不同 Goroutine 间共享conn的情况，那么conn的读写是否是 Goroutine 并发安全的呢？不过，在深入这个问题之前，我们先从应用的角度上，看看并发 read 操作和 write 操作的 Goroutine 安全的必要性。

对于 Read 操作而言，由于 TCP 是面向字节流，conn.Read无法正确区分数据的业务边界，因此，多个 Goroutine 对同一个 conn 进行 read 的意义不大，Goroutine 读到不完整的业务包，反倒增加了业务处理的难度。

但对于 Write 操作而言，倒是有多个 Goroutine 并发写的情况。不过 conn 读写是否是 Goroutine 安全的测试并不是很好做，我们先深入一下运行时代码，从理论上给这个问题定个性。

首先，net.conn只是*netFD 的外层包裹结构，最终 Write 和 Read 都会落在其中的fd字段上：

```go
//$GOROOT/src/net/net.go
type conn struct {
    fd *netFD
}
```

另外，netFD 在不同平台上有着不同的实现，我们以net/fd_posix.go中的netFD为例看看：

```go
// $GOROOT/src/net/fd_unix.go

// Network file descriptor.
type netFD struct {
    pfd poll.FD 
    
    // immutable until Close
    family      int
    sotype      int
    isConnected bool // handshake completed or use of association with peer
    net         string
    laddr       Addr
    raddr       Addr
}  



```

netFD 中最重要的字段是 poll.FD 类型的 pfd，它用于表示一个网络连接。我也把它的结构摘录了一部分：

```go
// $GOROOT/src/internal/poll/fd_unix.go

// FD is a file descriptor. The net and os packages use this type as a
// field of a larger type representing a network connection or OS file.
type FD struct {
    // Lock sysfd and serialize access to Read and Write methods.
    fdmu fdMutex
    
    // System file descriptor. Immutable until Close.
    Sysfd int
    
    // I/O poller.
    pd pollDesc 

    // Writev cache.
    iovecs *[]syscall.Iovec
    ... ...    
}
```

我们看到，FD类型中包含了一个运行时实现的fdMutex类型字段。从它的注释来看，这个fdMutex用来串行化对字段Sysfd的 Write 和 Read 操作。也就是说，所有对这个 FD 所代表的连接的 Read 和 Write 操作，都是由fdMutex来同步的。从FD的 Read 和 Write 方法的实现，也证实了这一点：

```go
// $GOROOT/src/internal/poll/fd_unix.go

func (fd *FD) Read(p []byte) (int, error) {
    if err := fd.readLock(); err != nil {
        return 0, err
    }
    defer fd.readUnlock()
    if len(p) == 0 {
        // If the caller wanted a zero byte read, return immediately
        // without trying (but after acquiring the readLock).
        // Otherwise syscall.Read returns 0, nil which looks like
        // io.EOF.
        // TODO(bradfitz): make it wait for readability? (Issue 15735)
        return 0, nil
    }
    if err := fd.pd.prepareRead(fd.isFile); err != nil {
        return 0, err
    }
    if fd.IsStream && len(p) > maxRW {
        p = p[:maxRW]
    }
    for {
        n, err := ignoringEINTRIO(syscall.Read, fd.Sysfd, p)
        if err != nil {
            n = 0
            if err == syscall.EAGAIN && fd.pd.pollable() {
                if err = fd.pd.waitRead(fd.isFile); err == nil {
                    continue
                }
            }
        }
        err = fd.eofError(n, err)
        return n, err
    }
}

func (fd *FD) Write(p []byte) (int, error) {
    if err := fd.writeLock(); err != nil {
        return 0, err
    }
    defer fd.writeUnlock()
    if err := fd.pd.prepareWrite(fd.isFile); err != nil {
        return 0, err
    }
    var nn int
    for {
        max := len(p)
        if fd.IsStream && max-nn > maxRW {
            max = nn + maxRW
        }
        n, err := ignoringEINTRIO(syscall.Write, fd.Sysfd, p[nn:max])
        if n > 0 {
            nn += n
        }
        if nn == len(p) {
            return nn, err
        }
        if err == syscall.EAGAIN && fd.pd.pollable() {
            if err = fd.pd.waitWrite(fd.isFile); err == nil {
                continue
            }
        }
        if err != nil {
            return nn, err
        }
        if n == 0 {
            return nn, io.ErrUnexpectedEOF
        }
    }
}
```

你看，每次 Write 操作都是受 lock 保护，直到这次数据全部写完才会解锁。因此，在应用层面，要想保证多个 Goroutine 在一个conn上 write 操作是安全的，需要一次 write 操作完整地写入一个“业务包”。一旦将业务包的写入拆分为多次 write，那也无法保证某个 Goroutine 的某“业务包”数据在conn发送的连续性。

同时，我们也可以看出即便是 Read 操作，也是有 lock 保护的。多个 Goroutine 对同一conn的并发读，不会出现读出内容重叠的情况，但就像前面讲并发读的必要性时说的那样，一旦采用了不恰当长度的切片作为 buf，很可能读出不完整的业务包，这反倒会带来业务上的处理难度。

比如一个完整数据包：world，当 Goroutine 的读缓冲区长度 < 5 时，就存在这样一种可能：一个 Goroutine 读出了“worl”，而另外一个 Goroutine 读出了"d"。



> ### socket关闭

通常情况下，当客户端需要断开与服务端的连接时，客户端会调用 net.Conn 的 Close 方法关闭与服务端通信的 Socket。如果客户端主动关闭了 Socket，那么服务端的Read调用将会读到什么呢？这里要分“有数据关闭”和“无数据关闭”两种情况。

“有数据关闭”是指在客户端关闭连接（Socket）时，Socket 中还有服务端尚未读取的数据。在这种情况下，服务端的 Read 会成功将剩余数据读取出来，最后一次 Read 操作将得到io.EOF错误码，表示客户端已经断开了连接。如果是在“无数据关闭”情形下，服务端调用的 Read 方法将直接返回io.EOF。

不过因为 Socket 是全双工的，客户端关闭 Socket 后，如果服务端 Socket 尚未关闭，这个时候服务端向 Socket 的写入操作依然可能会成功，因为数据会成功写入己方的内核 socket 缓冲区中，即便最终发不到对方 socket 缓冲区也会这样。因此，当发现对方 socket 关闭后，己方应该正确合理处理自己的 socket，再继续 write 已经没有任何意义了。

#### 3.6 建立对协议的抽象

解决 Go 语言学习“最后一公里”的实用思路就是“理解问题” -> “技术预研与储备” -> “设计与实现”的三角循环，并且我们也完成了“理解问题”和“技术预研与储备”这两个环节，按照“三角循环”中的思路，这一讲我们应该针对实际问题进行一轮设计与实现了。

我们的目标是实现一个基于 TCP 的自定义应用层协议的通信服务端，要完成这一目标，我们需要建立协议的抽象、实现协议的打包与解包、服务端的组装、验证与优化等工作。一步一步来，我们先在程序世界建立一个对上一讲中自定义应用层协议的抽象。

程序是对现实世界的抽象。对于现实世界的自定义应用协议规范，我们需要在程序世界建立起对这份协议的抽象。在进行抽象之前，我们先建立这次实现要用的源码项目 tcp-server-demo1，建立的步骤如下：

```go
$mkdir tcp-server-demo1
$cd tcp-server-demo1
$go mod init github.com/bigwhite/tcp-server-demo1
go: creating new go.mod: module github.com/bigwhite/tcp-server-demo1
```

为了方便学习，我这里再将上一讲中的自定义协议规范贴出来对照参考：![img](markdown%E5%9B%BE%E7%89%87/70b43197100a790f3a78db50997c1d21.jpg)

在这个协议规范中，我们看到：请求包和应答包的第一个字段（totalLength）都是包的总长度，它就是用来标识包边界的那个字段，也是在应用层用于“分割包”的最重要字段。请求包与应答包的第二个字段也一样，都是 commandID，这个字段用于标识包类型，这里我们定义四种包类型：

- 连接请求包（值为 0x01）

- 消息请求包（值为 0x02）

- 连接响应包（值为 0x81）

- 消息响应包（值为 0x82）换为对应的代码就是：

- ```go
  const (
      CommandConn   = iota + 0x01 // 0x01，连接请求包
      CommandSubmit               // 0x02，消息请求包
  )
  
  const (
      CommandConnAck   = iota + 0x80 // 0x81，连接请求的响应包
      CommandSubmitAck               // 0x82，消息请求的响应包
  )
  ```

  请求包与应答包的第三个字段都是 ID，ID 是每个连接上请求包的消息流水号，顺序累加，步长为 1，循环使用，多用来请求发送方后匹配响应包，所以要求一对请求与响应消息的流水号必须相同。请求包与响应包唯一的不同之处，就在于最后一个字段：请求包定义了有效载荷（payload），这个字段承载了应用层需要的业务数据；而响应包则定义了请求包的响应状态字段（result），这里其实简化了响应状态字段的取值，成功的响应用 0 表示，如果是失败的响应，无论失败原因是什么，我们都用 1 来表示。明确了应用层协议的各个字段定义之后，我们接下来就看看如何建立起对这个协议的抽象

> ### 建立 Frame 和 Packet 抽象

首先我们要知道，TCP 连接上的数据是一个没有边界的字节流，但在业务层眼中，没有字节流，只有各种协议消息。因此，无论是从客户端到服务端，还是从服务端到客户端，业务层在连接上看到的都应该是一个挨着一个的协议消息流。现在我们建立第一个抽象：Frame。每个 Frame 表示一个协议消息，这样在业务层眼中，连接上的字节流就是由一个接着一个 Frame 组成的，如下图所示：

![img](markdown%E5%9B%BE%E7%89%87/cdcc981fdc752dea66f4ba01f7da91cf.jpg)

我们的自定义协议就封装在这一个个的 Frame 中。协议规定了将 Frame 分割开来的方法，那就是利用每个 Frame 开始处的 totalLength，每个 Frame 由一个 totalLength 和 Frame 的负载（payload）构成，比如你可以看看下图中左侧的 Frame 结构：

![img](markdown%E5%9B%BE%E7%89%87/f94954cc6ca31c383665592504600da6.jpg)

这样，我们通过 Frame header: totalLength 就可以将 Frame 之间隔离开来。在这个基础上，我们建立协议的第二个抽象：Packet。我们将 Frame payload 定义为一个 Packet。上图右侧展示的就是 Packet 的结构。Packet 就是业务层真正需要的消息，每个 Packet 由 Packet 头和 Packet Body 部分组成。Packet 头就是 commandID，用于标识这个消息的类型；而 ID 和 payload（packet payload）或 result 字段组成了 Packet 的 Body 部分，对业务层有价值的数据都包含在 Packet Body 部分。那么到这里，我们就通过 Frame 和 Packet 两个类型结构，完成了程序世界对我们私有协议规范的抽象。接下来，我们要做的就是基于 Frame 和 Packet 这两个概念，实现的对我们私有协议的解包与打包操作。

> ### 协议的解包与打包

所谓协议的解包（decode），就是指识别 TCP 连接上的字节流，将一组字节“转换”成一个特定类型的协议消息结构，然后这个消息结构会被业务处理逻辑使用。

而打包（encode）刚刚好相反，是指将一个特定类型的消息结构转换为一组字节，然后这组字节数据会被放在连接上发送出去。

具体到我们这个自定义协议上，解包就是指字节流 -> Frame，打包是指Frame -> 字节流。你可以看一下针对这个协议的服务端解包与打包的流程图：

![img](markdown%E5%9B%BE%E7%89%87/2be4e8b5b2f3c973199be6dc8f454c2b.jpg)

我们看到，TCP 流数据先后经过 frame decode 和 packet decode，得到应用层所需的 packet 数据，而业务层回复的响应，则先后经过 packet 的 encode 与 frame 的 encode，写入 TCP 数据流中。到这里，我们实际上已经完成了协议抽象的设计与解包打包原理的设计过程了。接下来，我们先来看看私有协议部分的相关代码实现。

> ### Frame的实现

前面说过，协议部分最重要的两个抽象是 Frame 和 Packet，于是我们就在项目中建立 frame 包与 packet 包，分别与两个协议抽象对应。frame 包的职责是提供识别 TCP 流边界的编解码器，我们可以很容易为这样的编解码器，定义出一个统一的接口类型 StreamFrameCodec：

```go
// tcp-server-demo1/frame/frame.go

type FramePayload []byte

type StreamFrameCodec interface {
    Encode(io.Writer, FramePayload) error   // data -> frame，并写入io.Writer
    Decode(io.Reader) (FramePayload, error) // 从io.Reader中提取frame payload，并返回给上层
}
```

StreamFrameCodec 接口类型有两个方法 Encode 与 Decode。Encode 方法用于将输入的 Frame payload 编码为一个 Frame，然后写入 io.Writer 所代表的输出（outbound）TCP 流中。而 Decode 方法正好相反，它从代表输入（inbound）TCP 流的 io.Reader 中读取一个完整 Frame，并将得到的 Frame payload 解析出来并返回。这里，我们给出一个针对我们协议的 StreamFrameCodec 接口的实现

```go
// tcp-server-demo1/frame/frame.go

var ErrShortWrite = errors.New("short write")
var ErrShortRead = errors.New("short read")

type myFrameCodec struct{}

func NewMyFrameCodec() StreamFrameCodec {
    return &myFrameCodec{}
}

func (p *myFrameCodec) Encode(w io.Writer, framePayload FramePayload) error {
    var f = framePayload
    var totalLen int32 = int32(len(framePayload)) + 4

    err := binary.Write(w, binary.BigEndian, &totalLen)
    if err != nil {
        return err
    }

    n, err := w.Write([]byte(f)) // write the frame payload to outbound stream
    if err != nil {
        return err
    }
  
    if n != len(framePayload) {
        return ErrShortWrite
    }

    return nil
}

func (p *myFrameCodec) Decode(r io.Reader) (FramePayload, error) {
    var totalLen int32
    err := binary.Read(r, binary.BigEndian, &totalLen)
    if err != nil {
        return nil, err
    }

    buf := make([]byte, totalLen-4)
    n, err := io.ReadFull(r, buf)
    if err != nil {
        return nil, err
    }
  
    if n != int(totalLen-4) {
        return nil, ErrShortRead
    }

    return FramePayload(buf), nil
}
```

在在这段实现中，有三点事项需要我们注意：

- 网络字节序使用大端字节序（BigEndian），因此无论是 Encode 还是 Decode，我们都是用 binary.BigEndian；
- binary.Read 或 Write 会根据参数的宽度，读取或写入对应的字节个数的字节，这里 totalLen 使用 int32，那么 Read 或 Write 只会操作数据流中的 4 个字节；
- 这里没有设置网络 I/O 操作的 Deadline，io.ReadFull 一般会读满你所需的字节数，除非遇到 EOF 或 ErrUnexpectedEOF。

在工程实践中，保证打包与解包正确的最有效方式就是**编写单元测试**，StreamFrameCodec 接口的 Decode 和 Encode 方法的参数都是接口类型，这让我们可以很容易为 StreamFrameCodec 接口的实现编写测试用例。下面是我为 myFrameCodec 编写了两个测试用例：

```go
// tcp-server-demo1/frame/frame_test.go

func TestEncode(t *testing.T) {
    codec := NewMyFrameCodec()
    buf := make([]byte, 0, 128)
    rw := bytes.NewBuffer(buf)

    err := codec.Encode(rw, []byte("hello"))
    if err != nil {
        t.Errorf("want nil, actual %s", err.Error())
    }

    // 验证Encode的正确性
    var totalLen int32
    err = binary.Read(rw, binary.BigEndian, &totalLen)
    if err != nil {
        t.Errorf("want nil, actual %s", err.Error())
    }

    if totalLen != 9 {
        t.Errorf("want 9, actual %d", totalLen)
    }

    left := rw.Bytes()
    if string(left) != "hello" {
        t.Errorf("want hello, actual %s", string(left))
    }
}

func TestDecode(t *testing.T) {
    codec := NewMyFrameCodec()
    data := []byte{0x0, 0x0, 0x0, 0x9, 'h', 'e', 'l', 'l', 'o'}

    payload, err := codec.Decode(bytes.NewReader(data))
    if err != nil {
        t.Errorf("want nil, actual %s", err.Error())
    }

    if string(payload) != "hello" {
        t.Errorf("want hello, actual %s", string(payload))
    }
}
```

我们看到，测试 Encode 方法，我们其实不需要建立真实的网络连接，只要用一个满足 io.Writer 的 bytes.Buffer 实例“冒充”真实网络连接就可以了，同时 bytes.Buffer 类型也实现了 io.Reader 接口，我们可以很方便地从中读取出 Encode 后的内容，并进行校验比对。为了提升测试覆盖率，我们还需要尽可能让测试覆盖到所有可测的错误执行分支上。这里，我模拟了 Read 或 Write 出错的情况，让执行流进入到 Decode 或 Encode 方法的错误分支中：

```go
type ReturnErrorWriter struct {
    W  io.Writer
    Wn int // 第几次调用Write返回错误
    wc int // 写操作次数技术
}

func (w *ReturnErrorWriter) Write(p []byte) (n int, err error) {
    w.wc++
    if w.wc >= w.Wn {
        return 0, errors.New("write error")
    }
    return w.W.Write(p)
}

type ReturnErrorReader struct {
    R  io.Reader
    Rn int // 第几次调用Read返回错误
    rc int // 读操作次数技术
}

func (r *ReturnErrorReader) Read(p []byte) (n int, err error) {
    r.rc++
    if r.rc >= r.Rn {
        return 0, errors.New("read error")
    }
    return r.R.Read(p)
}

func TestEncodeWithWriteFail(t *testing.T) {
    codec := NewMyFrameCodec()
    buf := make([]byte, 0, 128)
    w := bytes.NewBuffer(buf)

    // 模拟binary.Write返回错误
    err := codec.Encode(&ReturnErrorWriter{
        W:  w,
        Wn: 1,
    }, []byte("hello"))
    if err == nil {
        t.Errorf("want non-nil, actual nil")
    }

    // 模拟w.Write返回错误
    err = codec.Encode(&ReturnErrorWriter{
        W:  w,
        Wn: 2,
    }, []byte("hello"))
    if err == nil {
        t.Errorf("want non-nil, actual nil")
    }
}

func TestDecodeWithReadFail(t *testing.T) {
    codec := NewMyFrameCodec()
    data := []byte{0x0, 0x0, 0x0, 0x9, 'h', 'e', 'l', 'l', 'o'}

    // 模拟binary.Read返回错误
    _, err := codec.Decode(&ReturnErrorReader{
        R:  bytes.NewReader(data),
        Rn: 1,
    })
    if err == nil {
        t.Errorf("want non-nil, actual nil")
    }

    // 模拟io.ReadFull返回错误
    _, err = codec.Decode(&ReturnErrorReader{
        R:  bytes.NewReader(data),
        Rn: 2,
    })
    if err == nil {
        t.Errorf("want non-nil, actual nil")
    }
}
```

为了实现错误分支的测试，我们在测试代码源文件中创建了两个类型：ReturnErrorWriter 和 ReturnErrorReader，它们分别实现了 io.Writer 与 io.Reader。我们可以控制在第几次调用这两个类型的 Write 或 Read 方法时，返回错误，这样就可以让 Encode 或 Decode 方法按照我们的意图，进入到不同错误分支中去。有了这两个用例，我们的 frame 包的测试覆盖率（通过 go test -cover . 可以查看）就可以达到 90% 以上了。

> ### Packet 的实现

接下来，我们再看看 Packet 这个抽象的实现。和 Frame 不同，Packet 有多种类型（这里只定义了 Conn、submit、connack、submit ack)。所以我们要先抽象一下这些类型需要遵循的共同接口：

```go
// tcp-server-demo1/packet/packet.go

type Packet interface {
    Decode([]byte) error     // []byte -> struct
    Encode() ([]byte, error) //  struct -> []byte
}
```

其中，Decode 是将一段字节流数据解码为一个 Packet 类型，可能是 conn，可能是 submit 等，具体我们要根据解码出来的 commandID 判断。而 Encode 则是将一个 Packet 类型编码为一段字节流数据。考虑到篇幅与复杂性，我们这里只完成 submit 和 submitack 类型的 Packet 接口实现，省略了 conn 流程，也省略 conn 以及 connack 类型的实现，你可以课后自己思考一下有 conn 流程时代码应该如何调整。

```go
// tcp-server-demo1/packet/packet.go

type Submit struct {
    ID      string
    Payload []byte
}

func (s *Submit) Decode(pktBody []byte) error {
    s.ID = string(pktBody[:8])
    s.Payload = pktBody[8:]
    return nil
}

func (s *Submit) Encode() ([]byte, error) {
    return bytes.Join([][]byte{[]byte(s.ID[:8]), s.Payload}, nil), nil
}

type SubmitAck struct {
    ID     string
    Result uint8
}

func (s *SubmitAck) Decode(pktBody []byte) error {
    s.ID = string(pktBody[0:8])
    s.Result = uint8(pktBody[8])
    return nil
}

func (s *SubmitAck) Encode() ([]byte, error) {
    return bytes.Join([][]byte{[]byte(s.ID[:8]), []byte{s.Result}}, nil), nil
}
```

这里各种类型的编解码被调用的前提，是明确数据流是什么类型的，因此我们需要在包级提供一个导出的函数 Decode，这个函数负责从字节流中解析出对应的类型（根据 commandID），并调用对应类型的 Decode 方法：

```go
// tcp-server-demo1/packet/packet.go

func Decode(packet []byte) (Packet, error) {
  commandID := packet[0]
  pktBody := packet[1:]

  switch commandID {
  case CommandConn:
    return nil, nil
  case CommandConnAck:
    return nil, nil
  case CommandSubmit:
    s := Submit{}
    err := s.Decode(pktBody)
    if err != nil {
      return nil, err
    }
    return &s, nil
  case CommandSubmitAck:
    s := SubmitAck{}
    err := s.Decode(pktBody)
    if err != nil {
      return nil, err
    }
    return &s, nil
  default:
    return nil, fmt.Errorf("unknown commandID [%d]", commandID)
  }
}
```

同样，我们也需要包级的 Encode 函数，根据传入的 packet 类型调用对应的 Encode 方法实现对象的编码：

```go
// tcp-server-demo1/packet/packet.go

func Encode(p Packet) ([]byte, error) {
  var commandID uint8
  var pktBody []byte
  var err error

  switch t := p.(type) {
  case *Submit:
    commandID = CommandSubmit
    pktBody, err = p.Encode()
    if err != nil {
      return nil, err
    }
  case *SubmitAck:
    commandID = CommandSubmitAck
    pktBody, err = p.Encode()
    if err != nil {
      return nil, err
    }
  default:
    return nil, fmt.Errorf("unknown type [%s]", t)
  }
  return bytes.Join([][]byte{[]byte{commandID}, pktBody}, nil), nil
}
```

不过，对 packet 包中各个类型的 Encode 和 Decode 方法的测试，与 frame 包的相似，这里我就把为 packet 包编写单元测试的任务就交给你自己完成了，如果有什么问题欢迎在留言区留言。好了，万事俱备，只欠东风！下面我们就来编写服务端的程序结构，将 tcp conn 与 Frame、Packet 连接起来。

#### 3.7 服务端的组装

在上一讲中，我们按照每个连接一个 Goroutine 的模型，给出了典型 Go 网络服务端程序的结构，这里我们就以这个结构为基础，将 Frame、Packet 加进来，形成我们的第一版服务端实现：

```go
// tcp-server-demo1/cmd/server/main.go

package main

import (
  "fmt"
  "net"

  "github.com/bigwhite/tcp-server-demo1/frame"
  "github.com/bigwhite/tcp-server-demo1/packet"
)

func handlePacket(framePayload []byte) (ackFramePayload []byte, err error) {
  var p packet.Packet
  p, err = packet.Decode(framePayload)
  if err != nil {
    fmt.Println("handleConn: packet decode error:", err)
    return
  }

  switch p.(type) {
  case *packet.Submit:
    submit := p.(*packet.Submit)
    fmt.Printf("recv submit: id = %s, payload=%s\n", submit.ID, string(submit.Payload))
    submitAck := &packet.SubmitAck{
      ID:     submit.ID,
      Result: 0,
    }
    ackFramePayload, err = packet.Encode(submitAck)
    if err != nil {
      fmt.Println("handleConn: packet encode error:", err)
      return nil, err
    }
    return ackFramePayload, nil
  default:
    return nil, fmt.Errorf("unknown packet type")
  }
}

func handleConn(c net.Conn) {
  defer c.Close()
  frameCodec := frame.NewMyFrameCodec()

  for {
    // decode the frame to get the payload
    framePayload, err := frameCodec.Decode(c)
    if err != nil {
      fmt.Println("handleConn: frame decode error:", err)
      return
    }

    // do something with the packet
    ackFramePayload, err := handlePacket(framePayload)
    if err != nil {
      fmt.Println("handleConn: handle packet error:", err)
      return
    }

    // write ack frame to the connection
    err = frameCodec.Encode(c, ackFramePayload)
    if err != nil {
      fmt.Println("handleConn: frame encode error:", err)
      return
    }
  }
}

func main() {
  l, err := net.Listen("tcp", ":8888")
  if err != nil {
    fmt.Println("listen error:", err)
    return
  }

  for {
    c, err := l.Accept()
    if err != nil {
      fmt.Println("accept error:", err)
      break
    }
    // start a new goroutine to handle the new connection.
    go handleConn(c)
  }
}
```

这个程序的逻辑非常清晰，服务端程序监听 8888 端口，并在每次调用 Accept 方法后得到一个新连接，服务端程序将这个新连接交到一个新的 Goroutine 中处理。新 Goroutine 的主函数为 handleConn，有了 Packet 和 Frame 这两个抽象的加持，这个函数同样拥有清晰的代码调用结构：

```go
// handleConn的调用结构

read frame from conn
    ->frame decode
      -> handle packet
        -> packet decode
        -> packet(ack) encode
    ->frame(ack) encode
write ack frame to conn
```

到这里，一个基于 TCP 的自定义应用层协议的经典阻塞式的服务端就完成了。不过这里的服务端依旧是一个简化的实现，比如我们这里没有考虑支持优雅退出、没有捕捉某个链接上出现的可能导致整个程序退出的 panic 等，这些我也想作为作业留给你。接下来，我们就来验证一下这个服务端实现是否能正常工作。

#### 3.8 验证测试

要验证服务端的实现是否可以正常工作，我们需要实现一个自定义应用层协议的客户端。这里，我们同样基于 frame、packet 两个包，实现了一个自定义应用层协议的客户端。下面是客户端的 main 函数：

```go
// tcp-server-demo1/cmd/client/main.go
func main() {
    var wg sync.WaitGroup
    var num int = 5

    wg.Add(5)

    for i := 0; i < num; i++ {
        go func(i int) {
            defer wg.Done()
            startClient(i)
        }(i + 1)
    }
    wg.Wait()
}
```

我们看到，客户端启动了 5 个 Goroutine，模拟 5 个并发连接。startClient 函数是每个连接的主处理函数，我们来看一下：

```go
func startClient(i int) {
    quit := make(chan struct{})
    done := make(chan struct{})
    conn, err := net.Dial("tcp", ":8888")
    if err != nil {
        fmt.Println("dial error:", err)
        return
    }
    defer conn.Close()
    fmt.Printf("[client %d]: dial ok", i)

    // 生成payload
    rng, err := codename.DefaultRNG()
    if err != nil {
        panic(err)
    }

    frameCodec := frame.NewMyFrameCodec()
    var counter int

    go func() {
        // handle ack
        for {
            select {
            case <-quit:
                done <- struct{}{}
                return
            default:
            }

            conn.SetReadDeadline(time.Now().Add(time.Second * 1))
            ackFramePayLoad, err := frameCodec.Decode(conn)
            if err != nil {
                if e, ok := err.(net.Error); ok {
                    if e.Timeout() {
                        continue
                    }
                }
                panic(err)
            }

            p, err := packet.Decode(ackFramePayLoad)
            submitAck, ok := p.(*packet.SubmitAck)
            if !ok {
                panic("not submitack")
            }
            fmt.Printf("[client %d]: the result of submit ack[%s] is %d\n", i, submitAck.ID, submitAck.Result)
        }
    }()

    for {
        // send submit
        counter++
        id := fmt.Sprintf("%08d", counter) // 8 byte string
        payload := codename.Generate(rng, 4)
        s := &packet.Submit{
            ID:      id,
            Payload: []byte(payload),
        }

        framePayload, err := packet.Encode(s)
        if err != nil {
            panic(err)
        }

        fmt.Printf("[client %d]: send submit id = %s, payload=%s, frame length = %d\n",
            i, s.ID, s.Payload, len(framePayload)+4)

        err = frameCodec.Encode(conn, framePayload)
        if err != nil {
            panic(err)
        }

        time.Sleep(1 * time.Second)
        if counter >= 10 {
            quit <- struct{}{}
            <-done
            fmt.Printf("[client %d]: exit ok", i)
            return
        }
    }
}
```

关于 startClient 函数，我们需要简单说明几点。

首先，startClient 函数启动了两个 Goroutine，一个负责向服务端发送 submit 消息请求，另外一个 Goroutine 则负责读取服务端返回的响应；

其次，客户端发送的 submit 请求的负载（payload）是由第三方包 github.com/lucasepe/codename 负责生成的，这个包会生成一些对人类可读的随机字符串，比如：firm-iron、 moving-colleen、game-nova 这样的字符串；

另外，负责读取服务端返回响应的 Goroutine，使用 SetReadDeadline 方法设置了读超时，这主要是考虑该 Goroutine 可以在收到退出通知时，能及时从 Read 阻塞中跳出来。

好了，现在我们就来构建和运行一下这两个程序。我在 tcp-server-demo1 目录下提供了 Makefile，如果你使用的是 Linux 或 macOS 操作系统，可以直接敲入 make 构建两个程序，如果你是在 Windows 下构建，可以直接敲入下面的 go build 命令构建：

```go
$make
go build github.com/bigwhite/tcp-server-demo1/cmd/server
go build github.com/bigwhite/tcp-server-demo1/cmd/client
```

构建成功后，我们先来启动 server 程序

```go
$./server
server start ok(on *.8888)
```

然后，我们启动 client 程序，启动后 client 程序便会向服务端建立 5 条连接，并发送 submit 请求，client 端的部分日志如下：

```go
$./client
[client 5]: dial ok
[client 1]: dial ok
[client 5]: send submit id = 00000001, payload=credible-deathstrike-33e1, frame length = 38
[client 3]: dial ok
[client 1]: send submit id = 00000001, payload=helped-lester-8f15, frame length = 31
[client 4]: dial ok
[client 4]: send submit id = 00000001, payload=strong-timeslip-07fa, frame length = 33
[client 3]: send submit id = 00000001, payload=wondrous-expediter-136e, frame length = 36
[client 5]: the result of submit ack[00000001] is 0
[client 1]: the result of submit ack[00000001] is 0
[client 3]: the result of submit ack[00000001] is 0
[client 2]: dial ok
... ...
[client 3]: send submit id = 00000010, payload=bright-monster-badoon-5719, frame length = 39
[client 4]: send submit id = 00000010, payload=crucial-wallop-ec2d, frame length = 32
[client 2]: send submit id = 00000010, payload=pro-caliban-c803, frame length = 29
[client 1]: send submit id = 00000010, payload=legible-shredder-3d81, frame length = 34
[client 5]: send submit id = 00000010, payload=settled-iron-monger-bf78, frame length = 37
[client 3]: the result of submit ack[00000010] is 0
[client 4]: the result of submit ack[00000010] is 0
[client 1]: the result of submit ack[00000010] is 0
[client 2]: the result of submit ack[00000010] is 0
[client 5]: the result of submit ack[00000010] is 0
[client 4]: exit ok
[client 1]: exit ok
[client 3]: exit ok
[client 5]: exit ok
[client 2]: exit ok
```

client 在每条连接上发送 10 个 submit 请求后退出。这期间服务端会输出如下日志：

```go
recv submit: id = 00000001, payload=credible-deathstrike-33e1
recv submit: id = 00000001, payload=helped-lester-8f15
recv submit: id = 00000001, payload=wondrous-expediter-136e
recv submit: id = 00000001, payload=strong-timeslip-07fa
recv submit: id = 00000001, payload=delicate-leatherneck-4b12
recv submit: id = 00000002, payload=certain-deadpool-779d
recv submit: id = 00000002, payload=clever-vapor-25ce
recv submit: id = 00000002, payload=causal-guardian-4f84
recv submit: id = 00000002, payload=noted-tombstone-1b3e
... ...
recv submit: id = 00000010, payload=settled-iron-monger-bf78
recv submit: id = 00000010, payload=pro-caliban-c803
recv submit: id = 00000010, payload=legible-shredder-3d81
handleConn: frame decode error: EOF
handleConn: frame decode error: EOF
handleConn: frame decode error: EOF
handleConn: frame decode error: EOF
handleConn: frame decode error: EOF
```

#### 3.9 Go 程序优化的基本套路







### 4 构建一个web服务



#### 4.1 预热：最简单的 HTTP 服务

我们首先按下面步骤建立一个 simple-http-server 目录，并创建一个名为 simple-http-server 的 Go Module：

```go
$mkdir simple-http-server
$cd simple-http-server
$go mod init simple-http-server
```

由于这个 HTTP 服务比较简单，我们采用最简项目布局，也就是在 simple-http-server 目录下创建一个 main.go 源文件：

```go
  package main
 
  import "net/http"
  
  func main() {
      http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request){
          w.Write([]byte("hello, world"))
      })
      http.ListenAndServe(":8080", nil)
 }
```

这些代码就是一个最简单的 HTTP 服务的实现了。在这个实现中，我们只使用了 Go 标准库的 http 包。可能你现在对 http 包还不熟悉，但没有关系，你现在只需要大致了解上面代码的结构与原理就可以了。

这段代码里，你要注意两个重要的函数，一个是 ListenAndServe，另一个是 HandleFunc。

你会看到，代码的第 9 行，我们通过 http 包提供的 ListenAndServe 函数，建立起一个 HTTP 服务，这个服务监听本地的 8080 端口。客户端通过这个端口与服务建立连接，发送 HTTP 请求就可以得到相应的响应结果。那么服务端是如何处理客户端发送的请求的呢？我们看上面代码中的第 6 行。在这一行中，我们为这个服务设置了一个处理函数。这个函数的函数原型是这样的：

```go
func(w http.ResponseWriter, r *http.Request)
```

这个函数里有两个参数，w 和 r。第二个参数 r 代表来自客户端的 HTTP 请求，第一个参数 w 则是用来操作返回给客户端的应答的，基于 http 包实现的 HTTP 服务的处理函数都要符合这一原型。

你也发现了，在这个例子中，所有来自客户端的请求，无论请求的 URI 路径（RequestURI）是什么，请求都会被我们设置的处理函数处理。为什么会这样呢？

这是因为，我们通过 http.HandleFunc 设置这个处理函数时，传入的模式字符串为“/”。HTTP 服务器在收到请求后，会将请求中的 URI 路径与设置的模式字符串进行**最长前缀匹配**，并执行匹配到的模式字符串所对应的处理函数。在这个例子中，我们仅设置了“/”这一个模式字符串，并且所有请求的 URI 都能与之匹配，自然所有请求都会被我们设置的处理函数处理。

接着，我们再来编译运行一下这个程序，直观感受一下 HTTP 服务处理请求的过程。我们首先按下面步骤来编译并运行这个程序：

```go
$cd simple-http-server
$go build
$./simple-http-server
```

接下来，我们用 curl 命令行工具模拟客户端，向上述服务建立连接并发送 http 请求：

```go
$curl localhost:8080/
hello, world
```

我们看到，curl 成功得到了 http 服务返回的“hello, world”响应数据。到此，我们的 HTTP 服务就构建成功了。当然了，真实世界的 Web 服务不可能像上述例子这么简单，这仅仅是一个“预热”。我想让你知道，使用 Go 构建 Web 服务是非常容易的。并且，这样的预热也能让你初步了解实现代码的结构，先有一个技术铺垫。



#### 4.2 图书管理 API 服务

首先，我们先来明确一下我们的业务逻辑。在这个实战小项目中，我们模拟的是真实世界的一个书店的图书管理后端服务。这个服务为平台前端以及其他客户端，提供针对图书的 CRUD（创建、检索、更新与删除）的基于 HTTP 协议的 API。API 采用典型的 RESTful 风格设计，这个服务提供的 API 集合如下:

![img](markdown%E5%9B%BE%E7%89%87/99717b62f7553e1a5139edcf2ac03b51.jpg)

这个 API 服务的逻辑并不复杂。简单来说，我们通过 id 来唯一标识一本书，对于图书来说，这个 id 通常是 ISBN 号。至于客户端和服务端中请求与响应的数据，我们采用放在 HTTP 协议包体（Body）中的 Json 格式数据来承载。



#### 4.3 项目建立与布局设计

我们按照下面步骤创建一个名为 bookstore 的 Go 项目并创建对应的 Go Module：

```go
$mkdir bookstore
$cd bookstore
$go mod init bookstore
go: creating new go.mod: module bookstore
```

通过上面的业务逻辑说明，我们可以把这个服务大体拆分为两大部分，一部分是 HTTP 服务器，用来对外提供 API 服务；另一部分是图书数据的存储模块，所有的图书数据均存储在这里。

同时，这是一个以构建可执行程序为目的的 Go 项目，我们参考 Go 项目布局标准一讲中的项目布局，把这个项目的结构布局设计成这样：

```go
├── cmd/
│   └── bookstore/         // 放置bookstore main包源码
│       └── main.go
├── go.mod                 // module bookstore的go.mod
├── go.sum
├── internal/              // 存放项目内部包的目录
│   └── store/
│       └── memstore.go     
├── server/                // HTTP服务器模块
│   ├── middleware/
│   │   └── middleware.go
│   └── server.go          
└── store/                 // 图书数据存储模块
    ├── factory/
    │   └── factory.go
    └── store.go
```



#### 4.4 项目 main 包

main 包是主要包，为了搞清楚各个模块之间的关系，我在这里给出了 main 包的实现逻辑图：

![img](markdown%E5%9B%BE%E7%89%87/5e8ee50b67a4229210b12afb94f55a19.jpg)

```go
  package main
 
  import (
      _ "bookstore/internal/store"
      "bookstore/server"
      "bookstore/store/factory"
      "context"
      "log"
      "os"
      "os/signal"
      "syscall"
      "time"
 )
 
 func main() {
     s, err := factory.New("mem") // 创建图书数据存储模块实例
     if err != nil {
         panic(err)
     }
 
     srv := server.NewBookStoreServer(":8080", s) // 创建http服务实例
 
     errChan, err := srv.ListenAndServe() // 运行http服务
     if err != nil {
         log.Println("web server start failed:", err)
         return
     }
     log.Println("web server start ok")
 
     c := make(chan os.Signal, 1)
     signal.Notify(c, syscall.SIGINT, syscall.SIGTERM)
 
     select { // 监视来自errChan以及c的事件
     case err = <-errChan:
         log.Println("web server run failed:", err)
         return
     case <-c:
         log.Println("bookstore program is exiting...")
         ctx, cf := context.WithTimeout(context.Background(), time.Second)
         defer cf()
         err = srv.Shutdown(ctx) // 优雅关闭http服务实例
     }
 
     if err != nil {
         log.Println("bookstore program exit error:", err)
         return
     }
     log.Println("bookstore program exit ok")
 }
```

在 Go 中，main 包不仅包含了整个程序的入口，它还是整个程序中主要模块初始化与组装的场所。那对应在我们这个程序中，主要模块就是第 16 行的创建图书存储模块实例，以及第 21 行创建 HTTP 服务模块实例。而且，你还要注意的是，第 21 行创建 HTTP 服务模块实例的时候，我们把图书数据存储实例 s 作为参数，传递给了 NewBookStoreServer 函数。这两个实例的创建原理，我们等会再来细细探讨。

这里，我们重点来看 main 函数的后半部分（第 30 行~ 第 42 行），这里表示的是，我们通过监视系统信号实现了 http 服务实例的优雅退出。

所谓优雅退出，指的就是程序有机会等待其他的事情处理完再退出。比如尚未完成的事务处理、清理资源（比如关闭文件描述符、关闭 socket）、保存必要中间状态、内存数据持久化落盘，等等。如果你经常用 Go 来编写 http 服务，那么 http 服务如何优雅退出，就是你经常要考虑的问题。

在这个问题的具体实现上，我们通过 signal 包的 Notify 捕获了 SIGINT、SIGTERM 这两个系统信号。这样，当这两个信号中的任何一个触发时，我们的 http 服务实例都有机会在退出前做一些清理工作。然后，我们再使用 http 服务实例（srv）自身提供的 Shutdown 方法，来实现 http 服务实例内部的退出清理工作，包括：立即关闭所有 listener、关闭所有空闲的连接、等待处于活动状态的连接处理完毕，等等。当 http 服务实例的清理工作完成后，我们整个程序就可以正常退出了。接下来，我们再重点看看构成 bookstore 程序的两个主要模块：图书数据存储模块与 HTTP 服务模块的实现。我们按照 main 函数中的初始化顺序，先来看看图书数据存储模块。



#### 4.5 图书数据存储模块（store)

图书数据存储模块的职责很清晰，就是用来**存储整个 bookstore 的图书数据**的。图书数据存储有很多种实现方式，最简单的方式莫过于在内存中创建一个 map，以图书 id 作为 key，来保存图书信息，我们在这一讲中也会采用这种方式。但如果我们要考虑上生产环境，数据要进行持久化，那么最实际的方式就是通过 Nosql 数据库甚至是关系型数据库，实现对图书数据的存储与管理。考虑到对多种存储实现方式的支持，我们将针对图书的有限种存储操作，放置在一个接口类型 Store 中，如下源码所示：

```go
// store/store.go

 type Book struct {
     Id      string   `json:"id"`      // 图书ISBN ID
     Name    string   `json:"name"`    // 图书名称
     Authors []string `json:"authors"` // 图书作者
     Press   string   `json:"press"`   // 出版社
 }
 
 type Store interface {
     Create(*Book) error        // 创建一个新图书条目
     Update(*Book) error        // 更新某图书条目
     Get(string) (Book, error)  // 获取某图书信息
     GetAll() ([]Book, error)   // 获取所有图书信息
     Delete(string) error       // 删除某图书条目
 }
```

这里，我们建立了一个对应图书条目的抽象数据类型 Book，以及针对 Book 存取的接口类型 Store。这样，对于想要进行图书数据操作的一方来说，他只需要得到一个满足 Store 接口的实例，就可以实现对图书数据的存储操作了，不用再关心图书数据究竟采用了何种存储方式。这就实现了图书存储操作与底层图书数据存储方式的解耦。而且，这种面向接口编程也是 Go 组合设计哲学的一个重要体现。那我们具体如何创建一个满足 Store 接口的实例呢？我们可以参考《设计模式》提供的多种创建型模式，选择一种 Go 风格的工厂模式（创建型模式的一种）来实现满足 Store 接口实例的创建。我们看一下 store/factory 包的源码：

```go
// store/factory/factory.go

 var (
     providersMu sync.RWMutex
     providers   = make(map[string]store.Store)
 )
 
 func Register(name string, p store.Store) {
     providersMu.Lock()
    defer providersMu.Unlock()
     if p == nil {
         panic("store: Register provider is nil")
     }
 
     if _, dup := providers[name]; dup {
         panic("store: Register called twice for provider " + name)
     }
     providers[name] = p
 }
 
 func New(providerName string) (store.Store, error) {
     providersMu.RLock()
     p, ok := providers[providerName]
     providersMu.RUnlock()
     if !ok {
         return nil, fmt.Errorf("store: unknown provider %s", providerName)
     }
 
     return p, nil
 }
```

这段代码实际上是效仿了 Go 标准库的 database/sql 包采用的方式，factory 包采用了一个 map 类型数据，对工厂可以“生产”的、满足 Store 接口的实例类型进行管理。factory 包还提供了 Register 函数，让各个实现 Store 接口的类型可以把自己“注册”到工厂中来。

一旦注册成功，factory 包就可以“生产”出这种满足 Store 接口的类型实例。而依赖 Store 接口的使用方，只需要调用 factory 包的 New 函数，再传入期望使用的图书存储实现的名称，就可以得到对应的类型实例了。

在项目的 internal/store 目录下，我们还提供了一个基于内存 map 的 Store 接口的实现，我们具体看一下这个实现是怎么自注册到 factory 包中的：

```go
// internal/store/memstore.go

 package store
  
 import (
     mystore "bookstore/store"
     factory "bookstore/store/factory"
     "sync"
 )
  
 func init() {
     factory.Register("mem", &MemStore{
         books: make(map[string]*mystore.Book),
     })
 }
 
 type MemStore struct {
     sync.RWMutex
     books map[string]*mystore.Book
 }
```

从 memstore 的代码来看，它是在包的 init 函数中调用 factory 包提供的 Register 函数，把自己的实例以“mem”的名称注册到 factory 中的。这样做有一个好处，依赖 Store 接口进行图书数据管理的一方，只要导入 internal/store 这个包，就可以自动完成注册动作了。理解了这个之后，我们再看下面 main 包中，创建图书数据存储模块实例时采用的代码，是不是就豁然开朗了？

```go
import (
    ... ...
    _ "bookstore/internal/store" // internal/store将自身注册到factory中
)

func main() {
    s, err := factory.New("mem") // 创建名为"mem"的图书数据存储模块实例
    if err != nil {
        panic(err)
    }
    ... ...
}
```

至于 memstore.go 中图书数据存储的具体逻辑，就比较简单了，我这里就不详细分析了.



#### 4.6 HTTP 服务模块（server)

HTTP 服务模块的职责是**对外提供 HTTP API 服务，处理来自客户端的各种请求，并通过 Store 接口实例执行针对图书数据的相关操作。**这里，我们抽象处理一个 server 包，这个包中定义了一个 BookStoreServer 类型如下：

```go
// server/server.go

 type BookStoreServer struct {
     s   store.Store
     srv *http.Server
 }
```

我们看到，这个类型实质上就是一个标准库的 http.Server，并且组合了来自 store.Store 接口的能力。server 包提供了 NewBookStoreServer 函数，用来创建一个 BookStoreServer 类型实例：

```go
// server/server.go

 func NewBookStoreServer(addr string, s store.Store) *BookStoreServer {
     srv := &BookStoreServer{
         s: s,
         srv: &http.Server{
             Addr: addr,
         },
     }
 
     router := mux.NewRouter()
     router.HandleFunc("/book", srv.createBookHandler).Methods("POST")
     router.HandleFunc("/book/{id}", srv.updateBookHandler).Methods("POST")
     router.HandleFunc("/book/{id}", srv.getBookHandler).Methods("GET")
     router.HandleFunc("/book", srv.getAllBooksHandler).Methods("GET")
     router.HandleFunc("/book/{id}", srv.delBookHandler).Methods("DELETE")
 
     srv.srv.Handler = middleware.Logging(middleware.Validating(router))
     return srv
 }
```

我们看到函数 NewBookStoreServer 接受两个参数，一个是 HTTP 服务监听的服务地址，另外一个是实现了 store.Store 接口的类型实例。这种函数原型的设计是 Go 语言的一种惯用设计方法，也就是接受一个接口类型参数，返回一个具体类型。返回的具体类型组合了传入的接口类型的能力。

这个时候，和前面预热时实现的简单 http 服务一样，我们还需为 HTTP 服务器设置请求的处理函数。

由于这个服务请求 URI 的模式字符串比较复杂，标准库 http 包内置的 URI 路径模式匹配器（ServeMux，也称为路由管理器）不能满足我们的需求，因此在这里，我们需要借助一个第三方包 github.com/gorilla/mux 来实现我们的需求。

在上面代码的第 11 行到第 16 行，我们针对不同 URI 路径模式设置了不同的处理函数。我们以 createBookHandler 和 getBookHandler 为例来看看这些处理函数的实现：

```go
// server/server.go

  func (bs *BookStoreServer) createBookHandler(w http.ResponseWriter, req *http.Request) {
      dec := json.NewDecoder(req.Body)
      var book store.Book
      if err := dec.Decode(&book); err != nil {
          http.Error(w, err.Error(), http.StatusBadRequest)
          return
      }
  
      if err := bs.s.Create(&book); err != nil {
          http.Error(w, err.Error(), http.StatusBadRequest)
          return
      }
  }

  func (bs *BookStoreServer) getBookHandler(w http.ResponseWriter, req *http.Request) {
      id, ok := mux.Vars(req)["id"]
      if !ok {
          http.Error(w, "no id found in request", http.StatusBadRequest)
          return
      }
  
     book, err := bs.s.Get(id)
     if err != nil {
         http.Error(w, err.Error(), http.StatusBadRequest)
         return
     }
 
     response(w, book)
 }

 func response(w http.ResponseWriter, v interface{}) {
     data, err := json.Marshal(v)
     if err != nil {
         http.Error(w, err.Error(), http.StatusInternalServerError)
         return
     }
     w.Header().Set("Content-Type", "application/json")
     w.Write(data)
 }
```

这些处理函数的实现都大同小异，都是先获取 http 请求包体数据，然后通过标准库 json 包将这些数据，解码（decode）为我们需要的 store.Book 结构体实例，再通过 Store 接口对图书数据进行存储操作。如果我们是获取图书数据的请求，那么处理函数将通过 response 函数，把取出的图书数据编码到 http 响应的包体中，并返回给客户端。

然后，在 NewBookStoreServer 函数实现的尾部，我们还看到了这样一行代码：

```go
     srv.srv.Handler = middleware.Logging(middleware.Validating(router))
```

这行代码的意思是说，我们在 router 的外围包裹了两层 middleware。什么是 middleware 呢？对于我们的上下文来说，这些 middleware 就是一些通用的 http 处理函数。我们看一下这里的两个 middleware，也就是 Logging 与 Validating 函数的实现：

```go
// server/middleware/middleware.go

  func Logging(next http.Handler) http.Handler {
     return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
         log.Printf("recv a %s request from %s", req.Method, req.RemoteAddr)
         next.ServeHTTP(w, req)
     })
 }
 
 func Validating(next http.Handler) http.Handler {
     return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
         contentType := req.Header.Get("Content-Type")
         mediatype, _, err := mime.ParseMediaType(contentType)
         if err != nil {
             http.Error(w, err.Error(), http.StatusBadRequest)
             return
         }
         if mediatype != "application/json" {
             http.Error(w, "invalid Content-Type", http.StatusUnsupportedMediaType)
             return
         }
         next.ServeHTTP(w, req)
     })
 }
```

我们看到，Logging 函数主要用来输出每个到达的 HTTP 请求的一些概要信息，而 Validating 则会对每个 http 请求的头部进行检查，检查 Content-Type 头字段所表示的媒体类型是否为 application/json。这些通用的 middleware 函数，会被串联到每个真正的处理函数之前，避免我们在每个处理函数中重复实现这些逻辑。

创建完 BookStoreServer 实例后，我们就可以调用其 ListenAndServe 方法运行这个 http 服务了，显然这个方法的名字是仿效 http.Server 类型的同名方法，我们的实现是这样的：

```go
// server/server.go

 func (bs *BookStoreServer) ListenAndServe() (<-chan error, error) {
     var err error
     errChan := make(chan error)
     go func() {
         err = bs.srv.ListenAndServe()
         errChan <- err
     }()
 
     select {
     case err = <-errChan:
         return nil, err
     case <-time.After(time.Second):
         return errChan, nil
     }
 }
```

我们看到，这个函数把 BookStoreServer 内部的 http.Server 的运行，放置到一个单独的轻量级线程 Goroutine 中。这是因为，http.Server.ListenAndServe 会阻塞代码的继续运行，如果不把它放在单独的 Goroutine 中，后面的代码将无法得到执行。为了检测到 http.Server.ListenAndServe 的运行状态，我们再通过一个 channel（位于第 5 行的 errChan），在新创建的 Goroutine 与主 Goroutine 之间建立的通信渠道。通过这个渠道，这样我们能及时得到 http server 的运行状态。



#### 4.7 编译、运行与验证

到这里，bookstore 项目的大部分重要代码我们都分析了一遍，是时候将程序跑起来看看了。不过，因为我们在程序中引入了一个第三方依赖包，所以在构建项目之前，我们需要执行下面这个命令，让 Go 命令自动分析依赖项和版本，并更新 go.mod：

```go
$go mod tidy
go: finding module for package github.com/gorilla/mux
go: found github.com/gorilla/mux in github.com/gorilla/mux v1.8.0
```

完成后，我们就可以按下面的步骤来构建并执行 bookstore 了：

```go
$go build bookstore/cmd/bookstore
$./bookstore
2021/10/05 16:08:36 web server start ok
```

如果你看到上面这个输出的日志，说明我们的程序启动成功了。现在，我们就可以像前面一样使用 curl 命令行工具，模仿客户端向 bookstore 服务发起请求了，比如创建一个新书条目：

```go
$curl -X POST -H "Content-Type:application/json" -d '{"id": "978-7-111-55842-2", "name": "The Go Programming Language", "authors":["Alan A.A.Donovan", "Brian W. Kergnighan"],"press": "Pearson Education"}' localhost:8080/book
```

此时服务端会输出如下日志，表明我们的 bookstore 服务收到了客户端请求。

```go
2021/10/05 16:09:10 recv a POST request from [::1]:58021
```

接下来，我们再来获取一下这本书的信息：

```go
$curl -X GET -H "Content-Type:application/json" localhost:8080/book/978-7-111-55842-2
{"id":"978-7-111-55842-2","name":"The Go Programming Language","authors":["Alan A.A.Donovan","Brian W. Kergnighan"],"press":"Pearson Education"}
```



# 手把手带你写一个Web框架



# 刘丹冰Zinx



# 微服务商城



# 孔令飞iam