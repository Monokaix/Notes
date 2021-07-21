# new&make

1. make用来初始化内置数据结构，也就是slice、map、channel
2. new根据传入的类型初始化内存，并返回指向内存的指针
3. 在编译期间，使用`new`和`var`声明变量，在编译器看来都是 `ONEW` 和 `ODCL` 节点。如果变量会逃逸到堆上，这些节点在这一阶段都会被 [`cmd/compile/internal/gc.walkstmt`](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkstmt) 转换成 [`runtime.newobject`](https://draveness.me/golang/tree/runtime.newobject) 函数并在堆上申请内存：
4. new最终会被转化为`ONEWOBJ`(在函数`(s *state) expr`中)的操作，然后调用`newobject`初始化，再调用`mallocgc`分配内存
5. make会根据make不同的类型来分配内存，底层也是调用`mallocgc`

