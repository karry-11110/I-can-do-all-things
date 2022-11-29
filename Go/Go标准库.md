## 1 fmt



> #### 向外输出

- **fmt.Print**

```go
func Print(a ...interface{}) (n int, err error)
func Printf(format string, a ...interface{}) (n int, err error)
func Println(a ...interface{}) (n int, err error)
```

`Print`系列函数会将内容输出到系统的标准输出，区别在于`Print`函数直接输出内容，`Printf`函数支持格式化输出字符串，`Println`函数会在输出内容的结尾添加一个换行符。



- **fmt.Fprint**

```go
func Fprint(w io.Writer, a ...interface{}) (n int, err error)
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error)
func Fprintln(w io.Writer, a ...interface{}) (n int, err error)
```

`Fprint`系列函数会将内容输出到一个`io.Writer`接口类型的变量`w`(**注意，只要满足`io.Writer`接口的类型都支持写入**)中，我们通常用这个函数往文件中写入内容

举例：

```go
// 向标准输出写入内容
fmt.Fprintln(os.Stdout, "向标准输出写入内容")
fileObj, err := os.OpenFile("./xx.txt", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
if err != nil {
	fmt.Println("打开文件出错，err:", err)
	return
}
name := "沙河小王子"
// 向打开的文件句柄中写入内容
fmt.Fprintf(fileObj, "往文件中写如信息：%s", name)
```



- **fmt.Sprint**

```go
func Sprint(a ...interface{}) string
func Sprintf(format string, a ...interface{}) string
func Sprintln(a ...interface{}) string
```

`Sprint`系列函数会把传入的数据生成并返回一个字符串



- **fmt.Errof**

```go
func Errorf(format string, a ...interface{}) error
```

`Errorf`函数根据format参数生成格式化字符串并返回一个包含该字符串的错误,通常使用这种方式来自定义错误类型

```go
e := errors.New("原始错误e")
w := fmt.Errorf("Wrap了一个错误%w", e)
```

Go1.13版本为`fmt.Errorf`函数新加了一个`%w`占位符用来生成一个可以包裹Error的Wrapping Error。



> #### 格式化占位符

- **通用占位符**

| 占位符 |                说明                |
| :----: | :--------------------------------: |
|   %v   |          值的默认格式表示          |
|  %+v   | 类似%v，但输出结构体时会添加字段名 |
|  %#v   |           值的Go语法表示           |
|   %T   |            打印值的类型            |
|   %%   |               百分号               |

- **数据类型**

| 占位符 |              说明              |
| :----: | :----------------------------: |
|   %t   |          true或false           |
|   %b   |          表示为二进制          |
|   %o   |          表示为八进制          |
|   %d   |          表示为十进制          |
|   %e   |  科学计数法，如-1234.456e+78   |
|   %s   |    直接输出字符串或者[]byte    |
|   %p   | 表示为十六进制，并加上前导的0x |
|   %f   |       默认宽度，默认精度       |
|  %9f   |        宽度9，默认精度         |



> #### 获取输入

- **fmt.Scan**

```go
func Scan(a ...interface{}) (n int, err error)
func Scanf(format string, a ...interface{}) (n int, err error)
func Scanln(a ...interface{}) (n int, err error)
```

Scan从标准输入扫描文本，读取由空白符分隔的值保存到传递给本函数的参数中，换行符视为空白符。`fmt.Scanf`不同于`fmt.Scan`简单的以空格作为输入数据的分隔符，`fmt.Scanf`为输入数据指定了具体的输入内容格式，只有按照格式输入数据才会被扫描并存入对应变量。`fmt.Scanf`不同于`fmt.Scan`简单的以空格作为输入数据的分隔符，`fmt.Scanf`为输入数据指定了具体的输入内容格式，只有按照格式输入数据才会被扫描并存入对应变量。

函数返回成功扫描的数据个数和遇到的任何错误。如果读取的数据个数比提供的参数少，会返回一个错误报告原因



- **bufio.NewReader**

```go
file, err := os.Open("./xx.txt")
reader := bufio.NewReader(file)
line, err := reader.ReadString('\n') //注意是字符
```



- **fmt.Fscan**

```go
func Fscan(r io.Reader, a ...interface{}) (n int, err error)
func Fscanln(r io.Reader, a ...interface{}) (n int, err error)
func Fscanf(r io.Reader, format string, a ...interface{}) (n int, err error)
```

这几个函数功能分别类似于`fmt.Scan`、`fmt.Scanf`、`fmt.Scanln`三个函数，只不过它们不是从标准输入中读取数据而是从`io.Reader`中读取数据



- **fmt.Sscan**

```go
func Sscan(str string, a ...interface{}) (n int, err error)
func Sscanln(str string, a ...interface{}) (n int, err error)
func Sscanf(str string, format string, a ...interface{}) (n int, err error)
```

这几个函数功能分别类似于`fmt.Scan`、`fmt.Scanf`、`fmt.Scanln`三个函数，只不过它们不是从标准输入中读取数据而是从指定字符串中读取数据



## 2 文件操作



> #### 打开,关闭,新建文件

- **os.Open**

```go
file, err := os.Open(filename)
```

如果文件不存在，就会返回错误，如果存在就以**只读**的方式打开文件



- **os.OpenFile**

```go
func OpenFile(name string, flag int, perm FileMode) (*File, error) {
	...
}
```

`os.OpenFile()`函数能够以指定模式打开文件，从而实现文件写入相关功能。`name`：要打开的文件名 `flag`：打开文件的模式`perm`：文件权限，一个八进制数。r（读）04，w（写）02，x（执行）01

