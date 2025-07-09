# Go语言内置包之strconv

Go语言中`strconv`包实现了基本数据类型和其字符串表示的相互转换。

### strconv包 <a href="#strconv-bao" id="strconv-bao"></a>

strconv包实现了基本数据类型与其字符串表示的转换，主要有以下常用函数： `Atoi()`、`Itoa()`、parse系列、format系列、append系列。

更多函数请查看[官方文档](https://golang.org/pkg/strconv/)。

### string与int类型转换 <a href="#string-yu-int-lei-xing-zhuan-huan" id="string-yu-int-lei-xing-zhuan-huan"></a>

这一组函数是我们平时编程中用的最多的。

#### Atoi() <a href="#atoi" id="atoi"></a>

`Atoi()`函数用于将字符串类型的整数转换为int类型，函数签名如下。

```
func Atoi(s string) (i int, err error)
```

如果传入的字符串参数无法转换为int类型，就会返回错误。

```
s1 := "100"
i1, err := strconv.Atoi(s1)
if err != nil {
	fmt.Println("can't convert to int")
} else {
	fmt.Printf("type:%T value:%#v\n", i1, i1) //type:int value:100
}
```

#### Itoa() <a href="#itoa" id="itoa"></a>

`Itoa()`函数用于将int类型数据转换为对应的字符串表示，具体的函数签名如下。

示例代码如下：

```
i2 := 200
s2 := strconv.Itoa(i2)
fmt.Printf("type:%T value:%#v\n", s2, s2) //type:string value:"200"
```

#### a的典故 <a href="#a-de-dian-gu" id="a-de-dian-gu"></a>

【扩展阅读】这是C语言遗留下的典故。C语言中没有string类型而是用字符数组(array)表示字符串，所以`Itoa`对很多C系的程序员很好理解。

### Parse系列函数 <a href="#parse-xi-lie-han-shu" id="parse-xi-lie-han-shu"></a>

Parse类函数用于转换字符串为给定类型的值：ParseBool()、ParseFloat()、ParseInt()、ParseUint()。

#### ParseBool() <a href="#parsebool" id="parsebool"></a>

```
func ParseBool(str string) (value bool, err error)
```

返回字符串表示的bool值。它接受1、0、t、f、T、F、true、false、True、False、TRUE、FALSE；否则返回错误。

#### ParseInt() <a href="#parseint" id="parseint"></a>

```
func ParseInt(s string, base int, bitSize int) (i int64, err error)
```

返回字符串表示的整数值，接受正负号。

base指定进制（2到36），如果base为0，则会从字符串前置判断，“0x"是16进制，“0"是8进制，否则是10进制；

bitSize指定结果必须能无溢出赋值的整数类型，0、8、16、32、64 分别代表 int、int8、int16、int32、int64；

返回的err是\*NumErr类型的，如果语法有误，err.Error = ErrSyntax；如果结果超出类型范围err.Error = ErrRange。

#### ParseUnit() <a href="#parseunit" id="parseunit"></a>

```
func ParseUint(s string, base int, bitSize int) (n uint64, err error)
```

`ParseUint`类似`ParseInt`但不接受正负号，用于无符号整型。

#### ParseFloat() <a href="#parsefloat" id="parsefloat"></a>

```
func ParseFloat(s string, bitSize int) (f float64, err error)
```

解析一个表示浮点数的字符串并返回其值。

如果s合乎语法规则，函数会返回最为接近s表示值的一个浮点数（使用IEEE754规范舍入）。

bitSize指定了期望的接收类型，32是float32（返回值可以不改变精确值的赋值给float32），64是float64；

返回值err是\*NumErr类型的，语法有误的，err.Error=ErrSyntax；结果超出表示范围的，返回值f为±Inf，err.Error= ErrRange。

#### 代码示例 <a href="#dai-ma-shi-li" id="dai-ma-shi-li"></a>

```
b, err := strconv.ParseBool("true")
f, err := strconv.ParseFloat("3.1415", 64)
i, err := strconv.ParseInt("-2", 10, 64)
u, err := strconv.ParseUint("2", 10, 64)
```

这些函数都有两个返回值，第一个返回值是转换后的值，第二个返回值为转化失败的错误信息。

### Format系列函数 <a href="#format-xi-lie-han-shu" id="format-xi-lie-han-shu"></a>

Format系列函数实现了将给定类型数据格式化为string类型数据的功能。

#### FormatBool() <a href="#formatbool" id="formatbool"></a>

```
func FormatBool(b bool) string
```

根据b的值返回"true"或"false”。

#### FormatInt() <a href="#formatint" id="formatint"></a>

```
func FormatInt(i int64, base int) string
```

返回i的base进制的字符串表示。base 必须在2到36之间，结果中会使用小写字母’a’到’z’表示大于10的数字。

#### FormatUint() <a href="#formatuint" id="formatuint"></a>

```
func FormatUint(i uint64, base int) string
```

是FormatInt的无符号整数版本。

#### FormatFloat() <a href="#formatfloat" id="formatfloat"></a>

```
func FormatFloat(f float64, fmt byte, prec, bitSize int) string
```

函数将浮点数表示为字符串并返回。

bitSize表示f的来源类型（32：float32、64：float64），会据此进行舍入。

fmt表示格式：‘f’（-ddd.dddd）、‘b’（-ddddp±ddd，指数为二进制）、’e’（-d.dddde±dd，十进制指数）、‘E’（-d.ddddE±dd，十进制指数）、‘g’（指数很大时用’e’格式，否则’f’格式）、‘G’（指数很大时用’E’格式，否则’f’格式）。

prec控制精度（排除指数部分）：对’f’、’e’、‘E’，它表示小数点后的数字个数；对’g’、‘G’，它控制总的数字个数。如果prec 为-1，则代表使用最少数量的、但又必需的数字来表示f。

#### 代码示例 <a href="#dai-ma-shi-li-1" id="dai-ma-shi-li-1"></a>

```
s1 := strconv.FormatBool(true)
s2 := strconv.FormatFloat(3.1415, 'E', -1, 64)
s3 := strconv.FormatInt(-2, 16)
s4 := strconv.FormatUint(2, 16)
```

### 其他 <a href="#qi-ta" id="qi-ta"></a>

#### isPrint() <a href="#isprint" id="isprint"></a>

```
func IsPrint(r rune) bool
```

返回一个字符是否是可打印的，和`unicode.IsPrint`一样，r必须是：字母（广义）、数字、标点、符号、ASCII空格。

#### CanBackquote() <a href="#canbackquote" id="canbackquote"></a>

```
func CanBackquote(s string) bool
```

返回字符串s是否可以不被修改的表示为一个单行的、没有空格和tab之外控制字符的反引号字符串。

#### 其他 <a href="#qi-ta-1" id="qi-ta-1"></a>

除上文列出的函数外，`strconv`包中还有Append系列、Quote系列等函数。具体用法可查看[官方文档](https://golang.org/pkg/strconv/)。

***
