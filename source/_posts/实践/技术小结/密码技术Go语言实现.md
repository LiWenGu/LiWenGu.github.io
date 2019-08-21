---
title: 密码技术Go语言实现
date: 2019-08-21 23:25:00
updated: 2019-08-21 23:25:00
tags:
  - 密码
categories: 
  - 小结
comments: true
permalink: tech/algo_go_1.html 
---

# 0. 基础

加密三要素：
- 明文/密文
- 秘钥
- 算法

CBC模式：分组模式的一种
>1. 明文经过填充，分块后和秘钥长度相同，接着第一块明文和初始向量进行数学运算后产生第一个密文块 A1  
>2. A1 与秘钥 XOR 得到第二个密文块 A2  
>3. 上一个密文块和秘钥 XOR 得到当前密文块
>4. 重复第三步，直到将全部明文加密完成 

# 1. 对称密码

- 明文分组后需要填充，密文解密时需要去掉填充数据  
- 使用哪种分组模式（CBC）

## 1.1 DES

DES（Data Encryption Standard，数据加密标准）算法，作为一个标准已经被高级加密标准 AES 取代，所以也被称为 DEA（Data Encryption Algorithm，数据加密算法）  
  
其秘钥长度为 56Bit（8 Bit = 1 byte）,每隔 7 Bit 会设置一个用于错误检查的比特位，即一共 `56+8=64Bit=8byte`  
其算法过程：
  - 假如明文长度为 64 byte
  - 将明文分组，每组长度=秘钥的长度，即 8 byte
  - 将每组数据和秘钥进行位运算
  - 得到每组的密文长度=明文长度
  - 如果分组后的长度小于秘钥长度，则进行填充

其中位运算Go密码框架已经为我们实现了。

## 1.2 3DES

三重数据加密算法（Triple Data Encryption Algorithm，缩写为TDEA，Triple DEA），或称3DES（Triple DES），是一种对称密钥加密块密码，相当于是对每个数据块应用三次数据加密标准（DES）算法。由于计算机运算能力的增强，原版DES密码的密钥长度变得容易被暴力破解；3DES即是设计用来提供一种相对简单的方法，即通过增加DES的密钥长度来避免类似的攻击，而不是设计一种全新的块密码算法。

要求需要3个秘钥，每个秘钥长度为 8 byte，明文分组后每组的数据长度和秘钥长度必须相同，同为 8 byte

加密过程：
  - 对分组数据使用 key1 加密得到 A1
  - 对 A1 使用 key2 解密得到 A2
  - 对 A2 使用 key3 加密得到最终的密文
![][1]

>为什么能兼容 DES 密码算法？  
假如 key1、2、3都不同：加密三次  
假如 key1、2、3都相同：使用秘钥 key1 加密一次（和 DES 相同）


## 1.3 AES

高级加密标准（英语：Advanced Encryption Standard，缩写：AES），在密码学中又称 `Rijndael` 加密法，是美国联邦政府采用的一种区块加密标准。这个标准用来替代原先的 `DES`。

要求秘钥长度：16 byte 或 24 byte 或 32 byte  
明文分组后每组的长度必须和秘钥长度相同  
在 `go` 中密码算法框架指定死了秘钥长度为 16 byte  

## 1.4 工程实现

