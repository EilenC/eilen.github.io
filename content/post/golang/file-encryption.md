---
title: "Golang 文件位运算与gob序列化"
date: 2023-12-30T00:07:30+08:00
draft: false
tags: ["golang"]
categories: ["golang","bitwise","gob"]
author: "EilenC"
---

> Golang 使用位运算加密文件并使用 `gob` 进行序列化



## 位运算

基本的位运算共 6 种，分别为 与、或、异或、取反、左移和右移。

| 运算 | 运算符 | Golang 运算符 | 解释                                             |
| :--- | :----: | :-----------: | :----------------------------------------------- |
| 与   |  `&`   |    `a & b`    | 只有两个对应位都为1时才为1                       |
| 或   |  `|`   |    `a | b`    | 只要两个对应位中有一个1时就为1                   |
| 异或 |  `^`   |    `a ^ b`    | 只有两个对应位不同时才为1                        |
| 取反 |  `~`   |     `^a`      | 将0变为1,将1变为0                                |
| 左移 |  `<<`  |    `a <<`     | 位全部左移若干位，左边的二进制位丢弃，右边补0    |
| 右移 |  `>>`  |    `a >>`     | 位全部右移若干位，正数左补0，负数左补1，右边丢弃 |



## Golang 位运算


code 

```go
func main() {
  a := int(5) // Binary: 00001100
  b := int(3) // Binary: 00001010
  // Bitwise AND
  fmt.Printf("%08b (%d) AND %08b (%d) : %08b (%d)\n", a, a, b, b, a&b, a&b)

  // Bitwise OR
  fmt.Printf("%08b (%d) OR %08b (%d) : %08b (%d)\n", a, a, b, b, a|b, a|b)

  // Bitwise XOR
  fmt.Printf("%08b (%d) XOR %08b (%d) : %08b (%d)\n", a, a, b, b, a^b, a^b)

  // Bitwise NOT
  fmt.Printf("NOT %08b (%d) : %08b (%d)\n", a, a, ^a, ^a)

  // Left Shift 2
  fmt.Printf("%08b (%d) Left Shift 2 : %08b (%d)\n", a, a, a<<2, a<<2)

  // Right Shift 2
  fmt.Printf("%08b (%d) Right Shift 2 : %08b (%d)\n", a, a, a>>2, a>>2)
}
```

output

```powershell
00000101 (5) AND 00000011 (3) : 00000001 (1)
00000101 (5) OR 00000011 (3) : 00000111 (7) 
00000101 (5) XOR 00000011 (3) : 00000110 (6)
NOT 00000101 (5) : -0000110 (-6)            
00000101 (5) Left Shift 2 : 00010100 (20)   
00000101 (5) Right Shift 2 : 00000001 (1) 
```

一般 **取反** 操作大多数语言为 `~`  符号  Golang 复用 `^` 符号,在双操作数中代表 **异或(XOR)** 在单操作数下代表**取反**。



## Golang 文件读取

所有文件在使用 Golang 代码编写的方式读取后 皆可以转换成 `[]byte` 同时每一位byte 也都可以进行 位运算。

以下是 Golang 中常用读取文件内容

- io.ReadAll()
- os.ReadFile()	



## 位运算混淆文件

可任意组合 6 种 位运算 对二进制文件内容进行混淆。



### 位运算混淆示例

code

```go
func main() {
  key := []uint8{1, 2, 3, 4}
  keyLength := len(key)

  // Original data
  data := []byte("Hello, world!")

  // Encrypt data
  encryptedData := make([]byte, len(data))
  for i, b := range data {
     // Left shift
     encryptedData[i] = b << 1

     // XOR
     encryptedData[i] = encryptedData[i] ^ key[i%keyLength]
  }

  // Decrypt data
  decryptedData := make([]byte, len(encryptedData))
  for i, b := range encryptedData {
     // XOR
     decryptedData[i] = b ^ key[i%keyLength]

     // Right shift
     decryptedData[i] = decryptedData[i] >> 1
  }

  // Print original data and encrypted data
  fmt.Println("Original data:", data)
  fmt.Println("Encrypted data:", encryptedData)

  // Print decrypted data
  fmt.Println("Decrypted data:", decryptedData)
}
```

output

```powershell
Original data: [72 101 108 108 111 44 32 119 111 114 108 100 33]
Encrypted data: [145 200 219 220 223 90 67 234 223 230 219 204 67]
Decrypted data: [72 101 108 108 111 44 32 119 111 114 108 100 33] 
```

**使用左移右移时要注意数据可能会因为变量范围导致越界从而无法还原数据**



## Golang Gob (序列化)

