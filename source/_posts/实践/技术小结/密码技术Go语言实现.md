---
title: 密码技术Go语言实现
date: 2019-08-21 23:25:00
updated: 2019-08-21 23:25:00
tags:
  - 密码
categories: 
  - 实践
  - 技术小结
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

## 2.1 相关知识

关键在于解决了秘钥的配送问题   
核心在于 RSA 算法以及 X509标准、Pem 格式文件

生成私钥：

1. 使用 rsa 生成私钥
2. 通过 x509 标准序列化为 ASN.1 的 DER 编码字符串
3. 将私钥字符串设置到 pem 格式块中
4. 通过 pem 将设置好的数据进行编码，写入文件中

生成公钥：

1. 从私钥对象取出公钥信息
2. 通过 x509 标准将得到的 rsa 公钥序列化为字符串
3. 将公钥字符串设置到 pem 格式块中
4. 通过 pem 将设置好的数据进行编码，写入文件中

## 2.2 工程实现

算法核心类：
```go
package src

import (
	"crypto/rand"
	"crypto/rsa"
	"crypto/x509"
	"encoding/pem"
	"os"
)

// 生成非对称秘钥对
func RsaGenKey(bits int) {
	// 1. 使用 rsa 的 GenerateKey 方法生成私钥
	privateKey, err := rsa.GenerateKey(rand.Reader, bits)
	if err != nil {
		panic(err)
	}
	// 2. 通过 x509 标准将得到的 rsa 私钥序列化为 ASN.1 的 DER 编码字符串
	privateStream := x509.MarshalPKCS1PrivateKey(privateKey)
	// 3. 将私钥字符串设置到 pem 格式块中
	block := pem.Block{
		Type:    "RSA Private Key",
		Headers: nil,
		Bytes:   privateStream,
	}
	// 4. 通过 pem 将设置好的数据进行编码，写入文件
	privateFile, err := os.Create("private.pem")
	if err != nil {
		panic(err)
	}
	err = pem.Encode(privateFile, &block)
	if err != nil {
		panic(err)
	}
	defer privateFile.Close()

	// 1. 从得到的私钥对象中将公钥信息取出
	pubKey := privateKey.PublicKey
	// 2. 通过 x509 标准将得到的 rsa 公钥序列化为字符串
	pubStream := x509.MarshalPKCS1PublicKey(&pubKey)
	// 3. 将公钥字符串设置到 pem 格式块中
	block = pem.Block{
		Type:    "RSA Public key",
		Headers: nil,
		Bytes:   pubStream,
	}
	// 4. 通过 pem 将设置好的数据进行编码，写入文件
	pubFile, err := os.Create("public.pem")
	if err != nil {
		panic(err)
	}
	err = pem.Encode(pubFile, &block)
	if err != nil {
		panic(err)
	}
	defer pubFile.Close()
}

// 公钥加密函数
func RSAPublicEncrypto(src []byte, pathName string) []byte {
	// 1. 得到公钥文件中的公钥
	file, err := os.Open(pathName)
	if err != nil {
		panic(err)
	}
	info, err := file.Stat()
	if err != nil {
		panic(err)
	}
	recevBuf := make([]byte, info.Size())
	file.Read(recevBuf)
	// 2. 将得到的字符串解码
	block, _ := pem.Decode(recevBuf)
	// 3. 使用 xx509 将编码之后公钥解析出来
	pubKey, err := x509.ParsePKCS1PublicKey(block.Bytes)
	if err != nil {
		panic(nil)
	}
	// 4. 使用公钥加密
	msg, err := rsa.EncryptPKCS1v15(rand.Reader, pubKey, src)
	if err != nil {
		panic(err)
	}
	return msg
}

// 使用私钥解密
func RSAPrivateDecrypt(src []byte, pathName string) []byte {
	// 1. 打开私钥文件
	file, err := os.Open(pathName)
	if err != nil {
		panic(err)
	}
	// 2. 读私钥文件内容
	info, _ := file.Stat()
	recevBuf := make([]byte, info.Size())
	file.Read(recevBuf)
	// 3. 将得到的字符串解码
	block, _ := pem.Decode(recevBuf)
	// 4. 通过 x509 还原私钥数据
	privateKey, err := x509.ParsePKCS1PrivateKey(block.Bytes)
	if err != nil {
		panic(err)
	}
	msg, err := rsa.DecryptPKCS1v15(rand.Reader, privateKey, src)
	if err != nil {
		panic(err)
	}
	return msg
}
```