|     模式      |                             含义                             |
| :-----------: | :----------------------------------------------------------: |
| `os.O_WRONLY` |                             只写                             |
| `os.O_CREATE` |                           创建文件                           |
| `os.O_RDONLY` |                             只读                             |
|  `os.O_RDWR`  |                             读写                             |
| `os.O_TRUNC`  |                             清空                             |
| `os.O_APPEND` |                             追加                             |
| ``os.O_EXCL`  | 如果使用O_CREAT时文件存在，就返回错误信息，它可以测试文件是否存在 |



- **file.close**

```go
defer f.Close()
```



- **os.Create**

```go
f, err := os.Create(filename)
```

如果文件已经存在，那么 `os.Create()` 会将文件清空



> #### 读取文件

- **file.Read**

```go
file, err := os.Open(filename)
func (f *File) Read(b []byte) (n int, err error)
```

它接收一个字节切片，返回读取的字节数和可能的具体错误，读到文件末尾时会返回`0`和`io.EOF`.但是使用该方法时需要预先分配空间，用于存储读取的文件内容。我们当然可以提前获取文件的大小，但是这种方式仍然不如 `ioutil.ReadAll()` 方便。甚至可以直接使用 `ioutil.ReadFile()`

举例：

```go
// 使用Read方法读取数据
	var tmp = make([]byte, 128)
	n, err := file.Read(tmp)
	if err == io.EOF {
		fmt.Println("文件读完了")
		return
	}
```



- **ioutil.ReadAll**

```go
file, err := os.Open(filename)
content, err := ioutil.ReadAll(file)
```

- **ioutil.ReadFile**

```go
content, err := ioutil.ReadFile(filename)
```

注意这里和ReadAll的区别，它接受的直接是文件路径参数



- **bufio.NewReader**

```go
file, err := os.Open("./xx.txt")
reader := bufio.NewReader(file)
line, err := reader.ReadString('\n') //注意是字符
```



> #### 写入操作

- **Write和WriteString**

```go
file, err := os.OpenFile("xx.txt", os.O_CREATE|os.O_TRUNC|os.O_WRONLY, 0666)
file.Write([]byte(str))       //写入字节切片数据
file.WriteString("hello 小王子") //直接写入字符串数据
```



- **bufio.NewWriter**

```go
file, err := os.OpenFile("xx.txt", os.O_CREATE|os.O_TRUNC|os.O_WRONLY, 0666)
writer := bufio.NewWriter(file)
for i := 0; i < 10; i++ {
writer.WriteString("hello沙河\n") //将数据先写入缓存
}
writer.Flush() //将缓存中的内容写入文件
```



- **ioutil.WriteFile**

```go
err := ioutil.WriteFile("./xx.txt", []byte(str), 0666)
```



> #### 获取文件路径



- **os.Args**

程序获取运行时给出的参数，可以通过os包来实现:

```go
func main () {
	for idx, args := range os.Args {
		fmt.Println("参数" + strconv.Itoa(idx) + ":", args)
	}
}
```

```go
$go run main.go 1 3 -X ?
参数0: /tmp/go-build116558042/command-line-arguments/_obj/exe/main
参数1: 1
参数2: 3
参数3: -X
参数4: ?
```

可以看到，命令行参数包括了程序路径本身，以及通常意义上的参数。
程序中os.Args的类型是 **[]string** ，也就是字符串切片。所以可以在for循环的range中遍历，还可以用 **len(os.Args)** 来获取其数量。



- **filepath.Ext**

```
func Ext(path string) string
```

**返回值：**它返回指定路径使用的文件扩展名,如‘.go’

## 3 time



> #### 获取时间对象及其时间戳

首先，我们需要了解的是Go 语言中使用`time.Time`类型表示时间。

```go
now := time.Now() // 获取当前时间

year := now.Year()     // 年
month := now.Month()   // 月
day := now.Day()       // 日
hour := now.Hour()     // 小时
minute := now.Minute() // 分钟
second := now.Second() // 秒
```



> #### 时间操作

我们需要了解的是：`time.Duration`是`time`包定义的一个类型，它代表两个时间点之间经过的时间，以纳秒为单位。time 包中定义的时间间隔类型的常量如下：

```go
const (
    Nanosecond  Duration = 1
    Microsecond          = 1000 * Nanosecond
    Millisecond          = 1000 * Microsecond
    Second               = 1000 * Millisecond
    Minute               = 60 * Second
    Hour                 = 60 * Minute
)
```

`time.Duration`表示1纳秒，`time.Second`表示1秒。

然后了解以下操作：

|                函数                |                     表意                     |
| :--------------------------------: | :------------------------------------------: |
| func (t Time) Add(d Duration) Time |        一个时间加若干单位时间后的时间        |
| func (t Time) Sub(u Time) Duration |              返回一个时间段t-u               |
|  func (t Time) Equal(u Time) bool  |             判断两个时间是否相同             |
| func (t Time) Before(u Time) bool  | 如果t代表的时间点在u之前，返回真；否则返回假 |
|  func (t Time) After(u Time) bool  |       func (t Time) After(u Time) bool       |



> #### 定时器



定时器实质是单向通道，time.Timer结构体类型中有一个time.Time类型的单向channel

```go
type Timer struct {
    C <-chan Time
    r runtimeTimer
}
```

**time.NewTimer:** 

初始化 Timer 方法为NewTimer

```go
t := time.NewTimer(time.Second * 2)
	defer t.Stop()
	for {
		<-t.C
		fmt.Println("timer running...")
		// 需要重置Reset 使 t 重新开始计时
		t.Reset(time.Second * 2)
    }
