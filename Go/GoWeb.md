#                                             Ⅴ 数据库学习

---

---

## 1 Mysql



### 1.1 连接数据库

`database/sql`包提供了保证SQL或类SQL数据库的泛用接口，并不提供具体的数据库驱动。使用`database/sql`包时必须注入（至少）一个数据库驱动。

> ##### 下载驱动依赖

```bash
go get -u github.com/go-sql-driver/mysql
```

> ##### 使用驱动

```go
func Open(driverName, dataSourceName string) (*DB, error)
```

```go
// 定义一个全局对象db
var db *sql.DB

// 定义一个初始化数据库的函数
func initDB() (err error) {
	dsn := "user:password@tcp(127.0.0.1:3306)/sql_test?charset=utf8mb4&parseTime=True"
	db, err = sql.Open("mysql", dsn)//注意！！！这里不要使用:=，我们是给全局变量赋值，然后在main函数中使用全局变量db。Open函数可能只是验证其参数格式是否正确，实际上并不创建与数据库的连接。如果要检查数据源的名称是否真实有效，应该调用Ping方法。
	if err != nil {
		return err
	}
	err = db.Ping()// 尝试与数据库建立连接（校验dsn是否正确）
	if err != nil {
		return err
	}
	return nil
}

func main() {
	err := initDB() // 调用输出化数据库的函数
	if err != nil {
		fmt.Printf("init db failed,err:%v\n", err)
		return
	}
}
```

**注意点：返回的DB对象可以安全地被多个goroutine并发使用，并且维护其自己的空闲连接池。因此，Open函数应该仅被调用一次，很少需要关闭这个DB对象。**

**易错点:主逻辑中使用 defer 最后关闭连接是没有问题的。如果是在模块化开发，我们把一些数据库连接的操作封装到函数中，就不能带有 defer。由于 defer 是在逻辑的最后执行的，实际上到 return 的时候，返回的已经是 null**

> ##### SetMaxOpenConns

```go
func (db *DB) SetMaxOpenConns(n int)//设置与数据库建立连接的最大数目
```

> ##### SetMaxIdleConns

```go
func (db *DB) SetMaxIdleConns(n int)//设置连接池中的最大闲置连接数
```



### 1.2 CRUD



>##### 查询

单行查询：`func (db *DB) QueryRow(query string, args ...interface{}) *Row`返回一行结果（即Row）参数args表示query中的占位参数。

```go
// 查询单条数据示例
func queryRowDemo() {
	sqlStr := "select id, name, age from user where id=?"
	var u user
	// 非常重要：确保QueryRow之后调用Scan方法，否则持有的数据库链接不会被释放
	err := db.QueryRow(sqlStr, 1).Scan(&u.id, &u.name, &u.age)
	if err != nil {
		fmt.Printf("scan failed, err:%v\n", err)
		return
	}
	fmt.Printf("id:%d name:%s age:%d\n", u.id, u.name, u.age)
}
```

多行查询：`func (db *DB) Query(query string, args ...interface{}) (*Rows, error)`执行一次查询，返回多行结果（即Rows），一般用于执行select命令。参数args表示query中的占位参数。

```go
// 查询多条数据示例
func queryMultiRowDemo() {
	sqlStr := "select id, name, age from user where id > ?"
	rows, err := db.Query(sqlStr, 0)
	if err != nil {
		fmt.Printf("query failed, err:%v\n", err)
		return
	}
	// 非常重要：关闭rows释放持有的数据库链接
	defer rows.Close()

	// 循环读取结果集中的数据
	for rows.Next() {
		var u user
		err := rows.Scan(&u.id, &u.name, &u.age)
		if err != nil {
			fmt.Printf("scan failed, err:%v\n", err)
			return
		}
		fmt.Printf("id:%d name:%s age:%d\n", u.id, u.name, u.age)
	}
}
```



**插入、更新和删除操作都使用`Exec`方法。`func (db *DB) Exec(query string, args ...interface{}) (Result, error)`返回的Result是对已执行的SQL命令的总结。参数args表示query中的占位参数。**

> ##### 插入

```go
// 插入数据
func insertRowDemo() {
	sqlStr := "insert into user(name, age) values (?,?)"
	ret, err := db.Exec(sqlStr, "王五", 38)
	if err != nil {
		fmt.Printf("insert failed, err:%v\n", err)
		return
	}
	theID, err := ret.LastInsertId() // 新插入数据的id
	if err != nil {
		fmt.Printf("get lastinsert ID failed, err:%v\n", err)
		return
	}
	fmt.Printf("insert success, the id is %d.\n", theID)
}
```

> ##### 更新

```go
// 更新数据
func updateRowDemo() {
	sqlStr := "update user set age=? where id = ?"
	ret, err := db.Exec(sqlStr, 39, 3)
	if err != nil {
		fmt.Printf("update failed, err:%v\n", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Printf("get RowsAffected failed, err:%v\n", err)
		return
	}
	fmt.Printf("update success, affected rows:%d\n", n)
}
```

> ##### 删除

```go
// 删除数据
func deleteRowDemo() {
	sqlStr := "delete from user where id = ?"
	ret, err := db.Exec(sqlStr, 3)
	if err != nil {
		fmt.Printf("delete failed, err:%v\n", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Printf("get RowsAffected failed, err:%v\n", err)
		return
	}
	fmt.Printf("delete success, affected rows:%d\n", n)
}
```



### 1.3 MYSQL预处理



> ##### what

普通SQL语句执行过程：

1. 客户端对SQL语句进行占位符替换得到完整的SQL语句。
2. 客户端发送完整SQL语句到MySQL服务端
3. MySQL服务端执行完整的SQL语句并将结果返回给客户端。

预处理执行过程：

1. 把SQL语句分成两部分，命令部分与数据部分。

2. 先把命令部分发送给MySQL服务端，MySQL服务端进行SQL预处理。

3. 然后把数据部分发送给MySQL服务端，MySQL服务端对SQL语句进行占位符替换。

4. MySQL服务端执行完整的SQL语句并将结果返回给客户端。

   

> ##### why

1. 优化MySQL服务器重复执行SQL的方法，可以提升服务器性能，提前让服务器编译，一次编译多次执行，节省后续编译的成本。

2. 避免SQL注入问题。

   

> ##### sql注入问题

**我们任何时候都不应该自己拼接SQL语句！**

```go
// 一个自行拼接SQL语句的示例，编写一个根据name字段查询user表的函数
func sqlInjectDemo(name string) {
	sqlStr := fmt.Sprintf("select id, name, age from user where name='%s'", name)
	..........
}
```

输入以下字符串都会引发注入问题：

```go
sqlInjectDemo("xxx' or 1=1#")
sqlInjectDemo("xxx' union select * from user #")
sqlInjectDemo("xxx' and (select count(*) from user) <10 #")
```



> ##### how

`database/sql`中使用下面的`Prepare`方法来实现预处理操作,返回一个准备好的状态用于之后的查询和命令。返回值可以同时执行多个查询和命令。

```go
func (db *DB) Prepare(query string) (*Stmt, error)
```

