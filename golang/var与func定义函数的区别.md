# var 与 func 定义函数的区别

看 `rclone` 源代码时，发现一段比较奇怪的代码：

```go
// fs/config/ui.go

// ReadLine reads some input
var ReadLine = func() string {
	buf := bufio.NewReader(os.Stdin)
	line, err := buf.ReadString('\n')
	if err != nil {
		log.Fatalf("Failed to read line: %v", err)
	}
	return strings.TrimSpace(line)
}
```

我很疑惑，为什么这里不使用 `func ReadLine` 定义函数呢？再继续看代码：

```go
// fs/config/ui.go

// makeReadLine makes a simple readLine which returns a fixed list of
// strings
func makeReadLine(answers []string) func() string {
	i := 0
	return func() string {
		i = i + 1
		return answers[i-1]
	}
}

func TestCRUD(t *testing.T) {
	defer testConfigFile(t, "crud.conf")()
	ctx := context.Background()

	// script for creating remote
	config.ReadLine = makeReadLine([]string{
		"config_test_remote", // type
		"true",               // bool value
		"y",                  // type my own password
		"secret",             // password
		"secret",             // repeat
		"y",                  // looks good, save
	})
    // ...
}
```

当单元测试时，我们需要直接传入值，而不是从客户端获取输入。这时就不能使用`func`在测试文件重复定义函数，重复定义是会报错的。但可以使用`var`定义的匿名函数。



##### 参考：

> 这两种形式的主要区别在于，它们不仅在语法上不同，而且在*语义上也不同：*
>
> - 使用`func name (...)`表单的“普通”表单创建了一个常规命名函数。
> - 您所谓的“使用创建函数`var`”实际上是创建一个所谓的匿名函数并将其值分配给一个变量。
>
> 不同之处在于，在 Go 中，匿名函数表现为闭包，而常规函数则不然。
>
> 闭包是一个函数，它从函数体中使用的外部词法范围“捕获”任何变量（同时不会被那里的局部变量和函数参数遮蔽）。
>
> 当在包的顶层使用任何一种形式时，区别可能并不明显 - 也就是说，在任何函数的主体之外，因为在这种情况下，外部作用域是包的全局变量，但在其他函数内部，区别是明显的：“正常”形式根本无法使用。
>
> 另一个区别是您可以替换包含函数值的变量中的值，而不能对普通函数执行相同操作。
>
> 尽管如此，如果我们谈论顶级代码，Flimzy 提供的建议仍然成立：在生产 Go 代码中，包含函数值的全局变量是代码味道，*直到*它被证明不是这样。

ps: 以上参考是`google`英转中的，所以文字读起来比较蹩脚，理解就行。