`gob` 是 Golang 语言中的一种用于序列化和反序列化数据的格式，它的全称是 "Go Object Notation"。与 JSON 或 XML 不同，`gob` 是一种二进制格式，专门用于在 Golang 程序之间传递数据。

### 特点

1. **Go 特定**：`gob` 是 Golang 语言特有的序列化格式，因此它更适用于在 Golang 程序之间传递数据。它通常用于在分布式系统中进行进程间通信，或在持久性存储中保存和加载数据。
2. **二进制格式**：与 JSON 或 XML 不同，`gob` 是一种二进制格式，它更为紧凑，效率更高。这使得它在传输大量数据时更为高效。
3. **类型安全**：`gob` 是类型安全的，它能够准确地保留和还原 Golang 语言中的数据结构，包括用户自定义的结构体、切片、映射等。这种类型安全性是 JSON 和 XML 等文本格式所不具备的。
4. **支持循环引用**：`gob` 支持序列化包含循环引用的数据结构，这在某些情况下是非常有用的。例如，一个对象引用了另一个对象，而后者又引用了第一个对象。
5. **适用场景**：`gob` 在 Golang 程序内部通信或数据存储的场景中表现出色，但在与其他语言通信时，可能会受到格式不兼容的限制。

### 示例

import package `encoding/gob`

code

```go
type Person struct {
  Name string
  Age  int
}

func main() {
  // Create a Person object
  person := Person{Name: "Alice", Age: 30}

  // Create a byte buffer to store the encoded data
  var buffer bytes.Buffer

  // Create an encoder
  encoder := gob.NewEncoder(&buffer)

  // Register the type to be serialized
  gob.Register(Person{})

  // Encode and store the data
  err := encoder.Encode(person)
  if err != nil {
  	fmt.Println("Error encoding data:", err)
  	return
  }

  fmt.Println("Encoded data:", buffer.String())

  // Create a decoder
  decoder := gob.NewDecoder(&buffer)

  // Create an object to store the decoded data
  var decodedPerson Person

  // Decode the data
  err = decoder.Decode(&decodedPerson)
  if err != nil {
  	fmt.Println("Error decoding data:", err)
  	return
  }

  // Print the original data and the decoded data
  fmt.Println("Original Person:", person)
  fmt.Println("Decoded Person:", decodedPerson)
}
```

output

```powershell
Encoded data: $Person�� Name
                              Age   
                                    ��Alice<                                                                                                                                                         
Original Person: {Alice 30}
Decoded Person: {Alice 30}
```



## 位运算配合Gob进行序列化

code

```go
type File struct {
	Data []byte
}

func main() {
  key := []uint8{1, 2, 3, 4}
  keyLength := len(key)

  // Original data
  data := []byte("Hello, world!")

  // Encrypt data
  encryptedData := make([]byte, len(data))
  for i, b := range data {
  	// XOR
  	encryptedData[i] = b ^ key[i%keyLength]
  }

  // Print original data and encrypted data
  fmt.Println("Original data:", data)
  fmt.Println("Encrypted data:", encryptedData)

  // Create a Person object
  encFile := File{Data: encryptedData}

  // Create a byte buffer to store the encoded data
  var buffer bytes.Buffer

  // Create an encoder
  encoder := gob.NewEncoder(&buffer)

  // Register the type to be serialized
  gob.Register(File{})

  // Encode and store the data
  err := encoder.Encode(encFile)
  if err != nil {
  	fmt.Println("Error encoding data:", err)
  	return
  }

  fmt.Println("Encoded data:", buffer.String())

  // Create a decoder
  decoder := gob.NewDecoder(&buffer)

  // Create an object to store the decoded data
  var decFile File

  // Decode the data
  err = decoder.Decode(&decFile)
  if err != nil {
  	fmt.Println("Error decoding data:", err)
  	return
  }

  // Print the original data and the decoded data
  fmt.Println("Gob Encoded File.Data:", encFile.Data)
  fmt.Println("Gob Decoded File.Data:", decFile.Data)
}
```

output

```powershell
Original data: [72 101 108 108 111 44 32 119 111 114 108 100 33]
Encrypted data: [73 103 111 104 110 46 35 115 110 112 111 96 32]       
Encoded data: File�� Data                                  
Igohn.#snpo`                                                           
Gob Encoded File.Data: [73 103 111 104 110 46 35 115 110 112 111 96 32]
Gob Decoded File.Data: [73 103 111 104 110 46 35 115 110 112 111 96 32]
```

首先使用位运算混淆得到 `Encrypted` 再将 `[]byte` 存放进 File Sturct 最后使用 Gob Encode 进行序列化。

## 总结

Golang 位运算相对其他语言最大的不同在于 `取反`。

Gob 为 Golang 自带的标准库 安装与使用相对简单 实测可以序列化存储 []byte 可作为存储资源文件的方案。