```go
// 预处理查询示例
func prepareQueryDemo() {
	sqlStr := "select id, name, age from user where id > ?"
	stmt, err := db.Prepare(sqlStr)
	if err != nil {
		fmt.Printf("prepare failed, err:%v\n", err)
		return
	}
	defer stmt.Close()
	rows, err := stmt.Query(0)
	if err != nil {
		fmt.Printf("query failed, err:%v\n", err)
		return
	}
	defer rows.Close()
	// 循环读取结果集中的数据
	for rows.Next() {
		var u user
		err := rows.Scan(&u.id, &u.name, &u.age)
		if err != nil {
			fmt.Printf("scan failed, err:%v\n", err)
			return
		}
		fmt.Printf("id:%d name:%s age:%d\n", u.id, u.name, u.age)
	}
}
```

```go
// 预处理插入示例
func prepareInsertDemo() {
	sqlStr := "insert into user(name, age) values (?,?)"
	stmt, err := db.Prepare(sqlStr)
	if err != nil {
		fmt.Printf("prepare failed, err:%v\n", err)
		return
	}
	defer stmt.Close()
	_, err = stmt.Exec("小王子", 18)
	if err != nil {
		fmt.Printf("insert failed, err:%v\n", err)
		return
	}
	_, err = stmt.Exec("沙河娜扎", 18)
	if err != nil {
		fmt.Printf("insert failed, err:%v\n", err)
		return
	}
	fmt.Println("insert success.")
}
```



### 1.4 事务



> ##### what

一个最小的不可再分的工作单元；通常一个事务对应一个完整的业务(例如银行账户转账业务，该业务就是一个最小的工作单元)，同时这个完整的业务需要执行多次的DML(insert、update、delete)语句共同联合完成。A转账给B，这里面就需要执行两次update操作。

**注意点：在MySQL中只有使用了`Innodb`数据库引擎的数据库或表才支持事务。事务处理可以用来维护数据库的完整性，保证成批的SQL语句要么全部执行，要么全部不执行。**



> ##### 事务的特性

![image-20211121205040761](markdown%E5%9B%BE%E7%89%87/image-20211121205040761.png)



> ##### how

```go
// 开始事务 func (db *DB) Begin() (*Tx, error)
// 提交事务 func (tx *Tx) Commit() error
// 回滚事务 func (tx *Tx) Rollback() error
```

```go
// 事务操作示例
func transactionDemo() {
	tx, err := db.Begin() // 开启事务
	if err != nil {
		if tx != nil {
			tx.Rollback() // 回滚
		}
		fmt.Printf("begin trans failed, err:%v\n", err)
		return
	}
	sqlStr1 := "Update user set age=30 where id=?"
	ret1, err := tx.Exec(sqlStr1, 2)
	if err != nil {
		tx.Rollback() // 回滚
		fmt.Printf("exec sql1 failed, err:%v\n", err)
		return
	}
	affRow1, err := ret1.RowsAffected()
	if err != nil {
		tx.Rollback() // 回滚
		fmt.Printf("exec ret1.RowsAffected() failed, err:%v\n", err)
		return
	}

	sqlStr2 := "Update user set age=40 where id=?"
	ret2, err := tx.Exec(sqlStr2, 3)
	if err != nil {
		tx.Rollback() // 回滚
		fmt.Printf("exec sql2 failed, err:%v\n", err)
		return
	}
	affRow2, err := ret2.RowsAffected()
	if err != nil {
		tx.Rollback() // 回滚
		fmt.Printf("exec ret1.RowsAffected() failed, err:%v\n", err)
		return
	}

	fmt.Println(affRow1, affRow2)
	if affRow1 == 1 && affRow2 == 1 {
		fmt.Println("事务提交啦...")
		tx.Commit() // 提交事务
	} else {
		tx.Rollback()
		fmt.Println("事务回滚啦...")
	}

	fmt.Println("exec trans success!")
}
```



## 2 Mysql进阶



### 2.1 连接数据库

> ##### 安装sqlx

```go
go get github.com/jmoiron/sqlx
```

> ##### 连接

```go
var db *sqlx.DB

func initDB() (err error) {
	dsn := "user:password@tcp(127.0.0.1:3306)/sql_test?charset=utf8mb4&parseTime=True"
	// 也可以使用MustConnect连接不成功就panic
	db, err = sqlx.Connect("mysql", dsn)
	if err != nil {
		fmt.Printf("connect DB failed, err:%v\n", err)
		return
	}
	db.SetMaxOpenConns(20)
	db.SetMaxIdleConns(10)
	return
}
```

### 2.2 CRUD

**注意点：sqlx相比sql实现了对数据的批量处理**

> ##### 查询

单行：

```go
// 查询单条数据示例
func queryRowDemo() {
	sqlStr := "select id, name, age from user where id=?"
	var u user
	err := db.Get(&u, sqlStr, 1)
	if err != nil {
		fmt.Printf("get failed, err:%v\n", err)
		return
	}
	fmt.Printf("id:%d name:%s age:%d\n", u.ID, u.Name, u.Age)
}
```

多行：

```go
// 查询多条数据示例
func queryMultiRowDemo() {
	sqlStr := "select id, name, age from user where id > ?"
	var users []user
	err := db.Select(&users, sqlStr, 0)
	if err != nil {
		fmt.Printf("query failed, err:%v\n", err)
		return
	}
	fmt.Printf("users:%#v\n", users)
}
```

NamedQuery: 

```go
func namedQuery(){
	sqlStr := "SELECT * FROM user WHERE name=:name"
	// 使用map做命名查询
	rows, err := db.NamedQuery(sqlStr, map[string]interface{}{"name": "七米"})
	if err != nil {
		fmt.Printf("db.NamedQuery failed, err:%v\n", err)
		return
	}
	defer rows.Close()
	for rows.Next(){
		var u user
		err := rows.StructScan(&u)
		if err != nil {
			fmt.Printf("scan failed, err:%v\n", err)
			continue
		}
		fmt.Printf("user:%#v\n", u)
	}

	u := user{
		Name: "七米",
	}
	// 使用结构体命名查询，根据结构体字段的 db tag进行映射
	rows, err = db.NamedQuery(sqlStr, u)
	if err != nil {
		fmt.Printf("db.NamedQuery failed, err:%v\n", err)
		return
	}
	defer rows.Close()
	for rows.Next(){
		var u user
		err := rows.StructScan(&u)
		if err != nil {
			fmt.Printf("scan failed, err:%v\n", err)
			continue
		}
		fmt.Printf("user:%#v\n", u)
	}
}
```



> 插入，更新，删除

```go
// 插入数据
func insertRowDemo() {
	sqlStr := "insert into user(name, age) values (?,?)"
	ret, err := db.Exec(sqlStr, "沙河小王子", 19)
	if err != nil {
		fmt.Printf("insert failed, err:%v\n", err)
		return
	}
	theID, err := ret.LastInsertId() // 新插入数据的id
	if err != nil {
		fmt.Printf("get lastinsert ID failed, err:%v\n", err)
		return
	}
	fmt.Printf("insert success, the id is %d.\n", theID)
}

// 更新数据
func updateRowDemo() {
	sqlStr := "update user set age=? where id = ?"
	ret, err := db.Exec(sqlStr, 39, 6)
	if err != nil {
		fmt.Printf("update failed, err:%v\n", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Printf("get RowsAffected failed, err:%v\n", err)
		return
	}
	fmt.Printf("update success, affected rows:%d\n", n)
}

// 删除数据
func deleteRowDemo() {
	sqlStr := "delete from user where id = ?"
	ret, err := db.Exec(sqlStr, 6)
	if err != nil {
		fmt.Printf("delete failed, err:%v\n", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Printf("get RowsAffected failed, err:%v\n", err)
		return
	}
	fmt.Printf("delete success, affected rows:%d\n", n)
}
```