```

这里使用NewTimer定时器需要t.Reset重置计数时间才能接着执行

defer t.Stop()在这里并不会停止定时器。这是因为Stop会停止Timer，停止后，Timer不会再被发送，但是Stop不会关闭通道，防止读取通道发生错误。
如果想停止定时器，只能让go程序自动结束。

**time.NewTicker:**

ticker只要定义完成后，不需要其他操作就可以定时执行。
这里的defer t.Stop()和上面示例相似，也不会停止定时器，解决办法一样。定时器的本质上是一个通道（channel）

```go
t := time.NewTicker(time.Second*2)
defer t.Stop()
for {
    <- t.C
    fmt.Println("Ticker running...")
}	
```

**time.After:**

time.After()表示多长时间长的时候后返回一条`time.Time`类型的通道消息。但是在取出channel内容之前不阻塞，后续程序可以继续执行。

```go
func After(d Duration) <-chan Time {
   return NewTimer(d).C
}
```

```go
t := time.After(time.Second * 3)
   fmt.Printf("t type=%T\n", t)
   //阻塞3秒
   fmt.Println("t=", <-t)
```





> #### 时间格式化

`time.Format`函数能够将一个时间对象格式化输出为指定布局的文本表示形式，需要注意的是 Go 语言中时间格式化的布局不是常见的`Y-m-d H:M:S`，而是使用 `2006-01-02 15:04:05.000`（记忆口诀为2006 1 2 3 4 5）

- 如果想格式化为12小时格式，需在格式化布局中添加`PM`。
- 小数部分想保留指定位数就写0，如果想省略末尾可能的0就写 9





## 4 strconv

strconv包实现了基本数据类型与其字符串表示的转换



> #### string与int类型互相转换：Atoi,Itoa

```go
func Atoi(s string) (i int, err error)
func Itoa(i int) string
```



> #### string与给定类型互相转换：Parse

```go
func ParseBool(str string) (value bool, err error)
func ParseInt(s string, base int, bitSize int) (i int64, err error)
func ParseUint(s string, base int, bitSize int) (n uint64, err error)
func ParseFloat(s string, bitSize int) (f float64, err error)
```

- `ParseUint`类似`ParseInt`但不接受正负号，用于无符号整型
- base指定进制（2到36），如果base为0，则会从字符串前置判断，”0x”是16进制，”0”是8进制，否则是10进制
- bitSize指定结果必须能无溢出赋值的整数类型，0、8、16、32、64 分别代表 int、int8、int16、int32、int64；
- 返回的err是*NumErr类型的，如果语法有误，err.Error = ErrSyntax；如果结果超出类型范围err.Error = ErrRange



> #### 给定类型与string互相转换：Format

```go
func FormatBool(b bool) string
func FormatInt(i int64, base int) string
func FormatUint(i uint64, base int) string
func FormatUint(i uint64, base int) string
func FormatFloat(f float64, fmt byte, prec, bitSize int) string
```

- bitSize表示f的来源类型（32：float32、64：float64），会据此进行舍入



## 5 log



> #### 使用logger

log包定义了Logger类型，该类型提供了一些格式化输出的方法。本包也提供了一个预定义的“标准”logger对象，可以通过调用函数print系列，fatal系列，panic系列来使用，比自行创建一个logger对象更容易使用。

例如下，通过log包调用上面提供的方法：

```go
log.Println("这是一条很普通的日志。")//2017/06/19 14:04:17 这是一条很普通的日志。
v := "很普通的"
log.Printf("这是一条%s日志。\n", v)//2017/06/19 14:04:17 这是一条很普通的日志。
log.Fatalln("这是一条会触发fatal的日志。")//2017/06/19 14:04:17 这是一条会触发fatal的日志。
log.Panicln("这是一条会触发panic的日志。")
```

**注意点：logger会打印每条日志信息的日期、时间，默认输出到系统的标准错误。Fatal系列函数会在写入日志信息后调用os.Exit(1)。Panic系列函数会在写入日志信息后panic**



> #### 配置logger

   标准的logger配置：默认情况下的logger只会提供日志的时间信息，但是很多情况下我们希望得到更多信息，比如记录该日志的文件名和行号等。`log`标准库中为我们提供了定制这些设置的方法。

```go
func Flags() int//返回标准logger的输出配置
func SetFlags(flag int)//设置标准logger的输出配置
func Prefix() string//查看标准logger的输出前缀
func SetPrefix(prefix string)//用来设置输出前缀
func SetOutput(w io.Writer)//设置标准logger的输出目的地，默认是标准错误输出
```

> ##### flag选项

```go
const (
    // 控制输出日志信息的细节，不能控制输出的顺序和格式。
    // 输出的日志在每一项后会有一个冒号分隔：例如2009/01/23 01:23:23.123123 /a/b/c/d.go:23: message
    Ldate         = 1 << iota     // 日期：2009/01/23
    Ltime                         // 时间：01:23:23
    Lmicroseconds                 // 微秒级别的时间：01:23:23.123123（用于增强Ltime位）
    Llongfile                     // 文件全路径名+行号： /a/b/c/d.go:23
    Lshortfile                    // 文件名+行号：d.go:23（会覆盖掉Llongfile）
    LUTC                          // 使用UTC时间
    LstdFlags     = Ldate | Ltime // 标准logger的初始值
)
```

```go
func main() {
	log.SetFlags(log.Llongfile | log.Lmicroseconds | log.Ldate)
	log.Println("这是一条很普通的日志。")//2017/06/19 14:05:17.494943 .../log_demo/main.go:11: 这是一条很普通的日志。
    log.SetPrefix("[小王子]")
	log.Println("这是一条很普通的日志。")//[小王子]2017/06/19 14:05:57.940542 .../log_demo/main.go:13: 这是一条很普通的日志。
    logFile, err := os.OpenFile("./xx.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
	if err != nil {
		fmt.Println("open log file failed, err:", err)
		return
	}
	log.SetOutput(logFile)
    log.Println("这是一条很普通的日志。")
}
```

**注意点：如果你要使用标准的logger，我们通常会把上面的配置操作写到`init`函数中**

```go
func init() {
	logFile, err := os.OpenFile("./xx.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
	if err != nil {
		fmt.Println("open log file failed, err:", err)
		return
	}
	log.SetOutput(logFile)
	log.SetFlags(log.Llongfile | log.Lmicroseconds | log.Ldate)
}
```



> #### 创建logger

不使用预定义对象，使用自己创建的logger对象：`log`标准库中还提供了一个创建新logger对象的构造函数–`New`，支持我们创建自己的logger示例

```go
func New(out io.Writer, prefix string, flag int) *Logger//参数out设置日志信息写入的目的地。参数prefix会添加到生成的每一条日志前面。参数flag定义日志的属性（时间、文件等等）,返回一个logger对象
```

例如：

```go
func main() {
	logger := log.New(os.Stdout, "<New>", log.Lshortfile|log.Ldate|log.Ltime)
	logger.Println("这是自定义的logger记录的日志。")//<New>2017/06/19 14:06:51 main.go:34: 这是自定义的logger记录的日志。
}
```



## 6 net/http

​     

​     go语言内置的net/http包就提供了http客户端和服务端的实现。http（超文本传输协议）最初的目的是提供一种发布和接收HTML页面的方法。

​     我们的浏览器其实就是一个发送和接收HTTP协议数据的客户端，我们平时通过浏览器访问网页其实就是从网站的服务器接收HTTP数据，然后浏览器会按照HTML、CSS等规则将网页渲染展示出来。



### 6.1 http客户端



> ##### 客户端发出http/https请求

​      Get、Head、Post和PostForm函数发出HTTP/HTTPS请求。**注意点：这些函数也可以带参数，具体用法可查询标准库**

```go
//右边发出请求，得到左边的响应
resp, err := http.Get("http://example.com/")
...
resp, err := http.Post("http://example.com/upload", "image/jpeg", &buf)
...
resp, err := http.PostForm("http://example.com/form",
	url.Values{"key": {"Value"}, "id": {"123"}})
```

**注意点：程序在使用完response后必须关闭相应的主体**

```go
resp, err := http.Get("http://example.com/")
if err != nil {
	// handle error
}
defer resp.Body.Close()
body, err := ioutil.ReadAll(resp.Body)
```



### 6.2 http服务端



> ##### 默认的Server

ListenAndServe函数使用指定的监听地址和处理器启动一个HTTP服务端。处理器参数通常是nil，这表示采用包变量DefaultServeMux作为处理器。

Handle和HandleFunc函数可以向DefaultServeMux添加处理器

```go
http.Handle("/foo", fooHandler)
http.HandleFunc("/bar", func(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello, %q", html.EscapeString(r.URL.Path))
})
log.Fatal(http.ListenAndServe(":8080", nil))
```

示例如下：

```go
// http server

func sayHello(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "Hello 沙河！")
}

func main() {
	http.HandleFunc("/", sayHello)
	err := http.ListenAndServe(":9090", nil)
	if err != nil {
		fmt.Printf("http server failed, err:%v\n", err)
		return
	}
}
```



## 7 http/template

   `html/template`包实现了数据驱动的模板，用于生成可防止代码注入的安全的HTML内容,Go语言中输出HTML的场景都应使用`html/template`这个包。

### 7.1 模板与渲染



​    在一些前后端不分离的Web架构中，我们通常需要在后端将一些数据渲染到HTML文档中，从而实现动态的网页（网页的布局和样式大致一样，但展示的内容并不一样）效果。

​    我们这里说的模板可以理解为事先定义好的HTML文档文件，模板渲染的作用机制可以简单理解为文本替换操作–使用相应的数据去替换HTML文档中事先准备好的标记。

​     很多编程语言的Web框架中都使用各种模板引擎，比如Python语言中Flask框架中使用的jinja2模板引擎。



### 7.2 Go的模板引擎



Go语言内置的文本模板引擎text/template和html文档模板引擎html/template。作用机制如下：

1. 模板文件，后缀通常为.tmpl和.tpl。使用utf8编码
2. 模板文件中使用{{}}包裹和标识需要传入的数据
3. 传给模板的数据通过（.）来访问。如果是复杂类型数据，通过{{.fieldname}}来访问它的字段
4. 除{{}}包裹的内容，其他内容不做修改原样输出



### 7.3 模板引擎使用



**使用步骤：1. 定义模板文件   2. 解析模板文件  3. 渲染模板文件**



> ##### 定义



```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Hello</title>
</head>
<body>
    <p>Hello {{.}}</p>
</body>
</html>
```



> ##### 解析



```go
func (t *Template) Parse(src string) (*Template, error)//使用func New(name string) *Template函数创建一个名为name的模板，然后对其调用上面的方法去解析模板字符串或模板文件，这里Parse函数的参数实际上就是要解析的文件的字符串形式：Parse(string(通过read方法读取tmpl文件))，具体用法见下面自定义函数的语法
func ParseFiles(filenames ...string) (*Template, error)
func ParseGlob(pattern string) (*Template, error)//解析多个模板文件
```

**易错点：解析多个模板文件一定要注意用ParseGlob，文件路径名：文件夹用两个星号，文件用一个星号来代替**



> ##### 渲染



```go
func (t *Template) Execute(wr io.Writer, data interface{}) error
func (t *Template) ExecuteTemplate(wr io.Writer, name string, data interface{}) error
```



解析和渲染示例如下：

```go
// main.go

func sayHello(w http.ResponseWriter, r *http.Request) {
	// 解析指定文件生成模板对象
	tmpl, err := template.ParseFiles("./hello.tmpl")
	if err != nil {
		fmt.Println("create template failed, err:", err)
		return
	}
	// 利用给定数据渲染模板，并将结果写入w
	tmpl.Execute(w, "沙河小王子")
}
func main() {
	http.HandleFunc("/", sayHello)//作用：注册了请求/的路由规则。第一个参数变量就是一个路由器，它用来匹配url跳转到其相应的handle函数
	err := http.ListenAndServe(":9090", nil)
	if err != nil {
		fmt.Println("HTTP server failed,err:", err)
		return
	}
}
```



### 7.4 模板语法



> ##### {{.}}

模板语法都包含在`{{`和`}}`中间，其中`{{.}}`中的点表示当前对象。

当我们传入一个结构体对象时，我们可以根据`.`来访问结构体的对应字段。**注意点：这里结构体字段名一定要首字母大写，因为外部要访问**



> ##### 注释

{{/*注释内容*/}}

**注意点：可以多行注释，但必须紧贴分界符始止，不能嵌套**



> ##### pipeline

**注意点：是产生数据的操作，传递数据，只要能产生数据的都是pipeline.**



> ##### 变量

可以在模板中声明变量，用来保存传入模板的数据或其它语句生成的结果，这样就可以在后续代码中使用该变量了

```template
$obj := {{.}}//后续在代码中使用obj这个变量名就行了
```



> ##### 移除空格

使用`{{-`语法去除模板内容左侧的所有空白符号， 使用`-}}`去除模板内容右侧的所有空白符号

```template
{{- .Name -}}
```

**注意点：-要紧靠{{和}}，同时与模板值之间要使用空格**



> ##### 条件判断



> ##### range



> ##### with



> ##### 预定义函数



> ##### 比较函数



> ##### 自定义函数

```go
func sayHello(w http.ResponseWriter, r *http.Request) {
          //1.定义模板
    //2.解析模板
    a,_:=ioutil.ReadFile("tmpl文件路径名")
    function:=func(参数列表)(返回值列表){}//自定义函数，要么有一个返回值，要么有两个返回值，第二个返回值必须是error类型
    b,_:=template.New("同上tmpl文件路径名")//创建了一个这样的模板，然后用parse方法解析
    b.Funcs(template.FuncMap{"function":function})//在解析模板之前，一定要告诉模板引擎，我现在多了一个自定义的函数kua
    b.Parse(string(a))//这里如果用ParseFile的解析方法，那参数直接为“tmpl文件路径名”
    //3.渲染模板
    b.Execute(w,结果)
}
```

**使用：在模板文件`hello.tmpl`中按照如下方式使用我们自定义的`kua`函数了：{{kua .Name}}**



> ##### 嵌套template



> ##### 继承template



> ##### 修改默认的标识符



## 8  Context

**引言知识点：控制并发有三种经典的方式，一种是WaitGroup，一种是channel结合select，另外一种就是Context**

WaitGroup:这种方式是控制多个goroutine同时完成.

在实际的业务中，我们可能会有这么一种场景：需要我们**主动的优雅的**通知某一个goroutine结束。比如我们开启一个后台goroutine一直做事情，比如监控，现在不需要了，就需要通知这个监控goroutine结束，不然它会一直跑，就泄漏了。

### 8.1 why

举例说明：如何**主动的优雅的**结束子协程？除开goroutine 自己跑完结束退出有定义全局变量和结构体通道结合select的方法。而**二者的弊端**为：使用全局变量在跨包调用时不容易实现规范和统一，可能很多goroutine需要维护一个共用的channel。

```go
//全局变量方式：使用全局变量在跨包调用时不容易统一，如果worker中再启动goroutine，就不太好控制了。
var exit bool
func worker() {
	for {
		fmt.Println("worker")
		time.Sleep(time.Second)
		if exit {
			break
		}
	}
	wg.Done()
}
func main() {
	wg.Add(1)
	go worker()
	time.Sleep(time.Second * 3) // sleep3秒以免程序过快退出
	exit = true                 // 修改全局变量实现子goroutine的退出
	wg.Wait()
	fmt.Println("over")
}

//channel方式 存在的问题：1. 使用全局变量在跨包调用时不容易实现规范和统一，需要维护一个共用的channel
func worker(exitChan chan struct{}) {
	LOOP:
		for {
			fmt.Println("worker")
			time.Sleep(time.Second)
			select {
			case <-exitChan: // 等待接收上级通知
				break LOOP
			default:
			}
		}
		wg.Done()
	}

	func main() {
		var exitChan = make(chan struct{})
		wg.Add(1)
		go worker(exitChan)
		time.Sleep(time.Second * 3) // sleep3秒以免程序过快退出
		exitChan <- struct{}{}      // 给子goroutine发送退出信号
		close(exitChan)
		wg.Wait()
		fmt.Println("over")
	}
```

**优雅的关闭：用context方法：**

```go
//使用Context：当子goroutine又开启另外一个goroutine时，只需要将ctx传入即可
func worker(ctx context.Context) {
	go worker2(ctx)
LOOP:
	for {
		fmt.Println("worker")
		time.Sleep(time.Second)
		select {
		case <-ctx.Done(): // 等待上级通知
			break LOOP
		default:
		}
	}
	wg.Done()
}

func worker2(ctx context.Context) {
LOOP:
	for {
		fmt.Println("worker2")
		time.Sleep(time.Second)
		select {
		case <-ctx.Done(): // 等待上级通知
			break LOOP
		default:
		}
	}
}
func main() {
	ctx, cancel := context.WithCancel(context.Background())
	wg.Add(1)
	go worker(ctx)
	time.Sleep(time.Second * 3)
	cancel() // 通知子goroutine结束
	wg.Wait()
	fmt.Println("over")
}

```



### 8.2 what

标准库`context`，它定义了`Context`类型，专门用来简化 对于处理单个请求的多个 goroutine 之间传递上下文信息,如:数据、取消信号、截止时间等相关操作，这些操作可能涉及多个 API 调用。

![image-20211122110839075](markdown%E5%9B%BE%E7%89%87/image-20211122110839075.png)

对服务器传入的请求应该创建上下文，而对服务器的传出调用应该接受上下文。它们之间的函数调用链必须传递上下文，或者可以使用`WithCancel`、`WithDeadline`、`WithTimeout`或`WithValue`创建的派生上下文。当一个上下文被取消时，它派生的所有上下文也被取消。

**总结：context 用来解决 goroutine 之间`退出通知`、`元数据传递`的功能。是 goroutine 的上下文，包含 goroutine 的运行状态、环境、现场等信息。**



```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)//返回当前Context被取消的时间，也就是完成工作的截止时间
    Done() <-chan struct{}//返回一个Channel，这个Channel会在当前工作完成或者上下文被取消之后关闭，多次调用Done方法会返回同一个Channel；
    Err() error//返回当前Context结束的原因，它只会在Done返回的Channel被关闭时才会返回非空的值；
    Value(key interface{}) interface{}//从Context中返回键对应的值，对于同一个上下文来说，多次调用Value 并传入相同的Key会返回相同的结果，该方法仅用于传递跨API和进程间跟请求域的数据；
}
```

> ##### Background()和TODO（）函数

Go内置两个函数：`Background()`和`TODO()`，这两个函数分别返回一个实现了`Context`接口的`background`和`todo`。我们代码中最开始都是以这两个内置的上下文对象作为最顶层的`partent context`，衍生出更多的子上下文对象。

`Background()`主要用于main函数、初始化以及测试代码中，作为Context这个树结构的最顶层的Context，也就是根Context。

`TODO()`，它目前还不知道具体的使用场景，如果我们不知道该使用什么Context的时候，可以使用这个。

`background`和`todo`本质上都是`emptyCtx`结构体类型，是一个不可取消，没有设置截止时间，没有携带任何值的Context。

### 8.3 With系列函数

> #### WithCancel

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)//WithCancel返回带有新Done通道的父节点的副本。当调用返回的cancel函数或当关闭父上下文的Done通道时，将关闭返回上下文的Done通道，无论先发生什么情况。

//取消此上下文将释放与其关联的资源，因此代码应该在此上下文中运行的操作完成后立即调用cancel。
```

