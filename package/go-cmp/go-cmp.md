# 包推荐: go-cmp之比较值

## 特性

- 当相等的默认行为不适合比较时,可以自定义Equal函数来定义相等测试的方法。
  例如,可以设置浮点数具有容限范围内的Equal函数,就可以认为容限范围内的浮点数是相等.
- 定义类型的Equal方法, 包将对这个类型执行所定义的Equal方法进行相等测试判断.
- 如果没有任何自定义Equal函数或选项,那么包将递归比较基本类型,类似reflect.DeepEqual函数,不同的是非导出字段是不比较的. 会引发panic. 但可以通过自定义option来决定是否进行判断或忽略来防止发生panic.

google出品,必属精品,这是一个强大的比较库,完美的go哲学.

## 快速使用

先安装:

```bash
    go get github/google/go-cmp
```

然后导入:

```bash
    import "github/google/go-cmp/cmp"
```

## 使用例子 - 官方的

### 符点数

自定义一个Compare选项,使float64在比较时,使符合精度范围内的值是相等的

```golang
    // 定义符点数的相等测试选项,在精度范围认为是相等的
	opt := cmp.Comparer(func(x, y float64) bool {
		delta := math.Abs(x - y)
		mean := math.Abs(x+y) / 2.0
		return delta/mean < 0.00001
	})
	x := []float64{1.0, 1.1, 1.2, math.Pi}
	y := []float64{1.0, 1.1, 1.2, 3.14159265359} // 符合精度值的Pi
	z := []float64{1.0, 1.1, 1.2, 3.1415}        // 不符合精度值的 Pi
	fmt.Println(cmp.Equal(x, y, opt))
	fmt.Println(cmp.Equal(y, z, opt))
	fmt.Println(cmp.Equal(z, x, opt))
	// output
	// true
	// false
	// false
```

### 避免调用Equal方法

自定义Transformer进行类型转换,避免cmp包调用类型的Equal方法.

```golang
// 如果在类型上定义的Equal方法不合适，可以动态地转换类型，不执行Equal方法(或任何方法)。
type otherString string
type myString otherString

// note: 自定义类型的Equal方法
func (sf otherString) Equal(s otherString) bool {
	return strings.EqualFold(string(sf), string(s))
}

func main() {
	// NOTE: 如果不想让指定类型进行执行equal方法,可以进行类型转换.
	trans := cmp.Transformer("", func(in otherString) myString {
		return myString(in)
	})

	x := []otherString{"foo", "bar", "baz"}
	y := []otherString{"fOO", "bAr", "Baz"}

	fmt.Println(cmp.Equal(x, y))        // 相等,并且不区分大小写
	fmt.Println(cmp.Equal(x, y, trans)) // 不相等,类型转换过,不会执行equal方法
	// output
	// true
	// false
}
```

### 空切片或空map认为与长度为0的切片或map相等

有时候一个空map或空切片被认为等于长度为0的Map或切片,例子只是展示option用法,可使用cmpopts.EquateEmpty代替

```golang
	alwaysEqual := cmp.Comparer(func(_, _ interface{}) bool { return true })

	// This option handles slices and maps of any type.
	opt := cmp.FilterValues(func(x, y interface{}) bool {
		vx, vy := reflect.ValueOf(x), reflect.ValueOf(y)
		return (vx.IsValid() && vy.IsValid() && vx.Type() == vy.Type()) &&
			(vx.Kind() == reflect.Slice || vx.Kind() == reflect.Map) &&
			(vx.Len() == 0 && vy.Len() == 0)
	}, alwaysEqual)

	type S struct {
		A []int
		B map[string]bool
	}
	x := S{nil, make(map[string]bool, 100)}
	y := S{make([]int, 0, 200), nil}
	z := S{[]int{0}, nil} // []int has a single element (i.e., not empty)
	fmt.Println(cmp.Equal(x, y, opt))
	fmt.Println(cmp.Equal(y, z, opt))
	fmt.Println(cmp.Equal(z, x, opt))
	// output
	// true
	// false
	// false
```