NamedExec:

```go
func insertUserDemo()(err error){
	sqlStr := "INSERT INTO user (name,age) VALUES (:name,:age)"
	_, err = db.NamedExec(sqlStr,
		map[string]interface{}{
			"name": "七米",
			"age": 28,
		})
	return
}
```

### 2.3 批量CRUD

> ##### 自行拼接语句构造批量插入

```go
// BatchInsertUsers 自行构造批量插入的语句
func BatchInsertUsers(users []*User) error {
	// 存放 (?, ?) 的slice
	valueStrings := make([]string, 0, len(users))
	// 存放values的slice
	valueArgs := make([]interface{}, 0, len(users) * 2)
	// 遍历users准备相关数据
	for _, u := range users {
		// 此处占位符要与插入值的个数对应
		valueStrings = append(valueStrings, "(?, ?)")
		valueArgs = append(valueArgs, u.Name)
		valueArgs = append(valueArgs, u.Age)
	}
	// 自行拼接要执行的具体语句
	stmt := fmt.Sprintf("INSERT INTO user (name, age) VALUES %s",
		strings.Join(valueStrings, ","))
	_, err := DB.Exec(stmt, valueArgs...)
	return err
}
```

>##### sqlx.In的批量插入

**前提：结构体需要实现`driver.Valuer`接口：**

```go
func (u User) Value() (driver.Value, error) {
	return []interface{}{u.Name, u.Age}, nil
}
```

```go
// BatchInsertUsers2 使用sqlx.In帮我们拼接语句和参数, 注意传入的参数是[]interface{}
func BatchInsertUsers2(users []interface{}) error {
	query, args, _ := sqlx.In(
		"INSERT INTO user (name, age) VALUES (?), (?), (?)",
		users..., // 如果arg实现了 driver.Valuer, sqlx.In 会通过调用 Value()来展开它
	)
    fmt.Println(query) // 查看生成的querystring INSERT INTO user (name, age) VALUES (?, ?), (?, ?), (?, ?)
	fmt.Println(args)  // 查看生成的args  [七米 18 q1mi 28 小王子 38]
	_, err := DB.Exec(query, args...)
	return err
}
```



> ##### 使用NamedExec实现批量插入

```go
// BatchInsertUsers3 使用NamedExec实现批量插入
func BatchInsertUsers3(users []*User) error {
	_, err := DB.NamedExec("INSERT INTO user (name, age) VALUES (:name, :age)", users)
	return err
}
```

```go
func main() {
	err := initDB()
	if err != nil {
		panic(err)
	}
	defer DB.Close()
	u1 := User{Name: "七米", Age: 18}
	u2 := User{Name: "q1mi", Age: 28}
	u3 := User{Name: "小王子", Age: 38}

	// 方法1
	users := []*User{&u1, &u2, &u3}
	err = BatchInsertUsers(users)
	if err != nil {
		fmt.Printf("BatchInsertUsers failed, err:%v\n", err)
	}

	// 方法2
	users2 := []interface{}{u1, u2, u3}
	err = BatchInsertUsers2(users2)
	if err != nil {
		fmt.Printf("BatchInsertUsers2 failed, err:%v\n", err)
	}

	// 方法3
	users3 := []*User{&u1, &u2, &u3}
	err = BatchInsertUsers3(users3)
	if err != nil {
		fmt.Printf("BatchInsertUsers3 failed, err:%v\n", err)
	}
}
```

> #####  sqlx.In的查询示例



### 2.4 事务

```go
func transactionDemo2()(err error) {
	tx, err := db.Beginx() // 开启事务
	if err != nil {
		fmt.Printf("begin trans failed, err:%v\n", err)
		return err
	}
	defer func() {
		if p := recover(); p != nil {
			tx.Rollback()
			panic(p) // re-throw panic after Rollback
		} else if err != nil {
			fmt.Println("rollback")
			tx.Rollback() // err is non-nil; don't change it
		} else {
			err = tx.Commit() // err is nil; if Commit returns error update err
			fmt.Println("commit")
		}
	}()

	sqlStr1 := "Update user set age=20 where id=?"

	rs, err := tx.Exec(sqlStr1, 1)
	if err!= nil{
		return err
	}
	n, err := rs.RowsAffected()
	if err != nil {
		return err
	}
	if n != 1 {
		return errors.New("exec sqlStr1 failed")
	}
	sqlStr2 := "Update user set age=50 where i=?"
	rs, err = tx.Exec(sqlStr2, 5)
	if err!=nil{
		return err
	}
	n, err = rs.RowsAffected()
	if err != nil {
		return err
	}
	if n != 1 {
		return errors.New("exec sqlStr1 failed")
	}
	return err
}
```

## 3 MongoDB





## 4 Redis

## Web

---

---

### 0 Web基础原理讲解

#### 0.1 web工作方式

> #### 普通上网过程：

1. 浏览器作为一个客户端，输入url后，会去请求DNS服务器获得相应域名对应的服务器IP地址
2. 找到服务器后发送建立TCP连接信息，同意连接后，浏览器发送http request请求。
3. 服务器接收到请求包后开始调用自身服务，返回http response相应包。
4. 客户端受到响应包后开始渲染这个相应包里面的主体，随后断开与服务器之间的TCP连接。

一个Web服务器也被称为HTTP服务器，它通过HTTP协议与客户端通信。这个客户端通常指的是Web浏览器(其实手机端客户端内部也是浏览器实现的)。

> #### Web服务器的工作原理可以简单地归纳为：

- 客户机通过TCP/IP协议建立到服务器的TCP连接
- 客户端向服务器发送HTTP协议请求包，请求服务器里的资源文档
- 服务器向客户机发送HTTP协议应答包，如果请求的资源包含有动态语言的内容，那么服务器会调用动态语言的解释引擎负责处理“动态内容”，并将处理得到的数据返回给客户端
- 客户机与服务器断开。由客户端解释HTML文档，在客户端屏幕上渲染图形结果

**注意点：客户机与服务器之间的通信是非持久连接的，也就是当服务器发送了应答后就与客户机断开连接，等待下一次请求**



#### 0.2 How does go make the web work



> #### 服务端概念



Request：用户请求的信息，用来解析用户的请求信息，包括post、get、cookie、url等信息

Response：服务器需要反馈给客户端的信息

Conn：用户的每次请求链接

Handler：处理请求和生成返回信息的处理逻辑



> #### http包运行机制



http包执行流程：

1. 创建Listen Socket，监听指定端口，等待客户端请求connect到来后，接收客户端请求accept得到client socket。
2. 通过client socket与客户端通信。从client socket读取http请求的协议头。如果是post方法，还可能要读取客户端提交的数据然后交给相应的handler处理请求，handler处理完毕后准备好客户端需要的数据，通过client socket写给客户端

![image-20211129162658680](markdown%E5%9B%BE%E7%89%87/image-20211129162658680.png)

> #### socket建立流程

![image-20211130154450515](markdown%E5%9B%BE%E7%89%87/image-20211130154450515.png)

**重点：net/http全套服务：**

1. 注册路由

   ```
   http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
   })
   ```

2. ln, err := net.Listen(“tcp”, addr)做了初试化了socket, bind, listen的操作.

3. rw, e := l.Accept()进行accept, 等待客户端进行连接

4. go c.serve(ctx) 启动新的goroutine来处理本次请求. 同时主goroutine继续等待客户端连接, 进行高并发操作