```go
func main() {
	ctx, clear := context.WithCancel(context.Background())
	message := make(chan int)
	go son(ctx, message)
	for i := 0; i < 10; i++ {
		message <- i
	}
	fmt.Println("1")
	clear() //// 当我们弄完之后调用cancel
	fmt.Println("2")
	time.Sleep(time.Second)
	fmt.Println("主进程结束")
}
func son(ctx context.Context, msg chan int) {
	t := time.Tick(time.Second)
	for _ = range t {
		select {
		case m := <-msg:
			fmt.Printf("收到值%d\n", m)
		case <-ctx.Done(): //Done方法需要返回一个Channel，这个Channel会在当前工作完成或者上下文被取消之后关闭
			fmt.Println("结束")
			return // return结束该goroutine，防止泄露
		}
	}
}

```



> #### WithDeadline

```go
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)//返回父上下文的副本，并将deadline调整为不迟于d。当截止日过期时，当调用返回的cancel函数时，或者当父上下文的Done通道关闭时，返回上下文的Done通道将被关闭，以最先发生的情况为准。  取消此上下文将释放与其关联的资源，因此代码应该在此上下文中运行的操作完成后立即调用cancel。
```

```go
func main() {
	d := time.Now().Add(50 * time.Microsecond) //定义了一个50毫秒之后过期的deadline
	ctx, cancel := context.WithDeadline(context.Background(), d)
	defer cancel()
	select { //让主程序陷入等待：等待1秒后打印overslept退出或者等待ctx过期后退出。
	case <-time.After(time.Second):
		fmt.Println("overslepy")
	case <-ctx.Done(): //ctx 50毫秒后就会过期，所以ctx.Done()会先接收到context到期通知，并且会打印ctx.Err()的内容。
		fmt.Println(ctx.Err())
	}
}
```



