# Go语言文件操作

本文主要介绍了Go语言中文件读写的相关操作。

文件是什么？

计算机中的文件是存储在外部介质（通常是磁盘）上的数据集合，文件分为文本文件和二进制文件。

### 打开和关闭文件 <a href="#da-kai-he-guan-bi-wen-jian" id="da-kai-he-guan-bi-wen-jian"></a>

`os.Open()`函数能够打开一个文件，返回一个`*File`和一个`err`。对得到的文件实例调用`close()`方法能够关闭文件。

```
package main

import (
	"fmt"
	"os"
)

func main() {
	// 只读方式打开当前目录下的main.go文件
	file, err := os.Open("./main.go")
	if err != nil {
		fmt.Println("open file failed!, err:", err)
		return
	}
	// 关闭文件
	file.Close()
}
```

为了防止文件忘记关闭，我们通常使用defer注册文件关闭语句。

### 读取文件 <a href="#du-qu-wen-jian" id="du-qu-wen-jian"></a>

### file.Read() <a href="#fileread" id="fileread"></a>

#### 基本使用 <a href="#ji-ben-shi-yong" id="ji-ben-shi-yong"></a>

Read方法定义如下：

```
func (f *File) Read(b []byte) (n int, err error)
```

它接收一个字节切片，返回读取的字节数和可能的具体错误，读到文件末尾时会返回`0`和`io.EOF`。 举个例子：

```
func main() {
	// 只读方式打开当前目录下的main.go文件
	file, err := os.Open("./main.go")
	if err != nil {
		fmt.Println("open file failed!, err:", err)
		return
	}
	defer file.Close()
	// 使用Read方法读取数据
	var tmp = make([]byte, 128)
	n, err := file.Read(tmp)
	if err == io.EOF {
		fmt.Println("文件读完了")
		return
	}
	if err != nil {
		fmt.Println("read file failed, err:", err)
		return
	}
	fmt.Printf("读取了%d字节数据\n", n)
	fmt.Println(string(tmp[:n]))
}
```

#### 循环读取 <a href="#xun-huan-du-qu" id="xun-huan-du-qu"></a>

使用for循环读取文件中的所有数据。

```
func main() {
	// 只读方式打开当前目录下的main.go文件
	file, err := os.Open("./main.go")
	if err != nil {
		fmt.Println("open file failed!, err:", err)
		return
	}
	defer file.Close()
	// 循环读取文件
	var content []byte
	var tmp = make([]byte, 128)
	for {
		n, err := file.Read(tmp)
		if err == io.EOF {
			fmt.Println("文件读完了")
			break
		}
		if err != nil {
			fmt.Println("read file failed, err:", err)
			return
		}
		content = append(content, tmp[:n]...)
	}
	fmt.Println(string(content))
}
```

### bufio读取文件 <a href="#bufio-du-qu-wen-jian" id="bufio-du-qu-wen-jian"></a>

bufio 是在 file 的基础上封装了一层API，支持更多的功能。

```
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	file, err := os.Open("./xx.txt")
	if err != nil {
		fmt.Println("open file failed, err:", err)
		return
	}
	defer file.Close()
	scanner := bufio.NewScanner(file) // MaxScanTokenSize = 64 * 1024
	for scanner.Scan() {
		fmt.Println(scanner.Text())
	}
	if err := scanner.Err(); err != nil {
		fmt.Println("read file failed, err:", err)
	}
}
```

### 读取整个文件 <a href="#du-qu-zheng-ge-wen-jian" id="du-qu-zheng-ge-wen-jian"></a>

`io/ioutil`包的`ReadFile`方法能够读取完整的文件，只需要将文件名作为参数传入。

```
package main

import (
	"fmt"
	"io/ioutil"
)

// ioutil.ReadFile读取整个文件
func main() {
	content, err := ioutil.ReadFile("./main.go")
	if err != nil {
		fmt.Println("read file failed, err:", err)
		return
	}
	fmt.Println(string(content))
}
```

### 文件写入操作 <a href="#wen-jian-xie-ru-cao-zuo" id="wen-jian-xie-ru-cao-zuo"></a>

`os.OpenFile()`函数能够以指定模式打开文件，从而实现文件写入相关功能。

```
func OpenFile(name string, flag int, perm FileMode) (*File, error) {
	...
}
```

其中：