5. h, _ := mux.Handler(r) 获取注册的路由, 然后拿到这个路由的handler, 然后将处理结果返回给客户端



>  #### question：

1. 如何监听端口？http.ListenAndServe

2. 如何接收客户端请求？http.ListenAndServe内部原理。**注意点：这里内部就有高并发体现了，用户的每一次请求都是在一个新的goroutine去服务**

3. 如何分配handler？http.ListenAndServe内部原理。Handle和HandleFunc函数可以向DefaultServeMux添加处理器，就是注册处理路由。

   ![image-20211129164805053](markdown%E5%9B%BE%E7%89%87/image-20211129164805053.png)

### 1 Gin

#### 1.1 初识Gin框架

> 下载安装Gin

```bash
go get -u github.com/gin-gonic/gin
```

> First demo

```go
//为什么使用http/net包就可以实现web服务端，框架的作用就是搭好这个舞台供你来实现
func sayHello(c *gin.Context) {
	c.JSON(200, gin.H{
		"message": "hello golang",
	})
}
func main() {
	r := gin.Default()        //返回默认的路由引擎
	r.GET("/hello", sayHello) //指定用户使用GET请求访问helllo时，执行sayHello函数
	//启动服务
	r.Run(":9090") //默认8080端口
}
```



#### 1.2 RESTful API

REST与技术无关，代表的是一种软件架构风格，REST是Representational State Transfer的简称。Gin框架支持开发RESTful API的开发。

简单来说，REST的含义就是客户端与Web服务器之间进行交互的时候，使用HTTP协议中的4个请求方法代表不同的动作。只要API程序遵循了REST风格，那就可以称其为RESTful API。目前在前后端分离的架构中，前后端基本都是通过RESTful API来进行交互。

- `GET`用来获取资源
- `POST`用来新建资源
- `PUT`用来更新资源
- `DELETE`用来删除资源。



#### 1.3 Gin渲染

##### HTML渲染

**易错点：多个模板模板解析,注意用r.LoadHTMLGlob**

```go
r:=gin.Default()	
r.LoadHTMLGlob("./templates/**/*")           //多个模板模板解析,注意用loadhtmlglob
r.GET("/posts/index", func(c *gin.Context) {
		//http请求
		c.HTML(http.StatusOK, "posts/index.tmpl", gin.H{//模板渲染
			"title": "posts/index",
		})
	})
r.GET("/users/index", func(c *gin.Context) { 
		//http请求
		c.HTML(http.StatusOK, "users/index.tmpl", gin.H{
			"title": "users/index",
		})
	})
r.Run()
```

> #### 自定义模板函数渲染

> ##### 定义模板：

```template
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <title>修改模板引擎的标识符</title>
</head>
<body>
<div>{{ . | safe }}</div>
</body>
</html>
```

> ##### 解析，渲染模板：

```go
r：=gin.Default()	
r.SetFuncMap(template.FuncMap{ //设置自定义函数，让模板知道有这个函数,一定要放在解析模板前面，顺序不能错！！！！！
		"curry": func(str string) template.HTML {
			return template.HTML(str)
		},
	})
	r.Static("/xxx", "./static")                 //一定要在解析模板前声明加载静态文件：html页面上用到的样式文件，css，js 图片
	r.LoadHTMLFiles("./templates/diyindex.tmpl") //单个模板解析
	r.GET("/diyindex", func(c *gin.Context) {
		c.HTML(http.StatusOK, "diyindex.tmpl", "<a href='https://liwenzhou.com'>李文周的博客</a>") //模板渲染之自定义函数渲染加上静态文件！！！！！！
	})
r.Run()
```

![image-20211123100043155](markdown%E5%9B%BE%E7%89%87/image-20211123100043155.png)

> #### 静态文件处理

**易错点：一定要在解析模板前加载静态文件**

```go
func main() {
	r := gin.Default()
	r.Static("/static", "./static")
	r.LoadHTMLGlob("templates/**/*")
   // ...
	r.Run(":8080")
}
```

> #### 补充文件路径处理

关于模板文件和静态文件的路径，我们需要根据公司/项目的要求进行设置。可以使用下面的函数获取当前执行程序的路径。

```go
func getCurrentPath() string {
	if ex, err := os.Executable(); err == nil {
		return filepath.Dir(ex)
	}
	return "./"
}
```

> #### 使用模板继承



#####  JSON渲染

**注意点： gin.H 是map[string]interface{}的缩写**

```go
r := gin.Default()
r.GET("/json01", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"message": "hello world! map"}) //方式一，自己拼接json类型map,gin.H实质就是一个map,json渲染方式一
	})
r.GET("/json02", func(c *gin.Context) {
		var msg struct { //字段首字母必须大写
			Name    string `json:"name"` //要想返回结构体字段的小写，就要给结构体字段写tag,通过指定tag实现json序列化该字段时的key,做定制化操作
			Message string
			Age     int
		}
		msg.Name = "王坤"
		msg.Message = "hello stephen"
		msg.Age = 18
		c.JSON(http.StatusOK, msg) //方式二，使用结构体渲染json
	})
r.Run(":8080")
```

#####  XML渲染

**易错点:使用结构体的话注意需要使用具名的结构体类型**

```go
	r.GET("/someXML", func(c *gin.Context) {
		// 方式一：自己拼接JSON
		c.XML(http.StatusOK, gin.H{"message": "Hello world!"})
	})
	r.GET("/moreXML", func(c *gin.Context) {
		// 方法二：使用结构体
		type MessageRecord struct {
			Name    string
			Message string
			Age     int
		}
		var msg MessageRecord //注意需要使用具名的结构体类型
		msg.Name = "小王子"
		msg.Message = "Hello world!"
		msg.Age = 18
		c.XML(http.StatusOK, msg)
	})
```

##### YAMAL渲染

```go
r.GET("/someYAML", func(c *gin.Context) {
	c.YAML(http.StatusOK, gin.H{"message": "ok", "status": http.StatusOK})
})
```

##### protobuf渲染





#### 1.4 获取参数



#####  获取querystring参数

`querystring`指的是URL中`?`后面携带的参数，例如：`/user/search?username=小王子&address=沙河`，通过Query()或DefaultQuery()方法获得。 获取请求的querystring参数的方法如下：

```go
	r.GET("/getquery", func(c *gin.Context) {//获取querystring参数！！！！！！！
		// name:=c.Query("query")//第一种方式获取浏览器那边发请求携带的querystring参数,通过query来获取携带的参数
		// name := c.DefaultQuery("query", "wangkun") //第二种方式通过query来获取携带的参数，取不到就去后面的默认值
		age := c.DefaultQuery("age", "18")
		name, ok := c.GetQuery("query") //第三种方式，根据取不到，第二个就返回布尔值
		if !ok {
			name = "wangkun"
		}
		c.JSON(http.StatusOK, gin.H{
			"name": name,
			"age":  age,
		})
	})
```



#####  获取form参数

当前端请求的数据通过form表单提交时，一次请求对应一次响应 获取form参数，表单参数可以通过PostForm()方法获取

```go
	r.LoadHTMLGlob("./templates/form/*")
	r.GET("/login", func(c *gin.Context) {
		c.HTML(http.StatusOK, "login.html", nil)
	})
	r.POST("/login", func(c *gin.Context) { //获取form表单用post提交请求方式！！！！！！
		//username := c.DefaultPostForm("username", "库里")// DefaultPostForm取不到值时会返回指定的默认值
		username := c.PostForm("username")
		password := c.PostForm("password")
		c.HTML(http.StatusOK, "index.html", gin.H{ //输出json结果给调用方
			"Name":     username,
			"Password": password,
		})
	})
```

