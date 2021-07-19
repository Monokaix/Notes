# defer

1. `defer`共有三种实现机制
    - 堆上分配 · 1.1 ~ 1.12
        - 编译期将 `defer` 关键字转换成 [`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc) 并在调用 `defer` 关键字的函数返回之前插入 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn)；
        - 运行时调用 [`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc) 会将一个新的 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体追加到当前 Goroutine 的链表头；
        - 运行时调用 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 会从 Goroutine 的链表中取出 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构并依次执行；
        - [`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc) 负责创建新的延迟调用；
        - [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 负责在函数调用结束时执行所有的延迟调用；
    - 栈上分配 · 1.13
        - 当该关键字在函数体中最多执行一次时，编译期间的 [`cmd/compile/internal/gc.state.call`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.call) 会将结构体分配到栈上并调用 [`runtime.deferprocStack`](https://draveness.me/golang/tree/runtime.deferprocStack)；
    - 开放编码 · 1.14 ~ 现在
        - 编译期间判断 `defer` 关键字、`return` 语句的个数确定是否开启开放编码优化；
        - 通过 `deferBits` 和 [`cmd/compile/internal/gc.openDeferInfo`](https://draveness.me/golang/tree/cmd/compile/internal/gc.openDeferInfo) 存储 `defer` 关键字的相关信息；
        - 如果 `defer` 关键字的执行可以在编译期间确定，会在函数返回前直接插入相应的代码，否则会由运行时的 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 处理；
        - `deferBits` 会记录哪些`defer`需要执行（因为有些`defer`在if语句里）
        - 延迟比特的作用就是标记哪些 `defer` 关键字在函数中被执行，这样在函数返回时可以根据对应 `deferBits` 的内容确定执行的函数，而正是因为 `deferBits` 的大小仅为 8 比特，所以该优化的启用条件为函数中的 `defer` 关键字少于 8 个。
2. 两个现象
    - 后调用的`defer`函数会先执行：
        - 后调用的 `defer` 函数会被追加到 Goroutine `_defer` 链表的最前面；
        - 运行 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 时是从前到后依次执行；
    - 函数的参数会被预先计算；
        - 调用 [`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc) 函数创建新的延迟调用时就会立刻拷贝函数的参数，函数的参数不会等到真正执行时计算；也就是说执行到`defer`时会把函数参数进行执行然后拷贝，所以不会等到返回前再执行