> #### WithTimeout

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

```go
var wg sync.WaitGroup
func worker(ctx context.Context) {
LOOP:
	for {
		fmt.Println("db connecting ...")
		time.Sleep(time.Millisecond * 10) // 假设正常连接数据库耗时10毫秒
		select {
		case <-ctx.Done(): // 50毫秒后自动调用
			break LOOP
		default:
		}
	}
	fmt.Println("worker done!")
	wg.Done()
}
func main() {
	// 设置一个50毫秒的超时
	ctx, cancel := context.WithTimeout(context.Background(), time.Millisecond*50)
	wg.Add(1)
	go worker(ctx)
	time.Sleep(time.Second * 5)
	cancel() // 通知子goroutine结束
	wg.Wait()
	fmt.Println("over")
}
```



>#### WithValue

```go
func WithValue(parent Context, key, val interface{}) Context
```

注意点：1. 所提供的键key必须是可比较的，并且不应该是`string`类型或任何其他内置类型，以避免使用上下文在包之间发生冲突,`WithValue`的用户应该为键定义自己的类型。



### 8.4 Context使用注意事项

1. 推荐以参数的方式显示传递Context
2. 以Context作为参数的函数方法，应该把Context作为第一个参数
3. 给一个函数方法传递Context参数的时候，不要传递nil，不知道传递什么就传context.Background()
4. Context是线程安全的，可以放心在多个goroutine中传递
5. Context的Value相关方法应该传递请求域的必要数据，不应该用于传递可选参数