```html
<!DOCTYPE html>
<head>
    <title>登陆页面</title>
</head>
<body>
<form action="/login" method="post" novalidate autocomplete="off">
    <div>
        <label for="username">username:</label>
        <input type="text" name="username" id="username">
    </div>
    <div>
        <label for="password">password:</label>
        <input type="password" name="password" id="password">
    </div>
    <div>
       
        <input type="submit" value="登录">
    </div>
</form>
</body>
```

```html
<!DOCTYPE html>
<head>
<title>跳转页面</title>
</head>
<body>
<h1>hello,{{ .Name }}!</h1>
<p>你的密码是：{{ .Password }}</p>
</body>
```



#####  获取json参数

当前端请求的数据通过JSON提交时。**注意点与form的形式不同**

```go
r.POST("/json", func(c *gin.Context) {
	// 注意：下面为了举例子方便，暂时忽略了错误处理
	b, _ := c.GetRawData()  // 从c.Request.Body读取请求数据
	// 定义map或结构体
	var m map[string]interface{}
	// 反序列化
	_ = json.Unmarshal(b, &m)

	c.JSON(http.StatusOK, m)
})
```



##### 获取path参数

请求的参数通过URL路径传递，例如：`/user/search/小王子/沙河`,通过context的Param方法获得。 **注意点：区别于querystring的不同**

```go
	r.GET("/path/:name/:age/*action", func(c *gin.Context) { //获取path参数!!!!!!
		name := c.Param("name")
		age := c.Param("age")
        action := c.Param("action")
		c.JSON(http.StatusOK, gin.H{
			"name": name,
			"age":  age,
            "action":  action,
		})
	})
```



##### 参数绑定

.ShouldBind()的强大功能：它能够基于请求自动提取`JSON`、`form表单`和`QueryString`类型的数据，并把值绑定到指定的结构体对象。

```go
type Login struct {
		User     string `form:"user" json:"user" binding:"required"`//因为这里c.ShouldBind传递的只是结构体名称的地址（因为涉及修改所以必须传递地址），并不知道具体的字段对应的取值，所以要用反射，
		Password string `form:"password" json:"password" binding:"required"`
	}
//绑定querystring参数示例
	r.GET("/bindquerystring", func(c *gin.Context) {//querystring参数绑定
		var l Login
		err := c.ShouldBind(&l)
		if err == nil {
			c.JSON(http.StatusOK, gin.H{
				"user":     l.User,
				"password": l.Password,
			})
		} else {
			c.JSON(http.StatusOK, gin.H{
				"error": err.Error(),
			})
		}
	})

//绑定form表单示例
r.LoadHTMLGlob("./templates/form/*")
	r.GET("/bindlogin", func(c *gin.Context) {
		c.HTML(http.StatusOK, "login.html", nil)
	})
	r.POST("/login", func(c *gin.Context) {
		var l Login
		err := c.ShouldBind(&l)
		if err == nil {
			c.JSON(http.StatusOK, gin.H{
				"user":     l.User,
				"password": l.Password,
			})
		} else {
			c.JSON(http.StatusOK, gin.H{
				"error": err.Error(),
			})
		}
	})
// 绑定JSON的示例 ({"user": "q1mi", "password": "123456"})
	r.POST("/loginJSON", func(c *gin.Context) {
		var l Login
		if err := c.ShouldBind(&l); err == nil {
			fmt.Printf("login info:%#v\n", login)
			c.JSON(http.StatusOK, gin.H{
				"user":     l.User,
				"password": l.Password,
			})
		} else {
			c.JSON(http.Status
                   BadRequest, gin.H{"error": err.Error()})
		}
	})

```



#### 1.5 文件上传



##### 单个文件上传

```go
r.LoadHTMLFiles("./templates/file/index.html")
	r.GET("/file", func(c *gin.Context) {
		c.HTML(http.StatusOK, "index.html", nil)
	})
	r.POST("/upload", func(c *gin.Context) { //上传一个文件
		file, _ := c.FormFile("f1") //从请求中读文件，参数和对应html中定义的一样
		//将读取的文件保存到本地（服务端本地），首先要制定保存地址，有两种方式
		// dst:=fmt.Sprintf("./%s",file.Filename)
		dst := path.Join("./", file.Filename)
		_ = c.SaveUploadedFile(file, dst)
		c.JSON(http.StatusOK, gin.H{
			"status": "ok",
		})
	})
```



##### 多个文件上传

```go
r.MaxMultipartMemory = 8 << 20
	r.LoadHTMLFiles("./templates/file/multiindex.html")
	r.GET("/file", func(c *gin.Context) {
		c.HTML(http.StatusOK, "multiindex.html", nil)
	})
	r.POST("/upload", func(c *gin.Context) { //上传多个文件
		form, _ := c.MultipartForm() //从请求中读多个文件，无参数
		files := form.File["file"]
		//将读取的文件循环保存到本地（服务端本地），再循环体内制定保存地址
		for index, file := range files {

			dst := fmt.Sprintf("./%s_%d", file.Filename, index)
			_ = c.SaveUploadedFile(file, dst)
		}
		c.JSON(http.StatusOK, gin.H{
			"status": "ok",
		})
	})
```



#### 1.6 重定向

```go
//对内，外
r.GET("/cross", func(c *gin.Context) { //内部、外部重定向均支持
		c.Redirect(http.StatusMovedPermanently, "http://www.sogo.com")
	})
//路由重定向
	r.GET("/a", func(c *gin.Context) {
		c.Request.URL.Path = "/b" //指定重定向的地址
		r.HandleContext(c)        //，转向指定的路由
	})
	r.GET("/b", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"status": "ok",
		})
	})
```



#### 1.7 Gin路由



##### 普通路由

路由：给个地址就给你条路

不同情况的路由：Get(),Post(),Put(),Delete(),Head(),Any()，NoRoute()

```go
r.GET("/index", func(c *gin.Context) {...})
r.GET("/login", func(c *gin.Context) {...})
r.POST("/login", func(c *gin.Context) {...})
r.Any("/test", func(c *gin.Context) {...})//可以匹配所有请求方法的Any方法
r.NoRoute(func(c *gin.Context) {//为没有配置处理函数的路由添加处理程序，默认情况下它返回404代码，为没有匹配到路由的请求都返回views/404.html页面
		c.HTML(http.StatusNotFound, "views/404.html", nil)
	})
```



##### 路由组

路由组：拥有共同URL前缀的路由划分为一个路由组。习惯性一对{}包裹同组的路由，这只是为了看着清晰，你用不用{}包裹功能上没什么区别。

```go
func main() {
	r := gin.Default()
	shopGroup := r.Group("/shop")
	{
		shopGroup.GET("/index", func(c *gin.Context) {...})
		shopGroup.GET("/cart", func(c *gin.Context) {...})
		shopGroup.POST("/checkout", func(c *gin.Context) {...})
	}
	r.Run()
}
```

同时路由组也支持嵌套：

```go
shopGroup := r.Group("/shop")
	{
		shopGroup.GET("/index", func(c *gin.Context) {...})
		shopGroup.GET("/cart", func(c *gin.Context) {...})
		shopGroup.POST("/checkout", func(c *gin.Context) {...})
		// 嵌套路由组
		xx := shopGroup.Group("xx")
		xx.GET("/oo", func(c *gin.Context) {...})
	}
```

