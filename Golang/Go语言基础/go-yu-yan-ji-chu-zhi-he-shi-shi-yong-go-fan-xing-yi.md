# Go语言基础之何时使用Go泛型【译】

Go 1.18版本增加了一个主要的新语言特性: 对泛型的支持。在本文中，我不会描述泛型是什么，也不会描述如何使用它们。本文讨论在 Go 代码中何时使用泛型，以及何时不使用它们。

事先声明，本文提供的是一般指导建议，而不是硬性规定。是否使用泛型需要你自己的判断。但如果你不确定，那么建议使用这里讨论的指南。

### 编写代码 <a href="#bian-xie-dai-ma" id="bian-xie-dai-ma"></a>

让我们从编写 Go 程序的一般准则开始：通过编写代码而不是定义类型来编写 Go 程序。在泛型方面，如果通过定义类型参数约束开始编写程序，那么你就错了。应该从编写函数开始。如果明确知道类型参数有用的情况下，很容易在以后添加类型参数。

### 应该使用类型参数 <a href="#ying-gai-shi-yong-lei-xing-can-shu" id="ying-gai-shi-yong-lei-xing-can-shu"></a>

让我们看一下类型参数对哪些情况有用。

#### 使用语言定义的容器类型时 <a href="#shi-yong-yu-yan-ding-yi-de-rong-qi-lei-xing-shi" id="shi-yong-yu-yan-ding-yi-de-rong-qi-lei-xing-shi"></a>

当我们编写的是操作 Go 语言定义的特殊容器类型（slice、map和chennel）的函数。如果函数具有包含这些类型的参数，并且函数的代码并不关心元素的类型，那么使用类型参数可能是有用的。

例如，这里有一个函数，它的功能是返回任何类型map中所有的key：

```
// MapKeys 返回m中所有key组成的切片
func MapKeys[Key comparable, Val any](m map[Key]Val) []Key {
    s := make([]Key, 0, len(m))
    for k := range m {
        s = append(s, k)
    }
    return s
}
```

这段代码并不关注 map 中键的类型，也根本没有使用 map 值类型。它适用于任何map类型。这是使用类型参数的一个很好示例。

在引入类型参数之前，想要实现类似功能通常是使用反射，但是使用反射实现通常是复杂的，并且在编译期间不会进行静态类型检查，在运行时通常速度也更慢。

#### 通用数据结构 <a href="#tong-yong-shu-ju-jie-gou" id="tong-yong-shu-ju-jie-gou"></a>

类型参数另一个适用场景就是用于通用数据结构。通用数据结构类似于slice或map，但不是内置在语言中的，例如链表或二叉树。

之前需要这种数据结构的程序通常采用下面两种方法中的一个：使用特定的元素类型编写数据结构，或者使用接口类型。用类型参数替换特定的元素类型可以生成更通用的数据结构，该数据结构可以在程序的其他部分或其他程序中使用。用类型参数替换接口类型可以更有效地存储数据，节省内存资源；它还允许代码避免类型断言，并在构建时进行完全的类型检查。

例如，下面是使用类型参数的二叉树数据结构的一部分：

```
// Tree is a binary tree.
type Tree[T any] struct {
    cmp  func(T, T) int
    root *node[T]
}

// A node in a Tree.
type node[T any] struct {
    left, right  *node[T]
    val          T
}

// find returns a pointer to the node containing val,
// or, if val is not present, a pointer to where it
// would be placed if added.
func (bt *Tree[T]) find(val T) **node[T] {
    pl := &bt.root
    for *pl != nil {
        switch cmp := bt.cmp(val, (*pl).val); {
        case cmp < 0:
            pl = &(*pl).left
        case cmp > 0:
            pl = &(*pl).right
        default:
            return pl
        }
    }
    return pl
}

// Insert inserts val into bt if not already there,
// and reports whether it was inserted.
func (bt *Tree[T]) Insert(val T) bool {
    pl := bt.find(val)
    if *pl != nil {
        return false
    }
    *pl = &node[T]{val: val}
    return true
}
```

树中的每个节点都包含类型参数 T 的值。使用特定类型参数实例化树时，该类型的值将直接存储在节点中。它们不会被存储为接口类型。

这是对类型参数的合理使用，因为 Tree 数据结构(包括方法中的代码)在很大程度上与元素类型 T 无关。

Tree数据结构需要知道如何比较元素类型T的值；它为此使用传入的比较函数。你可以在find方法的第四行，即对bt.cmp的调用中看到这一点。除此之外，type参数根本不重要。

#### 对于类型参数，优先选择函数而不是方法 <a href="#dui-yu-lei-xing-can-shu-you-xian-xuan-ze-han-shu-er-bu-shi-fang-fa" id="dui-yu-lei-xing-can-shu-you-xian-xuan-ze-han-shu-er-bu-shi-fang-fa"></a>

Tree 示例说明了另一个一般原则：当你需要比较函数之类的东西时，更喜欢使用函数而不是方法。

我们可以定义 Tree 类型，这样元素类型就必须有一个 Compare 或 Less 方法。这可以通过编写一个需要该方法的约束来实现，这意味着用于实例化 Tree 类型的任何类型参数都需要该方法。

这样做的结果是，任何想使用像 int 这样的简单数据类型的 Tree 的人都必须定义自己的整数类型，并编写自己的比较方法。如果我们像上面所示的代码那样定义 Tree 以获取比较函数，那么很容易传递所需的函数。编写比较函数和编写方法一样容易。

如果 Tree 元素类型碰巧已经有一个 Compare 方法，那么我们可以简单地使用一个类似 ElementType 的方法表达式。作为比较函数进行比较。