## 9 flag 

Go语言内置的`flag`包实现了命令行参数的解析，`flag`包使得开发命令行工具更为简单

> #### init函数常用定义命令行参数方法

1. `flag.Type(flag名, 默认值, 帮助信息)*Type`**注意点：返回的是对应类型的指针**

   ```go
   name := flag.String("name", "张三", "姓名")
   age := flag.Int("age", 18, "年龄")
   married := flag.Bool("married", false, "婚否")
   delay := flag.Duration("d", 0, "时间间隔")
   ```

2. `flag.TypeVar(Type指针，flag名, 默认值, 帮助信息)*Type`

   ```go
   var name string
   var age int
   var married bool
   var delay time.Duration
   flag.StringVar(&name, "name", "张三", "姓名")
   flag.IntVar(&age, "age", 18, "年龄")
   flag.BoolVar(&married, "married", false, "婚否")
   flag.DurationVar(&delay, "d", 0, "时间间隔")
   ```

> #### 主函数解析参数及一些其它函数

通过以上两种方法定义好命令行flag参数后，需要通过调用`flag.Parse()`来对命令行参数进行解析

```go
flag.Args()  ////返回命令行参数后的其他参数，以[]string类型
flag.NArg()  //返回命令行参数后的其他参数个数
flag.NFlag() //返回使用的命令行参数个数
```