### 与NaNs比较

正常的浮点运算将NaN与自身进行比较时，将==定义为false。 在某些情况下，这不是所需的属性。这个例子是为了说明的目的。可使用cmpopts.EquateNaNs代替

```golang
    // Comparer 只比较float64,如果操作float32s,定义一个类型的Equal方法,或采用Transformer转换float32为float64
	opt := cmp.Comparer(func(x, y float64) bool {
		return (math.IsNaN(x) && math.IsNaN(y)) || x == y
	})

	x := []float64{1.0, math.NaN(), math.E, -0.0, +0.0}
	y := []float64{1.0, math.NaN(), math.E, -0.0, +0.0}
	z := []float64{1.0, math.NaN(), math.Pi, -0.0, +0.0} // Pi constant instead of E

	fmt.Println(cmp.Equal(x, y, opt))
	fmt.Println(cmp.Equal(y, z, opt))
	fmt.Println(cmp.Equal(z, x, opt))
	//output
	//true
	//false
	//false
```

###  NaNs与符点数误差设置相结合

为了使浮点比较将NaNs等于其自身以及近似相等的值结合在一起，需要使用过滤器来限制比较的范围，以便它们可以组合。这个例子是为了说明的目的。 可使用cmpopts.EquateNaNs和cmpopts.EquateApprox。

```golang
alwaysEqual := cmp.Comparer(func(_, _ interface{}) bool { return true })

	opts := cmp.Options{
		// This option declares that a float64 comparison is equal only if
		// both inputs are NaN.
		cmp.FilterValues(func(x, y float64) bool {
			return math.IsNaN(x) && math.IsNaN(y)
		}, alwaysEqual),

		// This option declares approximate equality on float64s only if
		// both inputs are not NaN.
		cmp.FilterValues(func(x, y float64) bool {
			return !math.IsNaN(x) && !math.IsNaN(y)
		}, cmp.Comparer(func(x, y float64) bool {
			delta := math.Abs(x - y)
			mean := math.Abs(x+y) / 2.0
			return delta/mean < 0.00001
		})),
	}
	x := []float64{math.NaN(), 1.0, 1.1, 1.2, math.Pi}
	y := []float64{math.NaN(), 1.0, 1.1, 1.2, 3.14159265359} // 符合精度值的Pi
	z := []float64{math.NaN(), 1.0, 1.1, 1.2, 3.1415}        // 不符合精度值的Pi
	fmt.Println(cmp.Equal(x, y, opts))
	fmt.Println(cmp.Equal(y, z, opts))
	fmt.Println(cmp.Equal(z, x, opts))
	//output
	//true
	//false
	//false
```

### 切片值的比较

如果两个切片具有相同的元素,则无论它们出现的顺序如何,都可以视为相等.可以使用对切片进行排序。这个例子是为了说明的目的。 note: 使用cmpopts.SortSlices代替.

```golang
	// This Transformer sorts a []int.
	trans := cmp.Transformer("Sort", func(in []int) []int {
		out := append([]int(nil), in...) // 拷贝输入,防止外部改变
		sort.Ints(out)
		return out
	})

	x := struct{ Ints []int }{[]int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}}
	y := struct{ Ints []int }{[]int{2, 8, 0, 9, 6, 1, 4, 7, 3, 5}}
	z := struct{ Ints []int }{[]int{0, 0, 1, 2, 3, 4, 5, 6, 7, 8}}

	fmt.Println(cmp.Equal(x, y, trans))
	fmt.Println(cmp.Equal(y, z, trans))
    fmt.Println(cmp.Equal(z, x, trans))
    // output:
	// true
	// false
	// false
```

## 总结

这是一个强大的比较库,扩展了reflect.DeepEqual方法,并可以自定义类型方法,各种定义选项满足需求,是非常符合go的大道致简的哲学.

## 参考

1. cmp GitHub: https://github.com/google/go-cmp