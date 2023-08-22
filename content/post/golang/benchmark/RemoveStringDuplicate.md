---
title: "Golang 4 种方案对字符串切片去重基准测试"
date: 2022-04-04T19:37:56+08:00
lastmod: 2022-04-04T19:37:56+08:00
draft: false
tags: ["golang", "benchmark"]
categories: ["golang"]
author: "EilenC"
#copyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh" target="_blank">我是自定义文章版权声明</a>'
---

> 对字符串切片进行去重 不同方式的基准测试

## 测试环境
- OS: Windows 10 1909
- CPU: AMD Ryzen 7 3700X 8-Core Processor
- Memory: 16G

## 方法

### 标准库方式
#### [Russ Cox 源代码链接](https://go-review.googlesource.com/c/go/+/243941/8/src/go/build/build.go#1024)
```go
//RemoveStringDuplicateUseCopy 切片元素去重并排序 list 待去重的切片
func RemoveStringDuplicateUseCopy(list []string) []string {
	if list == nil {
		return nil
	}
	out := make([]string, len(list))
	copy(out, list)
	sort.Strings(out)
	uniq := out[:0]
	for _, x := range out {
		if len(uniq) == 0 || uniq[len(uniq)-1] != x {
			uniq = append(uniq, x)
		}
	}
	return uniq
}
```

### 修改标准库(破坏原Slices)
```go
//RemoveStringDuplicateNotCopy 切片元素去重并排序  破坏原list 待去重的切片
func RemoveStringDuplicateNotCopy(list []string) []string {
	if list == nil {
		return nil
	}
	sort.Strings(list)
	uniq := list[:0]
	for _, x := range list {
		if len(uniq) == 0 || uniq[len(uniq)-1] != x {
			uniq = append(uniq, x)
		}
	}
	return uniq
}
```

### 使用Map方式进行去重
```go
//RemoveStringDuplicateUseMap 切片元素去重 list 待去重的切片
func RemoveStringDuplicateUseMap(list []string) []string {
	var data []string
	rd := map[string]struct{}{}
	for _, v := range list {
		if _, ok := rd[v]; !ok { //通过map内是否存在对应key值去添加对应切片内元素
			rd[v] = struct{}{}
			data = append(data, v)
		}
	}
	return data
}
```

### 多次循环去重
```go
//RemoveStringDuplicateUseFor 数组去重 暴力循环
func RemoveStringDuplicateUseFor(a []string) []string {
	var res []string
	for _, i := range a {
		if len(res) == 0 {
			res = append(res, i)
		} else {
			for k, v := range res {
				if i == v {
					break
				}
				if k == len(res)-1 {
					res = append(res, i)
				}
			}
		}
	}
	return res
}
```

## 测试

### 用例随机生成字符串进行测试
```go
func GetRandomString(l int) []byte {
	str := "0123456789abcdefghijklmnopqrstuvwxyz"
	bytes := []byte(str)
	var result []byte
	r := rand.New(rand.NewSource(time.Now().UnixNano()))
	for i := 0; i < l; i++ {
		result = append(result, bytes[r.Intn(len(bytes))])
	}
	return result
}
```
### benchmark
```go
func Benchmark_RemoveStringDuplicateUseCopy(b *testing.B) {
	testString, _ := ioutil.ReadFile("./test.txt")
	test := make([]string, len(testString))
	for i, v := range testString {
		test[i] = string(v)
	}
	for i := 0; i < b.N; i++ {
		RemoveStringDuplicateUseCopy(test)
	}
}

func Benchmark_RemoveStringDuplicateUseMap(b *testing.B) {
	testString, _ := ioutil.ReadFile("./test.txt")
	test := make([]string, len(testString))
	for i, v := range testString {
		test[i] = string(v)
	}
	for i := 0; i < b.N; i++ {
		RemoveStringDuplicateUseMap(test)
	}
}

func Benchmark_RemoveStringDuplicateUseFor(b *testing.B) {
	testString, _ := ioutil.ReadFile("./test.txt")
	test := make([]string, len(testString))
	for i, v := range testString {
		test[i] = string(v)
	}
	for i := 0; i < b.N; i++ {
		RemoveStringDuplicateUseFor(test)
	}
}

func Benchmark_RemoveStringDuplicateNotCopy(b *testing.B) {
	testString, _ := ioutil.ReadFile("./test.txt")
	test := make([]string, len(testString))
	for i, v := range testString {
		test[i] = string(v)
	}
	for i := 0; i < b.N; i++ {
		RemoveStringDuplicateNotCopy(test)
	}
}
```
### 999长度随机字符 结果
```shell
goos: windows
goarch: amd64
pkg: goProject/test
cpu: AMD Ryzen 7 3700X 8-Core Processor
Benchmark_RemoveStringDuplicateUseCopy
Benchmark_RemoveStringDuplicateUseCopy-16          15592             76321 ns/op
           16409 B/op          2 allocs/op

Benchmark_RemoveStringDuplicateUseMap
Benchmark_RemoveStringDuplicateUseMap-16           44970             26773 ns/op
            4226 B/op         11 allocs/op

Benchmark_RemoveStringDuplicateUseFor
Benchmark_RemoveStringDuplicateUseFor-16           17068             70180 ns/op
            2033 B/op          7 allocs/op

Benchmark_RemoveStringDuplicateNotCopy
Benchmark_RemoveStringDuplicateNotCopy-16          85106             13075 ns/op
              24 B/op          1 allocs/op
```

### 10000长度随机字符 结果
```shell
goos: windows
goarch: amd64
pkg: goProject/test
cpu: AMD Ryzen 7 3700X 8-Core Processor
Benchmark_RemoveStringDuplicateUseCopy
Benchmark_RemoveStringDuplicateUseCopy-16            146           8212651 ns/op
         1620140 B/op        687 allocs/op

Benchmark_RemoveStringDuplicateUseMap
Benchmark_RemoveStringDuplicateUseMap-16             504           2317023 ns/op
            8414 B/op        209 allocs/op

Benchmark_RemoveStringDuplicateUseFor
Benchmark_RemoveStringDuplicateUseFor-16             169           7134641 ns/op
           14534 B/op        598 allocs/op

Benchmark_RemoveStringDuplicateNotCopy
Benchmark_RemoveStringDuplicateNotCopy-16            267           4471912 ns/op
            7937 B/op        375 allocs/op
```

## 结论
 - String Slices 不是特别长,几百个的情况下使用 [RemoveStringDuplicateNotCopy()](https://eilenc.github.io/post/golang/benchmark/removestringduplicate/#%E4%BF%AE%E6%94%B9%E6%A0%87%E5%87%86%E5%BA%93%E7%A0%B4%E5%9D%8F%E5%8E%9Fslices) 进行处理,有速度跟内存上的优势
 - String Slices 长度超过10000 后使用 Map[RemoveStringDuplicateUseMap](https://eilenc.github.io/post/golang/benchmark/removestringduplicate/#%E4%BD%BF%E7%94%A8map%E6%96%B9%E5%BC%8F%E8%BF%9B%E8%A1%8C%E5%8E%BB%E9%87%8D) 方式最佳