算法核心类：  
```go
package src

import (
	"bytes"
	"crypto/aes"
	"crypto/cipher"
	"crypto/des"
)

// 填充最后一个分组的函数
// src - 原始函数
// blockSize - 每个分组的数据长度
func paddingText(src []byte, blockSize int) []byte {
	// 1. 求出最后一个分组要填充多少个字节
	padding := blockSize - len(src)%blockSize
	// 2. 创建新的切片，切片的字节数为 padding，并初始化，每个字节的值为 padding
	padText := bytes.Repeat([]byte{byte(padding)}, padding)
	// 3. 将创建出的新切片和原始数据进行连接
	newText := append(src, padText...)
	// 4. 返回新的字符串
	return newText
}

// 删除末尾填充的字节
func unPaddingText(src []byte) []byte {
	// 1. 求出要处理的切片的长度
	len := len(src)
	// 2. 取出最后一个字符
	number := int(src[len-1])
	// 3. 将切片末尾的number个字节删除
	newText := src[:len-number]
	return newText
}

// 使用 des 加密
// src - 明文
// key - 秘钥
func encryptoDES(src []byte, key []byte) []byte {
	// 1. 创建并返回一个使用 DES 算法的 cipher.Block 接口
	block, err := des.NewCipher(key)
	if err != nil {
		panic(err)
	}
	// 2. 对最后一个明文分组填充
	src = paddingText(src, block.BlockSize())
	// 3. 创建一个密码分组为链接模式，底层使用 DES 加密的 BlockMode 接口
	// 初始化向量，必须长度和秘钥长度相同
	iv := []byte("aaaabbbb")
	blockMode := cipher.NewCBCEncrypter(block, iv)
	// 4. 加密连续的数据块
	dst := make([]byte, len(src))
	blockMode.CryptBlocks(dst, src)
	return dst
}

// 使用 des 解密
func decryptoDES(src []byte, key []byte) []byte {
	// 1. 创建并返回一个使用 DES 算法的 cipher.Block 接口
	block, err := des.NewCipher(key)
	if err != nil {
		panic(err)
	}
	// 2. 创建一个密码分组为链接模式的，底层使用 DES 解密的 BlockMode 接口
	iv := []byte("aaaabbbb")
	blockMode := cipher.NewCBCDecrypter(block, iv)
	// 3. 数据块解密
	dst := make([]byte, len(src))
	blockMode.CryptBlocks(dst, src)
	// 4. 去掉最后一组的填充数据
	newText := unPaddingText(dst)
	return newText
}

func encrypto3DES(src, key []byte) []byte {
	// 1. 创建并返回一个使用 3DES 算法的 cipher.Block 接口
	block, err := des.NewTripleDESCipher(key)
	if err != nil {
		panic(err)
	}
	src = paddingText(src, block.BlockSize())
	// 初始化向量，必须长度和秘钥长度相同
	iv := []byte("aaaabbbb")
	// 2. 创建一个密码分组为链接模式的，底层使用 3DES 加密的 BlockMode 接口
	blockMode := cipher.NewCBCEncrypter(block, iv)
	dst := make([]byte, len(src))
	blockMode.CryptBlocks(dst, src)
	return dst
}

func decrypto3DES(src, key []byte) []byte {
	// 1. 创建并返回一个使用 3DES 算法的 cipher.Block 接口
	block, err := des.NewTripleDESCipher(key)
	if err != nil {
		panic(err)
	}
	// 初始化向量，必须长度和秘钥长度相同
	iv := []byte("aaaabbbb")
	// 2. 创建一个密码分组为链接模式的，底层使用 3DES 解密的 BlockMode 接口
	blockMode := cipher.NewCBCDecrypter(block, iv)
	dst := make([]byte, len(src))
	blockMode.CryptBlocks(dst, src)
	newText := unPaddingText(dst)
	return newText
}

func encryptoAES(src, key []byte) []byte {
	// 1. 创建并返回一个使用 AES 算法的 cipher.Block 接口
	block, err := aes.NewCipher(key)
	if err != nil {
		panic(err)
	}
	src = paddingText(src, block.BlockSize())
	// 初始化向量，必须长度和秘钥长度相同
	iv := []byte("aaaabbbbccccdddd")
	blockMode := cipher.NewCBCEncrypter(block, iv)
	dst := make([]byte, len(src))
	blockMode.CryptBlocks(dst, src)
	return dst
}

func decryptoAES(src, key []byte) []byte {
	block, err := aes.NewCipher(key)
	if err != nil {
		panic(err)
	}
	// 初始化向量，必须长度和秘钥长度相同
	iv := []byte("aaaabbbbccccdddd")
	blockMode := cipher.NewCBCDecrypter(block, iv)
	dst := make([]byte, len(src))
	blockMode.CryptBlocks(dst, src)
	newText := unPaddingText(dst)
	return newText
}
```

算法测试类：  
```go
package src

import (
	"fmt"
	"testing"
)

func TestDES(t *testing.T) {
	src := []byte("我是一个不知道有多长的明文")
	key := []byte("12345678")
	str := encryptoDES(src, key)
	fmt.Println("加密之后的密文：" + string(str))
	str = decryptoDES(str, key)
	fmt.Println("解密之后的明文：" + string(str))
}

func Test3DES(t *testing.T) {
	src := []byte("我是一个不知道有多长的明文")
	key := []byte("12345678abcdefgh87654321")
	str := encrypto3DES(src, key)
	fmt.Println("加密之后的密文：" + string(str))
	str = decrypto3DES(str, key)
	fmt.Println("解密之后的明文：" + string(str))
}

func TestAES(t *testing.T) {
    src := []byte("我是一个不知道有多长的明文")
    // TODO 这里会报错，因为秘钥长度是 17 byte，去掉一个字符就好了
	key := []byte("1234abcd4321dcbaw")
	str := encryptoAES(src, key)
	fmt.Println("加密之后的密文：" + string(str))
	str = decryptoAES(str, key)
	fmt.Println("解密之后的明文：" + string(str))
}
```

# 2. 非对称密码

[1]: https://community.cisco.com/legacyfs/online/legacy/9/6/8/112869-des3p1.gif