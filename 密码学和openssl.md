# 密码学
## 分类
+ 对称加密（一个钥匙）
+ 非对称加密（两个钥匙）
+ Hash(没有钥匙，不可恢复)

## 对称加密常见算法
+ **DES**
+ 3DES
+ DESX
+ AES
  
## 非对称加密常见算法
+ **RSA**
+ **DSA（数字签名较优）**
+ ECC（移动设备用）
+ Diffie-Hellman
+ EI Gamal

> 以上算法产生public key和private key，public key传输给对端，private key自己保存，public key做加密，private key做解密
> **对称加密得性能优于非对称，非对称加密一般用在首次密钥交换的时候**
> 使用非对称加密还是有仿冒风险（截取到public key然后伪造身份发消息），所以认证也是不可缺少的。

## 基于数学难题分类
+ 大整数分解问题类 -> RSA
+ 离散对数问题类 -> DSA
+ 椭圆曲线类 -> ECC

# Hash算法（生成摘要，防止被篡改）
+ MD2
+ MD4
+ MD5
+ HAVAL
+ SHA
+ SHA-1
+ SHA-256

# openssl
+ 密码学算法的编程实现
+ 使用openssl可以生成key，加解密、签名等

# 数字证书详见参考材料

#  参考
[数字签名及数字证书原理](https://www.bilibili.com/video/BV18N411X7ty/?spm_id_from=333.337.search-card.all.click&vd_source=00c7bb189a105f317a347bc7d83911b5)
[密码学与 OpenSSL](https://www.bilibili.com/video/BV1VC4y1W7ua/?spm_id_from=333.337.search-card.all.click&vd_source=00c7bb189a105f317a347bc7d83911b5)