测试类：
```go
package src

import (
	"fmt"
	"testing"
)

func TestRsaGenKey(t *testing.T) {
	RsaGenKey(4096)

	src := []byte("我是一个明文")
	fmt.Println("加密前：" + string(src))
	data := RSAPublicEncrypto(src, PublicKeyFile)
	fmt.Println("加密后：" + string(data))
	data = RSAPrivateDecrypt(data, PrivateKeyFile)
	fmt.Println("解密后：" + string(data))
}
```

# 3. 单向散列函数

## 3.1 相关知识

根据任意长度的消息计算出固定长度的散列值的一种数学函数  

1. 能够快速计算出散列值  
2. 消息不同散列值也不同
3. 消息长度不影响散列值的长度

常用的函数有：  
1. MD4
2. MD5
3. SHA-1
4. SHA-256
5. SHA-512

## 3.2 工程实现

核心类
```go
package src

import (
	"crypto/md5"
	"encoding/hex"
)

func GetMD5Str_1(src []byte) string {
	res := md5.Sum(src)
	myres := hex.EncodeToString(res[:])
	return myres
}

func GetMD5Str_2(src []byte) string {
	hash := md5.New()
	// io.WriteString(hash, string(src))
	hash.Write(src)
	res := hash.Sum(nil)
	// 散列值格式化
	myres := hex.EncodeToString(res)
	return myres
}
```

测试类：
```go
package src

import (
	"fmt"
	"testing"
)

func TestGetMD5Str_1(t *testing.T) {
	str := GetMD5Str_1([]byte("我是一个明文"))
	// e8e0915549a588dca746a1db55c75b35
	fmt.Println(str)
}

func TestGetMD5Str_2(t *testing.T) {
	str := GetMD5Str_2([]byte("我是一个明文"))
	// e8e0915549a588dca746a1db55c75b35
	fmt.Println(str)
}
```

# 4. 消息认证码

## 4.1 相关知识

全称：message authentication code，简称 MAC  

它主要是为了解决消息的完整性，即消息在过程中是否被篡改 
算法过程：基于**秘钥**和**消息摘要（使用单向散列函数）**获得的一个值

A 发送给 B 消息，B 解码之后发现是乱码，但是不确定是不是 B 发错了还是说 B 本来发的就是乱码，即消息接收的是否是完整的，即消息没有被篡改，消息被改变，但是依然可以解出数据，但是不知道数据是否可靠

为什么出现消息认证码：  
1. 验证数据是否被篡改
2. 验证数据是否完整

使用场景：
IPSec：IP 协议的增强版

类型：
HMAC：使用 hash 算法实现的消息认证码

消息认证码存在的问题：  
1. 对第三方证明：A 向 B 发送消息，B 想要向 C 证明这条消息的确是 A 发送的，但是消息认证码无法证明：因为有可能这条消息是 B 自己发的，诬陷 A（这是由于秘钥的对称，A 和 B 都有相同的秘钥，导致的）
2. 防止否认：A 就算向 B 发送了消息，但只有 B 知道，B 无法跟别人证明这真是 A 发送的而不是 B 自己捏造的，因为 A 可以不承认，并且说：这是 B 发的，跟我无关。即无法防止发送方否认
3. 无法有效的配送秘钥：因为要保证 A B 都是相同的秘钥

## 4.2 工程实现

