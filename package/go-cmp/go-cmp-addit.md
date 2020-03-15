# 包推荐: go-cmp之比较值 - 补充

## 简介

前面介绍了go-cmp的基本功能,但还有一点需要补充,非导出字段比较时会发生panic,官方没有出例子,我查看了issuse,找到了用法.

### 非导出字段的比较与忽略

如果有非导出字段,需要使用cmpopts.IgnoreUnexported或cmp.AllowUnexported其中type为包含非导出的结构体值

```golang

type A struct {
	a int
	B int
}

// note: 如果有非导出字段,需要使用cmpopts.IgnoreUnexported, 其中type为包含非导出的结构体值
func main() {
	a1 := A{
		a: 1,
		B: 0,
	}
	a2 := A{
		a: 2,
		B: 0,
	}

	fmt.Println(cmp.Equal(a1, a2, cmpopts.IgnoreUnexported(A{})))
	fmt.Println(cmp.Equal(a1, a2, cmp.AllowUnexported(A{})))
	// output:
	// true
	// false
}
```

## 总结

这是一个强大的比较库,扩展了reflect.DeepEqual方法,并可以自定义类型方法,各种定义选项满足需求,是非常符合go的大道致简的哲学.

## 参考

1. cmp GitHub: https://github.com/google/go-cmp

## 

欢迎关注我的微信公众号[佛系学习golang].
![公众号](../assets/qrcode_thinkgos_2.jpg)