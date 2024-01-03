---
title: "Generics Http Request"
date: 2023-12-18T23:54:36+08:00
draft: false
tags: ["golang"]
categories: ["golang","generics","gin"]
author: "EilenC"
---

> Golang 泛型使用在HTTP Request 应用示例

## Request 请求参数

通常使用 `go` 编写 http 服务总会需要定义接口接收的参数, 前端上传(接口请求)通常是将数据上传方式修改为
`application/json`。

同理在 `go` 中也通常使用自定义一个struct进行接收, 使用类似 `json.Unmarshal()` 对上传的 `json` 字符串进行反序列化将对应的key通过`struct.tag`
方式映射到对应 `field` 上。

### 经典示例 

#### net/http

```go
type Releases struct {
  Version string `json:"version"`
  Name    string `json:"name"`
}

func ReleasesApi(w http.ResponseWriter, r *http.Request) {
  //get upload param
  b, err := io.ReadAll(r.Body)
  if err != nil {
    panic(err)
    return
  }
  defer func(Body io.ReadCloser) {
    _ = Body.Close()
  }(r.Body)
  req := ReleasesPkg{}
  err = json.Unmarshal(b, &req)
  if err != nil {
    panic(err)
    return
  }
  return
}
```

#### gin
```go
type Releases struct {
  Version string `json:"version"`
  Name    string `json:"name"`
}

func ReleasesApi(ctx *gin.Context) {
  //get upload param
  req := Releases{} 
  err := ctx.ShouldBindJSON(&req)
  if err != nil {
    panic(err)
    return
  }
  return
}
```

以上方式都可以做到将json请求参数反序列化到变量 `req` 。

### 规范请求 json 结构

实际开发中遇到有固定请求层级的 `json` 规范 例如

```json
{
  "funcId": "",
  "mid": "",
  "sn": "",
  "payload": {
    "params": {
      "version": "v1.0.0",
      "name": "test"
    },
    "page":1,
    "page_size": 2
  }
}
```
对应 `go` struct
```go
type Request struct {
  FuncId  string `json:"funcId"`
  Mid     string `json:"mid"`
  Sn      string `json:"sn"`
  Payload struct {
    Params struct {
      Version string `json:"version"`
      Name    string `json:"name"`
    } `json:"params"`
    Page     int `json:"page"`
    PageSize int `json:"page_size"`
  } `json:"payload"`
}
```

其中规范除 `Params` 之外全部一致,这样就无法复用通用结构体除 `Params` 之外 `field` 例如

```go
type Request struct {
  FuncId  string `json:"funcId"`
  Mid     string `json:"mid"`
  Sn      string `json:"sn"`
  Payload struct {
    Params any `json:"params"`
    Page     int `json:"page"`
    PageSize int `json:"page_size"`
  } `json:"payload"`
}

type Releases struct {
  Version string `json:"version"`
  Name    string `json:"name"`
}
```

### 解决方案

#### 1. 反复序列化
```go
func ReleasesApi(ctx *gin.Context) {
  var (
    err     error
    message Request
    req     Releases
  )
  b, err := io.ReadAll(ctx.Request.Body)
  if err != nil {
    panic(err)
    return
  }
  //first Unmarshal get params
  err = json.Unmarshal(b, &message)
  if err != nil {
    panic(err)
    return
  }
  b, err = json.Marshal(message.Payload.Params)
  if err != nil {
    panic(err)
    return
  }
  err = json.Unmarshal(b, &req)
  if err != nil {
    panic(err)
    return
  }
  fmt.Println(message)
  fmt.Println(req)
  return
}
```
第一层解析通用结构体 `Request` 再二次进行 解析 `Releases`

##### 优点
 - 没有额外的引入
 - 全部使用标准库

##### 缺点
 - 代码蹩脚
 - 反复序列化影响性能,占用CPU资源


#### 2. 反射 [mapstructure](https://github.com/mitchellh/mapstructure)
```go
func ReleasesApi(ctx *gin.Context) {
  var (
    err     error
    message Request
    req     Releases
  )
  b, err := io.ReadAll(ctx.Request.Body)
  if err != nil {
    panic(err)
    return
  }
  message.Payload.Params = req
  //first Unmarshal get params
  err = json.Unmarshal(b, &req)
  if err != nil {
    panic(err)
    return
  }
  err = mapstructure.Decode(message.Payload.Params, &req)
  if err != nil {
    panic(err)
    return
  }
  fmt.Println(message)
  fmt.Println(req)
  return
}
```
##### 优点
- 减少反复序列化
- 代码相对整洁

##### 缺点
- 引入第三方库(稳定性考虑)
- 第三方库使用依然使用反射机制

#### 3. 泛型
```go
type Request[T any] struct {
  FuncId  string `json:"funcId"`
  Mid     string `json:"mid"`
  Sn      string `json:"sn"`
  Payload struct {
    Params T `json:"params"`
  } `json:"payload"`
}
func ReleasesApi(ctx *gin.Context) {
  var (
    err     error
    req     Request[Releases]
  )
  err = ctx.ShouldBindJSON(&req)
  if err != nil {
    panic(err)
    return
  }
  fmt.Println(req)
  return
}
```
使用泛型将 `Request` 进行改造 使其达到 `Params` 这个 `field` 能兼容任意类型。

类似 `gin` 中 `ShouldBind()` 这个默认通用的 Bind struct 是无效的, 需要强行执行 `ShouldBindJSON()`。

##### 优点
- 无第三方依赖
- 代码整洁

##### 缺点
- go版本需要从 1.18 起(支持泛型最低版本)
- go泛型目前还处于类似 beta 阶段需要考虑是否使用
- 配合第三方框架需要注意是否能进行正常的绑定数据

## 总结

 - 允许使用泛型方式的前提下,优先使用泛型
 - 优先使用顺序为 泛型>反射>反复序列化
