## Golang defer 这该死的温柔
Defer是Go语言提供的一种用于注册延迟调用的机制: 让函数或语句可以在当前函数执行完毕后(包括通过return正常结束或者panic导致异常结束)执行。深受Go开发者的欢迎。<br>

#### 什么是defer？
defer语句通常用于一些成对操作的场景:打开链接/关闭链接；打开文件/关闭文件等<br>

defer在一些需要回收资源的场景非常有用，可以很方便地在函数结束前做一些清理操作。在打开资源语句的下一行，可以直接一句defer就可以在函数返回钱关闭资源，可谓相当优雅。<br>
```go
	f,err := os.Open("hello.txt")
	check(err)
	b1 := make([]byte,10)
	n1,err := f.Read(b1)
	check(err)
	fmt.Printf("%d bytes: %s\n", n1, string(b1[:n1]))
	defer func(){
		f.Close()
	}()
```
先判断err，需要判断 所以写了一个check，错了就直接panic
#### 为什么需要defer？ 
资源需要释放，不然就内存泄漏
#### 合理使用defer
如上代码所示，要判断是否error
### 又到进阶部分
定义：每次defer语句执行的时候，会把函数压栈，函数参数会被拷贝下来；当外层函数(非代码块，如一个for循环)退出时，defer函数按照定义的逆序执行；如果defer执行的函数为nil，那么会在最终调用函数的产生panic。<br>
什么意思？defer语句不会马上执行，而是会进入一个栈，函数return之前，会按照先进后出的顺序执行。最先被定义的defer语句最后执行。先进后出原因就是后面定义的函数可能会依赖前面的资源，自然要先执行；否则如果前面先执行了，那后面函数的依赖就没有了<br>
```go
	var whatever [3]struct{}
	fmt.Println(whatever[2])
	for index,val:=range whatever{
		defer func(){
			fmt.Println(index)
			fmt.Println(val)
		}()
	}
```
defer后面会跟一个closure，i是引用类型的变量，最后的i的值是2。<br>
#### 利用defer原理
故意用到defer先求值，再延迟调用的性质。在一个函数里需要打开两个文件进行合并操作，合并完后，在函数执行完后关闭打开的文件句柄

#### defer命令拆解
return xxx=> 1.返回值 = xxx 2. 调用defer函数 3. 返回空return。<br>
第二步是defer定义的语句，这里可能会操作返回值<br>
举个🌰
```go
func f()(r int){
    t := 5
    defer func(){
        t=t+5
    }()
    return t
}
=>拆解后
func f()(r int){
    t := 5
    r = t
    func(){
        t = t+5
    }
    return
}
```
所以这里的r并没有变成10

#### defer语句的参数
defer语句表达式的值在定义时就已经确定了。<br>
```go
defer fmt.Println("defer main") // 入栈
	var user = os.Getenv("User_")
	 go func(){
		 defer func(){
			 fmt.Println("defer caller")// 入栈
			 if err := recover();err !=nil{
				 fmt.Println("recover success.err:",err)
			 }
		 }()
		 func(){
			 defer func(){
				 fmt.Println("defer here")// 入栈
			 }()
			 if user == ""{
				 panic("should set user env")
			 }
			 // 此处不执行
			 fmt.Println("after panic")
		 }()
	 }()
	 time.Sleep(100)
	 fmt.Println("end of main function") // 主协程执行
```