核心类：
```go
package src

import (
	"crypto/hmac"
	"crypto/sha256"
)

func GenerateHMAC(src, key []byte) []byte {
	hasher := hmac.New(sha256.New, key)

	mac := hasher.Sum(nil)
	return mac
}

func VerifyHMAC(src, key, mac1 []byte) bool {
	mac2 := GenerateHMAC(src, key)
	return hmac.Equal(mac1, mac2)
}
```

测试类：
```go
package src

import (
	"fmt"
	"testing"
)

func TestGenerateHMAC(t *testing.T) {
	src := []byte("我是一个明文的hash值")
	key := []byte("12345678")
	mac1 := GenerateHMAC(src, key)
	fmt.Printf("mac1: %x\n", mac1)
	isEqual := VerifyHMAC(src, key, mac1)
	fmt.Println(isEqual)
}
```

# 5. 数字签名

## 5.1 相关知识

![][2]

解决消息认证码的问题：
1. 无法有效的配送密码->不需要配送（使用非对称加密）
2. 无法进行第三方证明->只要持有公钥，就可以帮助认证
3. 无法防止发送方否认->私钥只有发送方持有

算法过程，为了方便理解，这里将 A 直接指为明文，实际中 A 应该为加密后的密文：
1. 发送方：根据原文 A 使用摘要算法得到 H1，对 H1 使用私钥进行签名（可以理解为加密）H1 + privateKey = S，将 S 和 A 发送
2. 接收方：接收到 A 后，使用相同的摘要算法得到 H2。接收到 S 后，使用公钥验签（可以理解为解密）S + privateKey = H1。最后对比 H1 和接收到 H2，是否一致，来完成验签

## 5.2 工程实现

核心类：
```go
package src

import (
	"crypto"
	"crypto/rand"
	"crypto/rsa"
	"crypto/sha256"
	"crypto/x509"
	"encoding/pem"
	"io/ioutil"
)

// 私钥签名
func RsaSignData(filename string, src []byte) []byte {
	info, err := ioutil.ReadFile(filename)
	if err != nil {
		panic(err)
	}

	block, _ := pem.Decode(info)
	derText := block.Bytes
	privateKey, err := x509.ParsePKCS1PrivateKey(derText)
	if err != nil {
		panic(nil)
	}
	// 1. 获取原文的 hash 值
	hash := sha256.Sum256(src)
	// 2. 对原文 hash 值使用私钥进行签名
	signature, err := rsa.SignPKCS1v15(rand.Reader, privateKey, crypto.SHA256, hash[:])
	if err != nil {
		panic(err)
	}
	return signature
}

// 公钥认证
func RsaVerifySignature(sign []byte, src []byte, filename string) bool {
	info, err := ioutil.ReadFile(filename)
	if err != nil {
		panic(err)
	}
	block, _ := pem.Decode(info)
	derText := block.Bytes
	publicKey, err := x509.ParsePKCS1PublicKey(derText)
	if err != nil {
		panic(nil)
	}
	// 1. 获取原文的 hash 值
	hash := sha256.Sum256(src)
	// 2. 公钥解密签名
	err = rsa.VerifyPKCS1v15(publicKey, crypto.SHA256, hash[:], sign)
	if err != nil {
		return false
	}
	return true
}
```

测试类：
```go
package src

import (
	"fmt"
	"testing"
)

// 私钥签名
func TestRsaSign(t *testing.T) {
	src := []byte("我是一个明文哦")
	signature := RsaSignData(PrivateKeyFile, src)
	fmt.Println("密文：" + string(signature))
	isVerify := RsaVerifySignature(signature, src, PublicKeyFile)
	if isVerify {
		fmt.Println("验证成功")
	} else {
		fmt.Println("验证失败")
	}
}
```

[1]: https://community.cisco.com/legacyfs/online/legacy/9/6/8/112869-des3p1.gif
[2]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/somephoto/2019-08-27%E6%95%B0%E5%AD%97%E7%AD%BE%E5%90%8D.jpg