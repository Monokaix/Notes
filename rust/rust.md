# CTFE（Compile-Time Function Excute）

编译时函数执行，Rust会在编译的时候执行某些函数，以得到确定值（C++和D语言也有）
eg：Rust中的固定长度数组在编译时需要确定数组大小，因此在编译期间通过执行返回数组
大小的函数来获取数组的真实长度。

# match 关键字

类似于switch case
if let、while let在某些情况下会替代match，使代码变的更简洁
if let左边为模式，右边为要匹配的值
loop和while true
while true在编译期间编译器并不知道while后面的true，在while true之外
还会有返回值
而loop则是确定的死循环，loop代码体后面不再需要返回值。
eg：

```rust
fn while_true(x: i32) -> i32 {
    loop {
        return x + 1;
    }
    return x;
}
```

return x不会被执行到，编译时会告警

# range

(1 .. 5 ）表示左闭右开区间，（ l .. =5）则表示全闭区间。
range有个.sum()方法，返回范围中的元素的和

(1..5）.sum()

# 基本数据类型

- 整数：分三种类型
  1）固定大小类型：
  无符号：u8 1字节
  有符号：i32 4字节
  2）动态大小类型
  usize 4或8字节，取决于机器字长
  3）浮点数

- 字符： 占4字节

- 字符串：有两种类型
  1）不可变类型：也叫字符串切片，以不可变借用形式存在。&str
  2）可变类型：类型String
  Rust 中的字符串本质上是一段有效的UTF8字节序列

- 数组：
  声明：let arr: ［i32 ; 3] = [ 1,2,3] ;
  对于原始固定长度数组，只有实现Copy trait 的类型才能作为其元素，也就是说，只有
  可以在栈上存放的元素才可以存放在该类型的数组中。

- 切片：
  在底层，切片代表一个指向数组起始位置的指针和数组长度。用［T］类型表示连续序列，
  那么切片类型就是＆［T］和＆mut [T］ 。
  ＆对数组进行引用，就产生了一个切片＆arr

- 指针：
  1）包括引用( Reference ）、原生指针（ Raw Po int er ）、函数指针（ fn Pointer ）和智能指针（ Sm a rt Pointer ）。
  2）使用原生指针要使用unsafe块
  3）引用的本质上是一种非空指针
  4）Rust支持两种原生指针： 不可变原生指针＊ const T和可变原生指针＊ mut T

# 复合数据类型

- 元组（ Tuple)

    - 用（e1,e2）表示，当只有一份元素时，要加逗号，（1，），为了区分其他值（1）

    - 因为let 支持模式匹配，所以可以用来解构元组

    - let x,y = (1,2)

    - 元素类型可以不同，但长度固定

- 结构体（ Struct)

    - 具名结构体（ Named-Field Struct)

    - 元组结构体（ Tuple-Like Struct )（匿名结构体）
      字段只有类型，没有名称，当一个元组结构体只有一个字段的时候，我们称之为New Type 模式

    - 单元结构体（ Unit-Like Struct)
      Rust 中可以定义一个没有任何字段的结构体，即单元结构体
      eg：

      ```rust
      struct Empty;
      ```

- 枚举体（ Enum )

- 联合体（ Union )