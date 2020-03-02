
>加密及签名参考-[Base64，Hash，加密，签名知识点梳理](https://github.com/MadnessXiong/AndroidNote/blob/master/Network/Base64，Hash，加密，签名知识点梳理.md)

### 1. Https

   - 定义：Https是基于Http标准并通过SSL层对TCP传输数据进行加密，所以Https是对Http+SSL/TCP的简称。

### 2. SSL/TLS

  - 定义:SSL是指安全套接字层，是基于HTTP之下TCP之上的一个协议层，可确保互联网连接安全，保护两个系统之间发送的任何敏感数据。TLS是更为安全的SSL。

### 3. Https加密过程：

   - 客户端通过发送client hello报文开始SSL通信（建立在三次握手已完成的基础上）。报文中包含客户端支持的SSL指定版本，加密组件（所使用的加密算法，如对称加密算法，非对称加密算法，Hash算法等及密钥长度等）
  
   - 服务器以server hello报文作为应答。在报文中包含SSL版本及加密组件。服务器的加密组件内容是从接收到的客户端加密组件中选出来的。

   - 服务器发送certificate报文，报文中包含公开密钥证书（公钥）

   - 服务器发送server hello done报文通知客户端，最初阶段的协商部分结束

   - 客户端使用根证书验证服务器证书的合法性，然后客户端以client key exchange报文作为回应。报文中包含通信加密中一种被称为Pre-master secret的随机密码串。