`name`：要打开的文件名 `flag`：打开文件的模式。 模式有以下几种：

| 模式            | 含义   |
| ------------- | ---- |
| `os.O_WRONLY` | 只写   |
| `os.O_CREATE` | 创建文件 |
| `os.O_RDONLY` | 只读   |
| `os.O_RDWR`   | 读写   |
| `os.O_TRUNC`  | 清空   |
| `os.O_APPEND` | 追加   |

`perm`：文件权限，一个八进制数。r（读）04，w（写）02，x（执行）01。

### Write和WriteString <a href="#write-he-writestring" id="write-he-writestring"></a>

```
func main() {
	file, err := os.OpenFile("xx.txt", os.O_CREATE|os.O_TRUNC|os.O_WRONLY, 0666)
	if err != nil {
		fmt.Println("open file failed, err:", err)
		return
	}
	defer file.Close()
	str := "hello 沙河"
	file.Write([]byte(str))       //写入字节切片数据
	file.WriteString("hello 小王子") //直接写入字符串数据
}
```

### bufio.NewWriter <a href="#bufionewwriter" id="bufionewwriter"></a>

```
func main() {
	file, err := os.OpenFile("xx.txt", os.O_CREATE|os.O_TRUNC|os.O_WRONLY, 0666)
	if err != nil {
		fmt.Println("open file failed, err:", err)
		return
	}
	defer file.Close()
	writer := bufio.NewWriter(file)
	for i := 0; i < 10; i++ {
		writer.WriteString("hello沙河\n") //将数据先写入缓存
	}
	writer.Flush() //将缓存中的内容写入文件
}
```

### ioutil.WriteFile <a href="#ioutilwritefile" id="ioutilwritefile"></a>

```
func main() {
	str := "hello 沙河"
	err := ioutil.WriteFile("./xx.txt", []byte(str), 0666)
	if err != nil {
		fmt.Println("write file failed, err:", err)
		return
	}
}
```

### 练习 <a href="#lian-xi" id="lian-xi"></a>

### copyFile <a href="#copyfile" id="copyfile"></a>

借助`io.Copy()`实现一个拷贝文件函数。

```
// CopyFile 拷贝文件函数
func CopyFile(dstName, srcName string) (written int64, err error) {
	// 以读方式打开源文件
	src, err := os.Open(srcName)
	if err != nil {
		fmt.Printf("open %s failed, err:%v.\n", srcName, err)
		return
	}
	defer src.Close()
	// 以写|创建的方式打开目标文件
	dst, err := os.OpenFile(dstName, os.O_WRONLY|os.O_CREATE, 0644)
	if err != nil {
		fmt.Printf("open %s failed, err:%v.\n", dstName, err)
		return
	}
	defer dst.Close()
	return io.Copy(dst, src) //调用io.Copy()拷贝内容
}
func main() {
	_, err := CopyFile("dst.txt", "src.txt")
	if err != nil {
		fmt.Println("copy file failed, err:", err)
		return
	}
	fmt.Println("copy done!")
}
```

### 实现一个cat命令 <a href="#shi-xian-yi-ge-cat-ming-ling" id="shi-xian-yi-ge-cat-ming-ling"></a>

使用文件操作相关知识，模拟实现linux平台`cat`命令的功能。

```
package main

import (
	"bufio"
	"flag"
	"fmt"
	"io"
	"os"
)

// cat命令实现
func cat(r *bufio.Reader) {
	for {
		buf, err := r.ReadBytes('\n') //注意是字符
		if err == io.EOF {
			// 退出之前将已读到的内容输出
			fmt.Fprintf(os.Stdout, "%s", buf)
			break
		}
		fmt.Fprintf(os.Stdout, "%s", buf)
	}
}

func main() {
	flag.Parse() // 解析命令行参数
	if flag.NArg() == 0 {
		// 如果没有参数默认从标准输入读取内容
		cat(bufio.NewReader(os.Stdin))
	}
	// 依次读取每个指定文件的内容并打印到终端
	for i := 0; i < flag.NArg(); i++ {
		f, err := os.Open(flag.Arg(i))
		if err != nil {
			fmt.Fprintf(os.Stdout, "reading from %s failed, err:%v\n", flag.Arg(i), err)
			continue
		}
		cat(bufio.NewReader(f))
	}
}
```

***
