# range注意事项

- 以下代码会会与预期不同

```go
func main() {
	arr := []int{1, 2, 3}
	newArr := []*int{}
	for _, v := range arr {
		newArr = append(newArr, &v)
	}
	for _, v := range newArr {
		fmt.Println(*v)
	}
}
$ go run main.go
3 3 3
```

- 清空数组时会使用一个优化，不用遍历数组中所有元素。即调用runtime的`memclrNoHeapPointers`函数(汇编代码)直接重置一块内存

```go
// 原代码
for i := range a {
	a[i] = zero
}

// 优化后
if len(a) != 0 {
	hp = &a[0]
	hn = len(a)*sizeof(elem(a))
	memclrNoHeapPointers(hp, hn)
	i = len(a) - 1
}
```

- 当使用`range`遍历的过程中追加新元素时，数组长度并不会无限扩张，因为数组原来的长度是预先取好的 ，并不是追加后的长度

```go
func main() {
	arr := []int{1, 2, 3}
	for _, v := range arr {
		arr = append(arr, v)
	}
	fmt.Println(arr)
}

$ go run main.go
1 2 3 1 2 3
```

- 当通过`range`循环时，若直接range的第二个元素指针，则会发生意料之外的情况

```go
func main() {
	arr := []int{1, 2, 3}
	newArr := []*int{}
	for _, v := range arr {
		newArr = append(newArr, &v)
	}
	for _, v := range newArr {
		fmt.Println(*v)
	}
}

$ go run main.go
3 3 3
```

这是因为在range时会生成一个临时变量，这个变量在使用过程中会被覆盖，也即`v2`会被复用，最后就存了原数组的最后一个值3，所以打印出来的全是3

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

也就是说`v2`(下面代码的`v`)会被循环使用，打印出来地址都一样

```go
func testRange() {
   nums := []int{2, 4, 6}
   for _, v := range nums {
      fmt.Printf("%v %p\n", v, &v)
   }
}
2 0xc00000c0f8
4 0xc00000c0f8
6 0xc00000c0f8
```

- 同时很重要的一点：使用range遍历数组时会拷贝一份进行遍历，遍历过程中的修改原来的数组不会影响到被迭代的数组。而range遍历遍历slice虽然也会进行拷贝，但拷贝的是指针，所以修改原slice会影响到被遍历的slice

```go
func main() {
   numbers2 := [...]int{1, 2, 3, 4, 5, 6}
   maxIndex2 := len(numbers2) - 1
   for i, e := range numbers2 {
      if i == maxIndex2 {
         numbers2[0] += e
      } else {
         numbers2[i+1] += e
      }
   }
   fmt.Println(numbers2)
   numbers3 := []int{1, 2, 3, 4, 5, 6}
   maxIndex2 = len(numbers3) - 1
   for i, e := range numbers3 {
      if i == maxIndex2 {
         numbers3[0] += e
      } else {
         numbers3[i+1] += e
      }
   }
   fmt.Println(numbers3)
}
输出：
[7 3 5 7 9 11]
[22 3 6 10 15 21]
```

