<!-- MarkdownTOC -->

- [RSA加密](#rsa加密)
- [AES加密](#aes加密)
- [APP接口设计\(加密/签名/TOKEN认证/接口版本\)](#app接口设计加密签名token认证接口版本)
- [参考文献](#参考文献)

<!-- /MarkdownTOC -->

<a id="rsa加密"></a>
# RSA加密
-  加密历史
    + 对称加密算法
        * 甲方选择某一种加密规则，对信息进行加密
        * 乙方使用同一规则，对信息进行解密
    + 非对称加密算法(Diffie-Hellman密钥交换算法、RSA算法)
        * 乙方生成两把密钥(公钥和私钥)。公钥是公开的，私钥是保密的。
        * 甲方获取乙方的公钥，然后用它对信息加密
        * 乙方得到加密后的信息，用私钥解密
    + RSA算法
        * 最广为使用的非对称加密算法
        * 非常可靠，密钥越长，越难破解
        * 目前被破解的最长RSA密钥是768个二进制位。1024位的RSA密钥基本安全，2048位的密钥及其安全
- 互质关系
    + 两个整数，除了1以为，没有其他公因子，我们就称这两个数是互质关系coprime。不是质数也可以构成互质关系
        * 任何两个质数构成互质关系
        * 一个数是质数，另一个数只要不是前者的倍数，两者就构成互质关系
        * 两个数中较大的那个数是质数，则两者构成互质关系
        * 1和任何自然数构成互质关系(包括自身)
        * 任何大于1的整数，与比其小1的整数构成互质关系
        * 任何大于1的奇数，与比其小2的整数构成互质关系
- 欧拉函数
    + 任何给定正整数n，在小于等于n的正整数之中，计算有多少与n构成互质关系的正整数
    + 如果n = 1，则 φ(1) = 1
    + 如果n是质数，则 φ(n) = n-1
    + 如果n是某质数p的若干次方，则φ(n) = p^k - p^(k-1)
    + 如果n可以分解成两个互质的整数之积，则φ(n) = φ(p1p2) = φ(p1)φ(p2)
- 欧拉定理
    + a ^ φ(n) = 1(mod n)
        * a和n是两个互质的正整数，a的φ(n)次方被n除的余数是1
    + 费马小定理
        * n取质数p
        * a ^ (p-1) = 1(mod p) 
- 模反元素
    + ab = 1(mod n)
    + 如果两个正整数a和n互质，那么一定能找到整数b，是的ab-1被n整除，b就是a的模反元素
    + 如果b是a的模反元素，则b+kn(k为任意整数)都是a的模反元素
    + a*a^φ(n-1) = 1(mod n)  所有a^φ(n-1)就是a的一个模反元素
- 密钥生成步骤
    + 随机去两个不相等的质数p和q
    + 计算p和q的乘积n
        * n的长度就是密钥长度，n写成二进制一般是1024位，重要场合是2048位
    + 计算n的欧拉函数φ(n) = (p - 1)(q -1)
    + 随机选择一个整数e，条件是1 < e < φ(n),且e与φ(n)互质
        * 常常选择65537(2^16+1)
    + 计算e对φ(n)的模反元素d
    + 将n和e封装成公钥(乘积，随机整数)，n和d封装成私钥(乘积，模反元素)
- RSA算法的可靠性
    + 共出现六个数字：p、q、n、φ(n)、e、d
    + 有无可能在已知公钥(n、e)的情况下推到出d
        * d需要e和φ(n)
        * φ(n)需要p和q
        * n = pq，需要将n因式分解
        * 所以要破解私钥就是要知道n是哪两个质数的乘积
- 加密和解密
    + m^e = c(mod n)
        * m是发送要加密的信息，m必须是整数，字符串可用ascii值或unicode值
        * c是客户端发送给服务端用来解密的，需要求出
    + c^d = m(mod n)
        * 通过私钥(n,d)和客户端发来的c，解出m
        * 公钥只能加密小于n的m，想加密大于n的整数有两种办法
            - 长信息分割若干短信息，每段分别加密
            - 使用对称性加密算法(如DES,data encryption standard|AED,advanced encryption standard)加密传输的报文，再用RSA公钥将DES密钥加密
        
<a id="aes加密"></a>
# AES加密
- 加密
    + base64_encode(mcrypt_encrypt($cipher, $key,$data, $mode,$iv));
- 解密
    + mcrypt_decrypt($cipher, $key, base64_decode($data), $mode, $iv);

<a id="app接口设计加密签名token认证接口版本"></a>
# APP接口设计(加密/签名/TOKEN认证/接口版本)
+ 接口安全问题：
    * 请求来源是否合法
    * 请求参数是否被篡改
    * 请求是唯一的(即同样的请求，只有一次会生效)
    * 数据传输的安全性
+ 客户端请求流程
    * 请求服务端，获取RSA公钥(rsaPublicKey)以及接口密钥(key)
    * 生成AES密钥(aesKey)
    * 定义Params参数[TOKEN,PLATFORM,VERSION,MONCESTR,TIMESTAMP]加密，得到加密后的数据(encryptParams)
        - Token用户登陆凭证
        - Platform请求来源所属平台
        - Version平台版本
        - NonceStr随机字符串，用于验证请求是否重复
        - Timestamp请求时间，用于验证请求是否超时
    * 使用RSA公钥(rsaPublicKey) 对 AES密钥(aesKey)加密，得到加密后的AES密钥(encryptAesKey)
    * 将加密后的AES密钥作为HEADER参数之一，完整的HEADER参数：[KEY => encryptAesKey, VALUE => encryptParams]
    * 提取请求数据相关参数，拼接 接口密钥(key)生成签名，然后发送请求给服务端，等待响应
    * 收到服务端的响应后，使用请求时发送的AES密钥(aesKey)进行解密
+ 服务器处理流程
    * 响应客户端的请求，获取客户端传输过来的HEADER参数[KEY,VALUE]
    * 使用 RSA私钥(rsaPrivateKey)对加密后的 AES密钥(key)进行RSA解密，得到AES密钥(decryptKet)
    * 使用 解密后的AES密钥(decryptKey) 对 加密后的数据(VALUE) 进行AES解密, 得到解密后的JSON数据(decryptValue)
    * 对 解密后的JSON数据(decryptValue) 进行JSON解析
    * 若 解密后的AES密钥(decryptKey) 与 解密后的JSON数据(decryptValue) 不为空, 说明请求合法
    * 获取 请求时间(decryptKey[TIMESTAMP]) 与 服务端时间(serverTime), 判断时间差是否大于 请求有效时间(requestExpireTime), 若大于则说明请求已超时(无效)
    * 获取 随机字符串(decryptKey[NONCESTR]), 校验此记录是否存在 服务端的NONCESTR集合 内, 若存在则说明是重复请求(无效), 否则记录此NONCESTR, 并删除集合内超过 请求有效时间(requestExpireTime) 的NONCESTR (可以使用redis的expire, 新增nonce的同时设置它的超时失效时间)
    * 若当前请求需要验证用户是否登录, 则 用户登录凭证(decryptValue[TOKEN]) 不能为空, 并查询数据库, 判断 TOKEN 是否过期
    * 获取 请求数据(requestData), 使用 接口密钥(secretKey) 对请求数据验签
    * 返回数据时, 转为JSON格式后, 使用 解密后的AES密钥(decryptKey) 进行加密, 然后返回给客户端


<a id="参考文献"></a>
# 参考文献
[阮一峰-RSA算法原理](http://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html)