#####  路由拆分与注册

> #### 路由拆分成单独文件或包

文件目录：

```bash
gin_demo
├── go.mod
├── go.sum
├── main.go
└── routers.go
```

```bash
gin_demo
├── go.mod
├── go.sum
├── main.go
└── routers
    └── routers.go
```

```go
package routers

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func helloHandler(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{
		"message": "Hello q1mi!",
	})
}

// SetupRouter 配置路由信息
func SetupRouter() *gin.Engine {
	r := gin.Default()
	r.GET("/hello", helloHandler)
	return r
}
```

```go
package main

import (
	"fmt"
	"gin_demo/routers"
)

func main() {
	r := routers.SetupRouter()
	if err := r.Run(); err != nil {
		fmt.Println("startup service failed, err:%v\n", err)
	}
}
```



> #### 路由拆分成多个文件

```bash
gin_demo
├── go.mod
├── go.sum
├── main.go
└── routers
    ├── blog.go
    └── shop.go
```

```go
func main() {
	r := gin.Default()
	routers.LoadBlog(r)
	routers.LoadShop(r)
	if err := r.Run(); err != nil {
		fmt.Println("startup service failed, err:%v\n", err)
	}
}
```

```go
func LoadBlog(e *gin.Engine) {
	e.GET("/post", postHandler)
  e.GET("/comment", commentHandler)
  ...
}
```

```go
func LoadShop(e *gin.Engine)  {
	e.GET("/hello", helloHandler)
  e.GET("/goods", goodsHandler)
  e.GET("/checkout", checkoutHandler)
  ...
}
```

> #### 路由分到不同app

```bash
gin_demo
├── app
│   ├── blog
│   │   ├── handler.go
│   │   └── router.go
│   └── shop
│       ├── handler.go
│       └── router.go
├── go.mod
├── go.sum
├── main.go
└── routers
    └── routers.go
```





#### 1.8 Gin中间件



##### 定义中间件

定义中间件，Gin中的中间件必须是一个gin.HandlerFunc类型，一般用闭包形式，因为在执行中间件之前还可以执行一些其他操作

```go
func m1(doCheck bool) gin.HandlerFunc {
	//执行一些其他准备工作
	return func(c *gin.Context) {
		if doCheck {
			c.JSON(http.StatusOK, gin.H{
				"msa": "ok",
			})
		}
	}
}
func m2() gin.HandlerFunc { //统计请求处理函数的耗时的中间件
	//执行一些其他准备工作
	return func(c *gin.Context) {
		fmt.Println("m1 in")
		start := time.Now() //开始计时
		c.Next()            //调用后续处理函数，这里直接去别的函数了，千万注意！！！！
		//c.Abort()//阻止调用后续的处理函数
		cost := time.Since(start)
		fmt.Printf("cost:%v\n", cost)
		fmt.Println("m1 out")
	}
}
func m3() gin.HandlerFunc { //统计请求处理函数的耗时的中间件
	//执行一些其他准备工作
	return func(c *gin.Context) {
		fmt.Println("m2 in")
		c.Next() //调用后续处理函数，这里直接去别的函数了，千万注意！！！！
		fmt.Println("m2 out")
	}
}

```



##### 注册中间件

1. 为全局路由注册中间件:`r.Use(StatCost())`
2. 为单个路由注册中间件：`r.get("XXX",中间件1，中间件2.....) `**注意点：{有中间件执行先后顺序}**
3. 为路由组注册中间件：

```go
//写法一
shopGroup := r.Group("/shop", StatCost())
{
    shopGroup.GET("/index", func(c *gin.Context) {...})
    ...
}
```

```go
//写法二
shopGroup := r.Group("/shop")
shopGroup.Use(StatCost())
{
    shopGroup.GET("/index", func(c *gin.Context) {...})
    ...
}
```



##### 中间件注意事项

1. `gin.Default()`默认使用了`Logger`和`Recovery`中间件，其中：

   - `Logger`中间件将日志写入`gin.DefaultWriter`，即使配置了`GIN_MODE=release`。

   - `Recovery`中间件会recover任何`panic`。如果有panic的话，会写入500响应码。

     如果不想使用上面两个默认的中间件，可以使用`gin.New()`新建一个没有任何默认中间件的路由。

2. gin中间件中使用goroutine：

​        当在中间件或`handler`中启动新的`goroutine`时，**不能使用**原始的上下文（c *gin.Context），必须使用其只读副本（`c.Copy()`）。



##### 中间件相关函数

1. 在中间件函数中通过再次调用`c.Next()`实现嵌套调用

![gin_middleware2](markdown%E5%9B%BE%E7%89%87/gin_middleware2.png)

2. 调用`c.Abort()`中断整个调用链条，从当前函数返回
3. `c.Set()`和`c.Get()`这两个方法多用于在多个函数之间通过`c`传递数据的，比如我们可以在认证中间件中获取当前请求的相关信息（userID等）通过`c.Set()`存入`c`，然后在后续处理业务逻辑的函数中通过`c.Get()`来获取当前请求的用户![image-20211129191617334](markdown%E5%9B%BE%E7%89%87/image-20211129191617334.png)

```go
// 定义中间
func MiddleWare() gin.HandlerFunc {
    return func(c *gin.Context) {
        t := time.Now()
        fmt.Println("中间件开始执行了")
        // 设置变量到Context的key中，可以通过Get()取
        c.Set("request", "中间件")
        // 执行函数
        c.Next()
        // 中间件执行完后续的一些事情
        status := c.Writer.Status()
        fmt.Println("中间件执行完毕", status)
        t2 := time.Since(t)
        fmt.Println("time:", t2)
    }
}

func main() {
    // 1.创建路由
    // 默认使用了2个中间件Logger(), Recovery()
    r := gin.Default()
    // 注册中间件
    r.Use(MiddleWare())
    // {}为了代码规范
    {
        r.GET("/ce", func(c *gin.Context) {
            // 取值
            req, _ := c.Get("request")
            fmt.Println("request:", req)
            // 页面接收
            c.JSON(200, gin.H{"request": req})
        })

    }
    r.Run()
}
```

![img](markdown%E5%9B%BE%E7%89%87/m_e3ef8e82024ae4a638be74b1b4d8eb05_r.png)

#### 1.9 Cookie

##### why

http协议是无状态的，每次请求都是独立的。这里的状态可以理解为客户端和服务器在某次会话中产生的数据，无状态独立就表示这些数据不会被保留。那如何“保持状态呢”？

##### what

在Internet中，cookie指小量信息，由web服务器创建，将信息存储在客户端的数据文件。

网络用户习惯称为cookies，指某些网站为了辨别用户身份，进行session跟踪而存储在用户本地终端上的数据，这些数据通常会经过加密处理。

##### how

cookie是由服务器端生成，发送给user-agent（这里一般指浏览器），浏览器会将cookie的key/value保存到某个目录下的文本文件内，下次请求统一网站时就发送该cookie给服务器（这里前提是浏览器设置为启用cookie）。

cookie的key/value可以有服务器端开发自己定义，这样服务器可以知道该用户是否是合法用户以及是否需要重新登陆等，服务器可以设置或读取cookie中包含的信息，借此来维护用户跟服务器会话中的状态。

cookie的特点：

1. 浏览器发送请求时会自动携带该站点之前存储的cookie信息。
2. cookie数据可以配置过期时间，过期的cookie会被系统清除。
3. cookie是针对单个域名的，不同域名之间的cookie相互独立。
4. 服务端可以设置cookie数据。