​	示例：

```go
func main() {
	//定义命令行参数方式1
	var name string
	var age int
	var married bool
	var delay time.Duration
	flag.StringVar(&name, "name", "张三", "姓名")
	flag.IntVar(&age, "age", 18, "年龄")
	flag.BoolVar(&married, "married", false, "婚否")
	flag.DurationVar(&delay, "d", 0, "延迟的时间间隔")

	//解析命令行参数
	flag.Parse()
	fmt.Println(name, age, married, delay)
	//返回命令行参数后的其他参数
	fmt.Println(flag.Args())
	//返回命令行参数后的其他参数个数
	fmt.Println(flag.NArg())
	//返回使用的命令行参数个数
	fmt.Println(flag.NFlag())
}
```



## 10 sort

sort包的底层实现是多种排序的算法，例如快排，插入等等。调用时并不公开，也无需定义用那种算法。
sort包内部会根据实际情况，自动选择最高效的排序算法。



> #### 实现

- **原理：**

  ​	 1. 如果掌握多种底层排序算法的话，应该可以提炼出排序算法的三要素，即序列长度，两个元素比较结果，两个元素交换方式。
  ​     2. go语言sort包就是这么做的，提供三个方法，用来实现三要素。分别是Len，Less,Swap
  ​     3. 调用的时候不需要查看底层如何实现，只需要用户定义出三要素，go语言就会按照底层最高效的算法按照用户定义的三要素对序列进行排序

          4. 在实际工程开发中，对string，int等排序应该非常多。sort包提供了更加便捷的方法，不用用户自己定义三要素，可直接调用进行排序。

- **实现要求：**

  为了让sort包能识别序列，对序列进行排序，就必须让序列实现sort.interface{}接口。

  只有实现了sort.interface{}接口的序列，sort包才能进行排序，就是排序三要素，Len，Less，Swap。这三个方法,然后调用sort.Sort()方法，可根据定义对序列最终排序。

  sort包定义了sort.Reverse()方法，对当前定义的排序规则取反，也就是反向排序。

- **实例：**

```go
//定义序列类型，只要此类型实现了sort.interface{}接口(实现三要素方法)，就可以对此类型排序
type TestStringList []string

//元素个数
func (t TestStringList) Len() int {
    return len(t)
}

//比较结果
func (t TestStringList) Less(i, j int) bool {
    return t[i] < t[j]
}

//交换方式
func (t TestStringList) Swap(i, j int) {
    t[i], t[j] = t[j], t[i]
}

func SortPackageDemo() {

    stringList := TestStringList{
        "1-php",
        "2-java",
        "3-golang",
        "4-c",
        "5-cpp",
        "6-python",
    }

    //按照定义的规则排序
    sort.Sort(stringList)
    fmt.Println(stringList)
    //返回值
    //[1-php 2-java 3-golang 4-c 5-cpp 6-python]

    //按照定义的规则反向排序
    sort.Sort(sort.Reverse(stringList))
    fmt.Println(stringList)
    //返回值
    //[6-python 5-cpp 4-c 3-golang 2-java 1-php]
}

注意：这种排序方式，对任何实现了sort.interface{}接口的序列，都可以排序
```

​    

> #### sort包常见便捷排序

Go 1.8之后的版本，Go语言在 sort 包中提供了 sort.Slice() 函数进行更为简便的排序方法。  sort.Slice() 函数只要求传入需要排序的数据，以及一个排序时对元素的回调函数。

```go
sort.Slice(tmp,func(i,j int){
        return tmp[i]<tmp[j]
}
```

**实例：**

```go
func demoSortSlice(){
    a := []int{6,3,9,8,1,2,5,7}
    sort.Slice(a, func(i, j int) bool {
        return a[i]>a[j]
    })
    fmt.Println(a)
    //[9 8 7 6 5 3 2 1]
}
```



## 11 runtime

#### 11.1 前言知识

