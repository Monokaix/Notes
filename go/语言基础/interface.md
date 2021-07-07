# interface

- interface有两种类型，一种是iface，这种interface具有方法，另一种是eface，接口内没有方法

- 当是使用结构体指针实现接口时，如果使用结构体变量声明，则会报结构体没有实现接口的错误，因为使用结构体指针声明时，方法会返回一个指针需要接收

```go
package main

type Cat struct{}

type Duck interface {
   Quack()
}

func (c *Cat) Quack() {} // 使用结构体指针实现接口

var d Duck = Cat{} // 使用结构体指针初始化变量 编译不通过
```

以上这段代码最后一行编译不通过，会报一下错误：

Cannot use 'Cat{}' (type Cat) as the type Duck Type does not implement 'Duck' as the 'Quack' method has a pointer receiver

因为声明的方法是指针方法，该方法会有一个指针参数，而如果用结构体声明进行方法调用，则调用后的修改不会应用到源结构体，会发生一些不易发现的问题，go为了避免出现这种情况放弃了这种调用。

- eface 空结构体

```go
type eface struct { // 16 字节
	_type *_type
	data  unsafe.Pointer
}
```

```go
type _type struct {
	size       uintptr
	ptrdata    uintptr
	hash       uint32
	tflag      tflag
	align      uint8
	fieldAlign uint8
	kind       uint8
	equal      func(unsafe.Pointer, unsafe.Pointer) bool
	gcdata     *byte
	str        nameOff
	ptrToThis  typeOff
}
```

`size` 字段存储了类型占用的内存空间，为内存空间的分配提供信息；

`hash` 字段能够帮助我们快速确定类型是否相等；

`equal` 字段用于判断当前类型的多个对象是否相等，该字段是为了减少 Go 语言二进制包大小从 `typeAlg` 结构体中迁移过来的[4](https://draveness.me/golang/docs/part2-foundation/ch04-basic/golang-interface/#fn:4)；

- iface 非空结构体

```go
type iface struct { // 16 字节
	tab  *itab
	data unsafe.Pointer
}
```

```go
type itab struct { // 32 字节
	inter *interfacetype
	_type *_type
	hash  uint32
	_     [4]byte
	fun   [1]uintptr
}
```

inter：interface自己的静态类型

_type：具体类型

hash： `_type.hash` 的拷贝，当我们想将 `interface` 类型转换成具体类型时，可以使用该字段快速判断目标类型和具体类型 [`runtime._type`](https://draveness.me/golang/tree/runtime._type) 是否一致

fun：一个动态大小的数组，它是一个用于动态派发的虚函数表，存储了一组函数指针。虽然该变量被声明成大小固定的数组，但是在使用时会通过原始指针获取其中的数据，所以 `fun` 数组中保存的元素数量是不确定的

- 类型转换与断言

    - 类型转换：即将具体的实现转换为interface，然后用interface调用实现的方法

  ```
  package main
      
  type Duck interface {
  	Quack()
  }
  
  type Cat struct {
  	Name string
  }
  
  //go:noinline
  func (c *Cat) Quack() {
  	println(c.Name + " meow"){
  }
  
  func main() {
  	var c Duck = &Cat{Name: "draven"}
  	c.Quack()
  }
  ```

  通过汇编代码分析，这部分代码共包含三个主要步骤：

  1）结构体Cat的初始化

  2）赋值触发的类型转换过程

  3）调用接口的方法Quack()

  主要是第二步，Duck这个`runtime.iface`的tab字段会填充Cat和Duck类型（即inter和_type字段），然后复制到栈指针SP，最后调用Quack方法完成函数执行

    - 断言

      断言时会通过hash字段比较接口类型和目标类型是否相等，然后执行不同的操作

- 动态派发

  在进行类型转换时，会用到动态派发

  ```go
  func main() {
  	var c Duck = &Cat{Name: "draven"}
  	c.Quack()  // 1
  	c.(*Cat).Quack() // 2
  }
  ```



在1处，会在运行时进行动态派发，会生成较多的汇编指令，效率较低

c.Quack()方法展开进行三个步骤：

1. 从接口变量中获取保存 `Cat.Quack` 方法指针的 `tab.func[0]`；
2. 接口变量在 [`runtime.iface`](https://draveness.me/golang/tree/runtime.iface) 中的数据会被拷贝到栈顶；
3. 方法指针会被拷贝到寄存器中并通过汇编指令 `CALL` 触发：

在2处，**调用的函数指针在编译时就确定了**，执行效率较高