##### Gin框架操作Cookie

模拟实现权限验证中间件：

- 有2个路由，login和home
- login用于设置cookie
- home是访问查看信息的请求
- 在请求home之前，先跑中间件代码，检验是否存在cookie

```go
//定义中间件
func MiddleWare() gin.HandlerFunc {
	return func(c *gin.Context) {
		cookie, err := c.Cookie("a") //获取客户端cookie并校验
		if err == nil {
			if cookie == "13" {
				c.Next()
				return
			}
		}
		c.JSON(http.StatusUnauthorized, gin.H{
			"error": "err",
		})
		c.Abort() // 若验证不通过，不再调用后续的函数处理
		return
	}
}
func main() {
	r := gin.Default() //1.创建路由
	r.GET("/login", func(c *gin.Context) {
		c.SetCookie("a", "13", 60, "/", "localhost", false, true) //有服务端给客户端设置cookie,这里注意域名为localhost，不要写成127.0.0.1
		c.String(200, "Login success!!!")
	})
	r.GET("/home", MiddleWare(), func(c *gin.Context) {
		c.JSON(200, gin.H{
			"data": "home",
		})
	})
	r.Run(":8080")
}

```



#### 1.10 Session

##### why

Cookie虽然保证了“保持状态”的需求，但缺点是：本身只最大支持4096字节，以及本身存在安全性。因此Session出来了，支持更多的字节，并且它保存在服务器，有较高的安全性。

##### what

用户登陆成功之后，我们在服务端为每个用户创建一个特定的session和一个唯一的标识，它们一一对应。其中：

- Session是在服务端保存的一个数据结构，用来跟踪用户的状态，这个数据可以保存在集群、数据库、文件中；
- 唯一标识通常称为`Session ID`会写入用户的Cookie中。

这样该用户后续再次访问时，请求会自动携带Cookie数据（其中包含了`Session ID`），服务器通过该`Session ID`就能找到与之对应的Session数据，也就知道来的人是“谁”。

总结而言：Cookie弥补了HTTP无状态的不足，让服务器知道来的人是“谁”；但是Cookie以文本的形式保存在本地，自身安全性较差；所以我们就通过Cookie识别不同的用户，对应的在服务端为每个用户保存一个Session数据，该Session数据中能够保存具体的用户数据信息。



### 2 GORM

#### 2.1 前言

ORM: Object Relational Mapping 程序中的对象或者实例和关系数据库之间的一种映射  优点：提高开发效率  缺点：牺牲执行性能和灵活性，弱化SQL能力

go中映射关系：1. 数据表---结构体 2. 数据行---结构体实例 3. 字段---结构体字段

#### 2.2连接MySQL

```go
import (
  "github.com/jinzhu/gorm"
  _ "github.com/jinzhu/gorm/dialects/mysql"//连接不同的数据库都需要导入对应数据的驱动程序，GORM已经贴心的为我们包装了一些驱动程序
)

func main() {
  db, err := gorm.Open("mysql", "user:password@(localhost)/dbname?charset=utf8mb4&parseTime=True&loc=Local")
  defer db.Close()
}
```

#### 2.3 GORM基本示例

```go
type UserInfo struct {
	Id   string
	Name string
	Age  int
}

func main() {
	db, err := gorm.Open("mysql", "root:123456@tcp(127.0.0.1:3306)/sql_test?charset=utf8mb4&parseTime=True") //连接mysql数据库
	if err != nil {
		panic(err)
	}
	defer db.Close()

	db.AutoMigrate(&UserInfo{}) //创建表 自动迁移 把结构体和数据表进行对应
	//创建
	u1 := UserInfo{"7", "wangkun", 22} //创建数据行
	db.Create(&u1)
	//查询
	var u UserInfo
	db.First(&u) //把查询的第一行数据给到u
	fmt.Println(u)
	var uu UserInfo
	db.Find(&uu, "id=?", "8") //把查询的第一行数据给到u
	fmt.Println(uu)
	//更新
	db.Model(&u).Update("name", "wk")
	//删除
	db.Delete(&u)
}
```

#### 2.4 GORM Model

   在使用orm工具时，我们要在代码中定义模型（Models）来与数据库中的数据表进行映射。在GORM中，模型通常是正常定义的结构体（结构体的字段就对应数据表中的字段，一个结构体实例就对应数据表中的一行）、基本go类型或它们的指针。

##### gorm.Model

 为方便模型定义，GORM内置了一个gorm.Model结构体。我们可以在构造自己的结构体的时候选择继承内置结构体或者自己完全定义一个模型。**注意点：gorm默认用Id作为主键。**

```go
// gorm.Model 定义
type Model struct {
  ID        uint `gorm:"primary_key"`
  CreatedAt time.Time
  UpdatedAt time.Time
  DeletedAt *time.Time
}
```

```go
// 不使用gorm.Model，自行定义模型
type User struct {
  ID   int
  Name string
}
```

##### 结构体标记（tags）

gorm支持以下tags(可选项)：

![image-20211123160419968](markdown%E5%9B%BE%E7%89%87/image-20211123160419968.png)

##### Model的约束

1. 主键：GORM 默认会使用名为ID的字段作为表的主键

   ```go
   type User struct {
     ID   string // 名为`ID`的字段会默认作为表的主键
     Name string
   }
   
   // 使用`AnimalID`作为主键
   type Animal struct {
     AnimalID int64 `gorm:"primary_key"`
     Name     string
     Age      int64
   }
   ```

2. 表名：默认就是结构体名称的复数

   ```go
   type User struct {} // 默认表名是 `users`
   
   // 将 User 的表名设置为 `profiles`
   func (User) TableName() string {
     return "profiles"
   }
   
   func (u User) TableName() string {
     if u.Role == "admin" {
       return "admin_users"
     } else {
       return "users"
     }
   }
   
   // 禁用默认表名的复数形式，如果置为 true，则 `User` 的默认表名是 `user`
   db.SingularTable(true)
   ```

3. 列名: 由字段名称进行下划线分割来生成

   ```go
   type User struct {
     ID        uint      // column name is `id`
     Name      string    // column name is `name`
     Birthday  time.Time // column name is `birthday`
     CreatedAt time.Time // column name is `created_at`
   }
   ```

​      可以使用结构体tag指定列名：(用的多)

```go
type Animal struct {
  AnimalId    int64     `gorm:"column:beast_id"`         // set column name to `beast_id`
  Birthday    time.Time `gorm:"column:day_of_the_beast"` // set column name to `day_of_the_beast`
  Age         int64     `gorm:"column:age_of_the_beast"` // set column name to `age_of_the_beast`
}
```



##### 时间戳跟踪

> ####  CreatedAt

如果模型有 `CreatedAt`字段，该字段的值将会是初次创建记录的时间。

```go
db.Create(&user) // `CreatedAt`将会是当前时间

// 可以使用`Update`方法来改变`CreateAt`的值
db.Model(&user).Update("CreatedAt", time.Now())
```

> #### UpdatedAt

如果模型有`UpdatedAt`字段，该字段的值将会是每次更新记录的时间。

```go
db.Save(&user) // `UpdatedAt`将会是当前时间

db.Model(&user).Update("name", "jinzhu") // `UpdatedAt`将会是当前时间
```

> #### DeletedAt

如果模型有`DeletedAt`字段，调用`Delete`删除该记录时，将会设置`DeletedAt`字段为当前时间，而不是直接将记录从数据库中删除。