换句话说，将方法转换为函数要比将方法添加到类型中简单得多。因此，对于通用数据类型，更喜欢使用函数而不是编写需要方法的约束。

#### 实现通用方法 <a href="#shi-xian-tong-yong-fang-fa" id="shi-xian-tong-yong-fang-fa"></a>

类型参数可能有用的另一种情况是，不同类型需要实现某些公共方法，而不同类型的实现看起来都是相同的。

例如，考虑标准库的 `sort.Interface`，它要求类型实现三个方法: `Len`、 `Swap` 和 `Less`。

下面是一个泛型类型 `SliceFn` 的示例，它为切片类型实现 `sort.Interface`：

```
// SliceFn 为T类型切片实现 sort.Interface 
type SliceFn[T any] struct {
    s    []T
    less func(T, T) bool
}

func (s SliceFn[T]) Len() int {
    return len(s.s)
}
func (s SliceFn[T]) Swap(i, j int) {
    s.s[i], s.s[j] = s.s[j], s.s[i]
}
func (s SliceFn[T]) Less(i, j int) bool {
    return s.less(s.s[i], s.s[j])
}
```

对于任意切片类型，`Len` 和 `Swap` 方法完全相同。`Less` 方法需要进行比较，这是名称 `SliceFn` 中的 `Fn` 的部分。与前面的 `Tree` 示例一样，我们将在创建 `SliceFn` 时传入一个函数。

下面是如何通过 `SliceFn`函数使用比较函数对任意切片进行排序：

```
// SortFn 使用比较函数进行排序。
func SortFn[T any](s []T, less func(T, T) bool) {
    sort.Sort(SliceFn[T]{s, less})
}
```

这类似于标准库函数`sort.Slice`，但比较函数是使用值而不是切片索引编写的。

对这类代码使用类型参数是合适的，因为所有切片类型的方法看起来完全相同。

### 不应该使用类型参数 <a href="#bu-ying-gai-shi-yong-lei-xing-can-shu" id="bu-ying-gai-shi-yong-lei-xing-can-shu"></a>

接下来谈谈何时不使用类型参数。

#### 不要用类型参数替换接口类型 <a href="#bu-yao-yong-lei-xing-can-shu-ti-huan-jie-kou-lei-xing" id="bu-yao-yong-lei-xing-can-shu-ti-huan-jie-kou-lei-xing"></a>

众所周知，Go有接口类型。接口类型允许一种通用编程。

例如，广泛使用的`io.Reader`接口提供了一种通用机制，用于从包含信息（例如文件）或产生信息（例如随机数生成器）的任何值读取数据。如果对某个类型的值只需要调用该值的方法，则使用接口类型，而不是类型参数。`io.Reader`易于阅读、高效且有效。不需要使用类型参数，通过调用read方法从值中读取数据。

例如，你可能会尝试将这里的第一个函数签名（仅使用接口类型）更改为第二个版本（使用类型参数）。

```
func ReadSome(r io.Reader) ([]byte, error)

func ReadSome[T io.Reader](r T) ([]byte, error)
```

不要做出那种改变。省略type参数使函数更容易编写，更容易读取，并且执行时间可能相同。

最后一点值得强调。虽然可以用几种不同的方式实现泛型，而且随着时间的推移，实现也会发生变化和改进，但在许多情况下，Go 1.18中使用的实现将处理类型为类型参数的值，就像处理类型为接口类型的值一样。这意味着使用类型参数通常不会比使用接口类型快。因此，不要为了速度而从接口类型更改为类型参数，因为它可能不会运行得更快。

#### 如果方法实现不同，不要使用类型参数 <a href="#ru-guo-fang-fa-shi-xian-bu-tong-bu-yao-shi-yong-lei-xing-can-shu" id="ru-guo-fang-fa-shi-xian-bu-tong-bu-yao-shi-yong-lei-xing-can-shu"></a>

在决定是否使用类型参数或接口类型时，请考虑方法的实现。前面我们说过，如果一个方法的实现对于所有类型都是相同的，那么就使用一个类型参数。相反，如果每种类型的实现都不同，则使用接口类型并编写不同的方法实现，不要使用类型参数。

例如，从文件读取的实现与从随机数生成器读取的实现完全不同。这意味着我们应该编写两个不同的Read方法，并使用像io.Reader这样的接口类型。

#### 在适当的地方使用反射 <a href="#zai-shi-dang-de-di-fang-shi-yong-fan-she" id="zai-shi-dang-de-di-fang-shi-yong-fan-she"></a>

Go具有运行时反射。反射允许一种泛型编程，因为它允许你编写适用于任何类型的代码。

如果某些操作甚至必须支持没有方法的类型（不能使用接口类型），并且每个类型的操作都不同（不能使用类型参数），请使用反射。

encoding/json包就是一个例子。我们不想要求我们编码的每个类型都有MarshalJSON方法，所以我们不能使用接口类型。但对接口类型的编码与对结构类型的编码不同，因此我们不应该使用类型参数。相反，该包使用反射。代码不简单，但它有效。有关详细信息，请参阅[源代码](https://go.dev/src/encoding/json/encode.go)。

### 一个简单的准则 <a href="#yi-ge-jian-dan-de-zhun-ze" id="yi-ge-jian-dan-de-zhun-ze"></a>

最后，关于何时使用泛型的讨论可以简化为一个简单的指导原则。

如果您发现自己多次编写完全相同的代码，其中副本之间的唯一区别是代码使用不同的类型，请考虑是否可以使用类型参数。

另一种说法是，在注意到要多次编写完全相同的代码之前，应该避免使用类型参数。

原文链接：[https://go.dev/blog/when-generics](https://go.dev/blog/when-generics)

***
