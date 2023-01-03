# for range 原理

```go
var waitGroup sync.WaitGroup

func main() {
	//...
    waitGroup.Add(len(rules))
	for _, rule := range rules {
		fmt.Println(rule)
		go backupsSingleCompute(&rule)
	}
	waitGroup.Wait()
	//...
}

// 备份单个节点
func backupsSingleCompute(rule *Rule) {
    waitGroup.Done()
    //...
}
```

我上面这段代码目的是使用多线程备份多个节点，但发现`backupsSingleCompute`函数获得的传参都是最后一个切片元素。

为了了解清楚内存地址发生的变化，我将上面的代码，简单修改一下进行测试：

```go
func main() {
	rules := [3]string{"1", "2", "3"}
	fmt.Println(&rules[0], &rules[1], &rules[2])
	for _, rule := range rules {
		fmt.Println("main", rule, &rule)
		go backupsSingleCompute(&rule)
	}
	time.Sleep(1 * time.Second)
}

func backupsSingleCompute(rule *string) {
	fmt.Println("backupsSingleCompute:", *rule, rule)
}
// 输出：
// 0xc0000121b0 0xc0000121c0 0xc0000121d0
// main 1 0xc000010250
// main 2 0xc000010250
// main 3 0xc000010250
// backupsSingleCompute: 3 0xc000010250
// backupsSingleCompute: 3 0xc000010250
// backupsSingleCompute: 3 0xc000010250
// https://go.dev/play/p/Y5alJop7MFT
```

从上面的输出我们可以看到，`for range` 结构块内切片地址都变成同一个（`0xc000010250`）了，而且跟之前的地址（`0xc0000121b0 0xc0000121c0 0xc0000121d0`）不同。所以`backupsSingleCompute`函数获得的传参都是最后一个切片元素：3。

为什么`for range` 结构块内切片地址都变成同一个（`0xc000010250`）了呢？这就涉及到`for range`的内部原理了。关于这个原理，可详细看《Go 语言设计与实现》一书的《[5.1 for 和 range](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-for-range/)》。书中说到`for range`遍历索引和元素的情况，编译器最终会转换成如下形式：

```go
ha := a
hv1 := 0
hn := len(ha)
v1 := hv1
v2 := nil
for ; hv1 < hn; hv1++ {
    tmp := ha[hv1]
    v1, v2 = hv1, tmp
    ...
}
```

上面`v1`是索引，`v2`是元素，可以看到`v2`在`for`循环外面就声明定义好了，所以`for`循环内，怎样赋值它的内存地址都是不会变的。这就解析了，为什么`for range` 结构块内切片地址都变成同一个（`0xc000010250`）了。

如果我们想达到预期正确的输出，可以不使用指针参数，改为下面这种方式：

```go
func main() {
	rules := [3]string{"1", "2", "3"}
	fmt.Println(&rules[0], &rules[1], &rules[2])
	for _, rule := range rules {
		fmt.Println("main", rule, &rule)
		go backupsSingleCompute(rule)
	}
	time.Sleep(1 * time.Second)
}

func backupsSingleCompute(rule string) {
	fmt.Println("backupsSingleCompute:", rule, &rule)
}
```
