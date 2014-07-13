---
layout: post
title: "Node.js Crypto模块 使用示例"
date: 2013-11-14 17:44
comments: true
categories: node.js
keywords: Node.js, crypto, 加密, 解密, 公钥, 私钥
description: 
#published: false
---

* 目录
{:toc}
<!--more-->

Node版本: v0.10.20 (API Stability: 2 - Unstable) [^1]
===============================================================================

公钥私钥的签名和验证
===============================================================================

用 ssh 的公私钥做例子。
`ssh-keygen` 可以生成不同种类和长度的公私钥对。

查看 ssh key 的种类和长度[^3]:

```bash
ssh-keygen -l -f id_rsa.pub  # 假设输出长度是 1024 , 种类是 RSA
```

将 ssh格式 的公钥转换为 RFC4716/PKCS8/PEM 等格式[^4]：

```bash
ssh-keygen -f id_rsa.pub -e -m PEM  # 假设保存到 id_rsa.pub.pem
```


```javascript
// 准备公私钥和数据
var privatekey = fs.readFileSync('.ssh/id_rsa', 'utf8');
var publickey = fs.readFileSync('.ssh/id_rsa.pub.pem', 'utf8');
var data = 'www.youku.com';

// 用私钥签名
var signer = crypto.createSign('RSA-SHA256');
signer.update(data);
var signature = signer.sign(privatekey, 'base64');

// 用公钥验证
var verifier = crypto.createVerify('RSA-SHA256');
verifier.update(data);
var isok = verifier.verify(publickey, signature, 'base64');
console.log(isok); // true
```


加密和解密[^2]
===============================================================================

用 aes256 算法做例子, 输入字符串为utf-8编码, 加密输出base64编码串.


```javascript
// 准备密码和数据
var algo = 'aes256'
var plain_format = 'utf8';
var crypt_format = 'base64';
var key = '123456';
var data = 'secret string';

// 加密
var cipher = crypto.createCipher(algo, key);
var encrypted = cipher.update(data, plain_format, crypt_format);
encrypted += cipher.final(crypt_format);

// 解密
var decipher = crypto.createDecipher(algo, key);
var decrypted = decipher.update(encrypted, crypt_format, plain_format);
decrypted += decipher.final(plain_format);
console.log(decrypted); // 'secret string'
```

_______________________________________________________________________________
[^1]: [Node.js Manual & Documentation](http://nodejs.org/api/crypto.html)
[^2]: [StackOverflow](http://stackoverflow.com/questions/6953286/node-js-encrypting-data-that-needs-to-be-decrypted)
[^3]: [prefetch.net](http://prefetch.net/blog/index.php/2010/12/13/locating-the-ssh-key-type-and-key-size-from-a-public-key-file/)
[^4]: [unix.stackexchange.com](http://unix.stackexchange.com/questions/26924/how-do-i-convert-a-ssh-keygen-public-key-into-a-format-that-openssl-pem-read-bio)