#### 2.5 GORM CRUD

 

##### 创建

>1.定义模型

```go
type User struct {
	ID           int64
	Name         string
	Age          int64
}
```

2.使用NewRecord()判断记录是否存在，为空就用Create()创建，防止结构体过大，传入指针比较好

```go
user := User{Name: "q1mi", Age: 18}
db.NewRecord(user) // 主键为空返回`true`
db.Create(&user)   // 创建user
db.NewRecord(user) // 创建`user`后返回`false`
```

3.默认值的使用：可以通过 tag 定义字段的默认值，在创建记录时候生成的 SQL 语句会排除没有值或值为 零值 的字段，所有字段的零值, 比如`0`, `""`,`false`或者其它`零值`，都不会保存到数据库内，但会使用他们的默认值。如果你想避免这种情况，可以考虑使用指针或实现 `Scanner/Valuer`接口

```go
type User struct {
  ID   int64
  Name string `gorm:"default:'小王子'"`
  Age  int64
}
var user = User{Name: "", Age: 99}
db.Create(&user)//代码实际执行的SQL语句是INSERT INTO users("age") values('99');，排除了零值字段Name，而在数据库中这一条数据会使用设置的默认值小王子作为Name字段的值。
```

**易错点：这里注意小王子因为字符型要打单引号，否则数据库创建不了，也不会报错！！！！！！！！**

4.使用指针方式实现零值存入数据库

```go
// 使用指针
type User struct {
  ID   int64
  Name *string `gorm:"default:'小王子'"`
  Age  int64
}
user := User{Name: new(string), Age: 18))}
db.Create(&user)  // 此时数据库中该条记录name字段的值就是''
```

使用Scanner/Valuer接口方式实现零值存入数据库

```go
// 使用 Scanner/Valuer
type User struct {
	ID int64
	Name sql.NullString `gorm:"default:'小王子'"` // sql.NullString 实现了Scanner/Valuer接口
	Age  int64
}
user := User{Name: sql.NullString{"", true}, Age:18}
db.Create(&user)  // 此时数据库中该条记录name字段的值就是''
```



#####  查询

> #### 一般查询

```go
var (
		u1 NbaInfo
		u2 NbaInfo
		u3 NbaInfo
		u4 []NbaInfo
		u5 NbaInfo
	)
	db.First(&u1)       //根据主键查询第一条记录 SELECT * FROM nabinfos ORDER BY id LIMIT 1;
	db.Take(&u2)        //随机获取一条记录 SELECT * FROM nabinfos LIMIT 1;
	db.Last(&u3)        //根据主键获取最后一条记录  SELECT * FROM nabinfos ORDER BY id DESC LIMIT 1;
	db.Find(&u4)        //查询所有记录   SELECT * FROM nabinfos;
	db.First(&u5, "30") //查询指定的某条记录   SELECT * FROM nabinfos WHERE id="30";
```

> #### where条件

```go
var (
		u1 NbaInfo
		u2 []NbaInfo
		u3 []NbaInfo
		u4 []NbaInfo
		u5 []NbaInfo
		u6 []NbaInfo
		//u7 []NbaInfo
		u8    []NbaInfo
		u66   []NbaInfo
		u666  []NbaInfo
		u6666 []NbaInfo
		u9    []NbaInfo
	)
	db.Where("age = ?", 33).First(&u1)                 //查询第一条匹配到的记录  SELECT * FROM nbainfos WHERE age=33 limit 1;
	db.Where("age = ?", 33).Find(&u2)                  // 查询所有相匹配的记录 SELECT * FROM nbainfos WHERE age=33;
	db.Where("age <> ?", 33).Find(&u3)                 // <> SELECT * FROM nbainfos WHERE age <> 33;
	db.Where("age IN (?)", []int{33, 30}).Find(&u4)    // IN SELECT * FROM nbainfos WHERE age in (33,30);
	db.Where("name LIKE ?", "%ur%").Find(&u5)          // LIKE SELECT * FROM nbainfos WHERE name LIKE '%ur%'
	db.Where("age = ? AND id > ?", 33, "31").Find(&u6) // AND SELECT * FROM nbainfos WHERE age=33 AND id>31;
	// db.Where("updated_at > ?",)                         // Time SELECT * FROM nbainfos WHERE  updated_at > '2000-01-01 00:00:00';
	db.Where("id BETWEEN ? AND ?", "10", "30").Find(&u8) // BETWEEN SELECT * FROM nbainfos WHERE id BETWEEN 10 AND 30;
```

```go
//where结合struct&map查询
	db.Where(&NbaInfo{Age: 33, Id: "31"}).Find(&u66)                    // AND SELECT * FROM nbainfos WHERE age=33 AND id=31;
	db.Where(map[string]interface{}{"Age": 33, "Id": "31"}).Find(&u666) // AND SELECT * FROM nbainfos WHERE age=33 AND id=31;
	db.Where(&NbaInfo{Age: 33, Id: ""}).Find(&u6666)                    //当通过结构体查询时，gorm只通过非零值字段，如果字段为0，false,''或其他零值，将不构成查询条件,可以通过指针或实现scanner、valuer接口来避免这个问题。 SELECT *FROM nbainfos WHERE AGE = 33;
	//主键的切片放在where中
	db.Where([]string{"30", "11"}).Find(&u9)   // SELECT * FROM nbainfos WHERE id IN (30,11);
```

> #### NOT条件

```go
var (
		u1 NbaInfo
		u2 []NbaInfo
		u3 []NbaInfo
		u4 []NbaInfo
		u5 NbaInfo
	)
	db.Not("name", "curry").First(&u1)          //SELECT * FROM nbainfos WHERE NAME <>"curry" LIMIT 1;
	db.Not("age", []int{30, 33}).Find(&u2)      //SELECT * FROM nbainfos WHERE age NOT IN (30,33);
	db.Not([]string{"7", "30", "11"}).Find(&u3) // SELECT * FROM nbainfos WHERE id NOT IN ("7","30","11");
	db.Not([]string{}).Find(&u4)                //SELECT * FROM nbainfos;
	db.Not("name = ?", "curry").First(&u5)      //SELECT * FROM nbainfos WHERE NOT(name ="curry");
```

> #### OR条件

```go
	db.Where("name = ?", "curry").Or("name = ?", "klay").Find(&u1)// SELECT * FROM nbainfos WHERE name = 'curry' OR name = 'klay';
	db.Where("name = 'curry'").Or(NbaInfo{Name: "klay"}).Find(&u2)// Struct  SELECT * FROM nbainfos WHERE name = 'curry' OR name = 'klay';
	db.Where("name = 'curry'").Or(map[string]interface{}{"name": "klay"}).Find(&u3)//Map  SELECT * FROM users WHERE name = 'jinzhu' OR name = 'jinzhu 2';

```

> #### 内联条件

> #### 额外查询选项

> #### FirstOrInit

> #### FirstOrCreate

> #### 高级查询

> #### Pluck和Scan





##### 链式操作

> #### 基本示例

> #### 立即执行方法

> #### 范围

> #### 多个立即执行方法





#####  更新

> #### 更新所有字段

> #### 更新修改字段

> #### 更新选定字段

> #### 无Hooks更新

> #### 批量更新

> #### 使用sql表达式更新

> #### 修改Hooks中的值

> #### 其他更新选项



#####  删除

> #### 删除记录

> #### 批量删除

> #### 软删除

> #### 物理删除