Go 语言的 goroutine 是由 运行时（runtime）调度和管理的。尽管 Go 编译器产生的是本地可执行代码，这些代码仍旧运行在 Go 的 runtime（这部分的代码可以在 runtime 包中找到）当中。Go 语言的 runtime 类似 Java 和 .NET 语言所用到的虚拟机，它负责管理包括内存分配、垃圾回收、栈处理、goroutine、channel、切片（slice）、map 和反射（reflection）等等。

**函数调用栈：**

我们按照编程语言的语法定义的函数，会被编译器编译为一堆堆机器指令，写入可执行文件。程序执行时可执行文件被加载到内存，这些机器指令对应到虚拟地址空间中，位于代码段。
如果在一个函数中调用另一个函数，编译器就会对应生成一条**call**指令，程序执行到这条指令时，就会跳转到被调用函数入口处开始执行，而每个函数的最后都有一条**ret**指令，负责在函数结束后跳回到调用处，继续执行。

**函数栈帧：**

函数执行时需要有足够的内存空间，供它存放局部变量、参数等数据，这段空间对应到虚拟地址空间的栈。栈，只有一个口可供进出，先入栈的在底，后入栈的在顶，最后入栈的最早被取出。运行时栈，上面是高地址，向下增长，栈底通常被称为**“栈基”**，栈顶被称为**“栈指针”**。![image-20220307160402855](markdown%E5%9B%BE%E7%89%87/image-20220307160402855.png)

分配给函数的栈空间被称为**“函数栈帧”**，Go语言中函数栈帧布局是这样的，先是调用者栈基地址，然后是函数的局部变量，最后是被调用函数的返回值和参数。

![image-20220307160441736](markdown%E5%9B%BE%E7%89%87/image-20220307160441736.png)

BP of callee和SP of callee标识被调用函数执行时，栈基寄存器和栈指针寄存器指向的位置，但是注意“BP of caller”不一定会存在，有些情况下可能会被优化掉，也有可能是平台不支持

举个例子：

```
func A() {   
    var a1, a2, r1, r2 int64   
    a1, a2 = 1, 2  
    r1, r2 = B(a1, a2)   
    r1 = C(a1)   
    println(r1, r2)
}
func B(p1, p2 int64) (int64, int64) {  
    return p2, p1
}
func C(p1 int64) int64 { 
    return p1
}
```

函数A的栈帧布局如下图所示，局部变量之后的空间用于存放被调用函数的返回值和参数，接下来要调用函数B，所以先有两个int64类型的变量空间用作B的返回值，再有两个int64类型的变量空间用于存放传递给B的参数

![image-20220307161042542](markdown%E5%9B%BE%E7%89%87/image-20220307161042542.png)

注意观察参数的顺序，先入栈第二个参数，再入栈第一个参数，返回值也是一样，上面是第二个返回值的空间，然后才是第一个返回值的空间。
其实这也好解释，因为这些是被调用函数的返回值和参数，被调用函数是通过栈指针加上偏移值这样相对寻址的方式来定位到自己的参数和返回值的，这样由下至上正好先找到第一个参数，再找到第二个参数。所以参数和返回值采用由右至左的入栈顺序比较合适。

**Go语言中，函数栈帧是一次性分配的**，也就是在函数开始执行的时候分配足够大的栈帧空间。就像上例中函数A一样，它要调用两个函数，除了调用者栈基地址、局部变量以外，再有四个int64的空间用作被调用函数的参数与返回值就足够了。
一次性分配函数栈帧的主要原因是避免栈访问越界，如下图所示，三个goroutine初始分配的栈空间是一样的，如果g2剩余的栈空间不够执行接下来的函数，若函数栈帧是逐步扩张的，那么执行期间就可能发生栈访问越界。



#### 11.2 具体函数用法



```go
func Stack(buf []byte, all bool) int
```

Stack将调用其的go程的调用栈踪迹格式化后写入到buf中并返回写入的字节数。若all为true，函数会在写入当前go程的踪迹信息后，将其它所有go程的调用栈踪迹都格式化写入到buf中。



```go
func Caller(skip int) (pc uintptr, file string, line int, ok bool)
```

`Caller`报告当前go程调用栈所执行的函数的文件和行号信息。实参`skip`为上溯的[栈帧](https://so.csdn.net/so/search?q=栈帧&spm=1001.2101.3001.7020)数，0表示`Caller`的调用者，当前函数（`Caller`所在的调用栈），1表示上一层函数。第一个返回值代表的是程序计数（pc）；第二个和第三个参数代表对应函数所在的源文件名以及所在行数，这里我们暂时不需要；最后一个参数代表是否能成功获取这些信息，如果获取失败，我们抛出 panic。

根据pc获取函数名：根据返回的调用栈标识符`pc`，我们可以获取对应的函数名。需要用到一个函数和一个方法，签名如下：

```go
func FuncForPC(pc uintptr) *Func

func (*Func) Name
```





## 12 strings



```go
func Fields(str string) []string
```

返回值：str的子字符串切片或如果str仅包含空格的空白切片。



```go
func TrimPrefix(s, prefix string) string
```

返回：s中如果有prefix前缀则去掉后返回



## 13 bytes



```go
func IndexByte(s []byte, c byte) int
```

字符c在s中第一次出现的位置，不存在则返回-1。



```go
func TrimPrefix(s, prefix []byte) []byte
```

返回去除s可能的前缀prefix的子切片。（共用底层数组）



- bytes.buffer:是一个缓冲byte类型的**缓冲器**

1·、使用bytes.NewBuffer创建：参数是[]byte的话，缓冲器里就是这个slice的内容；如果参数是nil的话，就是创建一个空的缓冲器。

2、bytes.NewBufferString创建

3、bytes.Buffer{} 

```go
func (b *Buffer) Bytes() []byte { return b.buf[b.off:] }
```





#                                                     