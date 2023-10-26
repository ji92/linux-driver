# IPsec Overview

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904191845.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904192035.png)
+ IPsec是个框架，具体的协议由IKE进行协商

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904192232.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904192243.png)
+ 无限制的网络

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904192402.png)
+ 主要用ESP和IKE

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904192450.png)
+ 使用IKE进行协商完之后的结果，就是安全联盟SA

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904192506.png)
+ AH-ESP和AH基本不会用

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904193338.png)
+ 外面的是global ip，里面的是private ip,路由不可达

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904193634.png)
+ 或者：通讯点到通讯点全局可路由，就是传输模式；否则就是隧道模式

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904194032.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904194156.png)
+ 手动配置流程，可忽略，重点了解IKE协商流程

# IPsec 详解

## 简介

Internet Protocol Security (IPsec)是一个安全网络协议套件，它可以完成认证加密数据包，从而为IP层网络设备提供安全加密通信。IPSec还被用到了VPN中。

IPsec包括了用于在客户端之间开始会话前建立相互认证和会话所用秘钥分发的协议簇。IPsec可以保护数据流在端到端，网络到网络，端到网络之间。IPsec使用密码安全服务来保护IP网络中的通信。它支持网络层对等身份认证，数据源认证，数据加密和重放保护。

## IPsec的应用

+ 加密应用层数据
+ 在公共互联网上为路由器发送数据提供安全保障
+ 在不加密的情况下进行身份认证，比如验证来自一个已知发送者的数据报
+ 通过使用 IPsec 隧道设置电路来保护网络数据，在这些电路中，所有数据在两个端点之间发送时都被加密，就像使用虚拟专用网络(VPN)连接一样

## IPsec优势

+ 在路由器和防火墙中使用IPsec，对通过其边界的所有通信流提供强安全性，内部通信无安全开销
+ 位于传输层之下，对所有应用透明
+ 可以对终端用户透明，不需要对用户进行安全机制培训

### 安全架构

IPsec是一个开放的组件，是IPv4套件的一部分。IPsec使用了下面的这些协议实现各种各样的功能。

+ `Authentication Headers (AH)`提供无连接数据完整性及为IP数据报提供数据源认证，同时为重放攻击提供保护（这里防止重放是用序列号，其他情况下可能使用时间戳的形式）。
+ `Encapsulating Security Payloads (ESP)` (封装安全有效负载)提供加密，无连接数据完整性，数据源认证，抗重放共计，和有限的流量保密性。
+ `Security Associations (SA)` 提供算法和数据包，这些算法和数据包提供AH和ESP所需的参数。互联网安全协会和秘钥管理协议(ISAKMP)提供了用于身份验证和秘钥交换的框架，带有通过手动配置的预共享密钥，Internet密钥交换（IKE和IKEv2），Kerberized Internet密钥协商（KINK）或IPSECKEY DNS记录提供的实际经过身份验证的密钥材料。

## IPsec在IP层提供的服务
+ 访问控制
+ 无连接的完整性
+ 数据来源验证
+ 防重放攻击
+ 加密和数据流分类加密

## IPsec的两种模式
+ 传送模式
  + 当协议在一台主机上实现时使用，IPsec头部被插在IP头后面。主要为上层协议提供保护，加密和可选择的认证，用于在两个主机之间进行端到端通信。
  
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904195816.png)

+ 隧道模式
  + 整个包被封装加密，形成新的IP头部。沿途路由器不检查内部的IP报头。
  + 适用于隧道结束在某个非目的地(安全网关，防火墙)该目的地结束IPsec隧道，还原成原来数据包在本地网（局域网）传输（局域网不用理解IPsec）
  + 隧道模式的另一个好处是可以将聚集TCP连接当成一个流加密。可以抵抗traffic analysis
  + 加密发生在外部主机与安全网关之间或者发生在两个安全网关之间。
  
  ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904200003.png)

## IPsec中的安全组合(SA)
+ 本质上位一个安全通信信道提供的一套安全参数，是单向的，需要双向则两个关联
+ SA是通过密钥管理协议在通信双方之间进行协商，协商完毕之后，双方都在它们的安全关联数据库中存储该SA参数

`安全关联`
通信双方对某些要素之间的协定，包括：
加密算法和模式 加密秘钥 加密参数（如初始化向量），鉴别协议和秘钥，关联的生命期（长时间会话时，尽可能频繁的选择新的加密密钥），关联对应端地址，受保护数据敏感级别。


一个安全关联由三个参数唯一确定
+ 安全参数索引
  + 指向一张安全关联表指针，由AH和ESP报头携带，使得接受系统选择合适的SA
+ IP目的地址
  + 目前仅允许单播地址，是SA的目的端点地址
+ 安全协议标识
  + 表明是AH还是ESP

## IPsec通信协议
AH(Authentication Header)
+ 无连接数据完整性：通过哈希函数产生的校验来保证
+ 数据源认证：通过在计算验证码时加入一个共享秘钥来实现
+ 抗重放服务：AH头部中的序列号可以防止重放攻击

AH的协议号是51，AH头比ESP头简单很多，因为它不提供机密性，不会加密所保护的数据报。不论是在传输模式还是在隧道模式下，AH提供对数据报的保护时，它保护的是整个IP数据包（易变的字段除外，如IP头中的TTL和TOS字段）
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904201200.png)

ESP协议
+ 无连接数据完整性
+ 数据源认证
+ 抗重放攻击
+ 数据保密：密码算法加密IP数据包
+ 有限的数据流保护：由隧道模式下的保密服务提供
ESP通常使用DES、3DES、AES等加密算法实现数据加密，使用MD5或SHA1来实现数据完整性认证。

ESP同样被当做一种IP协议对待，紧贴在ESP头前的IP头，协议号为50.
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904201249.png)

+ 在隧道模式中，ESP保护整个IP包，整个IP包将会以ESP载荷的方式加入新建的数据包，同时，系统根据隧道起点和终点等参数，建立一个隧道IP头，作为这个数据包的新IP头，ESP头夹在隧道IP头和原始IP头之间，并点缀ESP尾。
+ ESP提供加密服务，所以原始IP包和ESP尾以密文的形式出现。
+ ESP在验证过程中，只对ESP头部、原始数据包IP包头、原始数据包数据进行验证；只对原始的整个数据包进行加密，而不验证数据。

**AH和ESP的对比**
+ ESP在隧道模式不验证外部IP头，因此ESP在隧道模式下可以在NAT环境中运行
+ ESP在传输模式下会验证外部IP头部，将导致校验失败
+ AH因为提供数据来源确认（源IP地址一旦改变，AH校验失败），所以无法穿越NAT
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904201621.png)

+ ESP认证数据：是变长字段，只有选择了验证服务时才需要有该字段。
+ 很多情况下，AH的功能已经能满足安全的需要，ESP由于需要使用高强度的加密算法，需要消耗更多的计算机运算资源，使用上受到一定限制。

## IPsec建立过程
**IKE协商**
+ IPsec的密钥管理：密钥的确定和分发
  + 典型的两个应用之间的通信需要4个密钥：用于完整性和机密性的发送对和接受对。
+ IPsec支持两种方式的SA建立和秘钥管理
  + 手工方式
  + 自动方式（IKE协商）
    + SA可以通过协商方式产生
    + SA过期以后重新协商，提高了安全性
    + 适用于较为复杂的拓扑和较高安全性的网络
  
**IKE第一阶段**

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904201941.png)

+ 用途
  + 协商创建一个通信信道(IKE SA)
  + 对该信道进行认证
  + 为双方进一步的IKE通信提供机密性、数据完整性以及数据源认证服务
+ 步骤
  + 策略协商
  + DH交换
  + 认证

1. 安全联盟SA(Security Association)：是两个IPSec通信实体之间经协商建立起来的一种共同协定，它规定了通信双方使用哪种IPSec协议保护数据安全、应用的算法标识、加密和验证的密钥取值以及密钥的生存周期等等安全属性值。通过使用安全关联(SA) ， IPSec能够区分对不同的数据流提供的安全服务。
2. IPSec是在两个端点之间提供安全通信，端点被称为IPSec对等体。IPSec能够允许系统、网络的用户或管理员控制对等体间安全服务的粒度。通过SA（Security Association），IPSec能够对不同的数据流提供不同级别的安全保护。
3. 安全联盟是IPSec的基础，也是IPSec的本质。SA是通信对等体间对某些要素的约定，例如，使用哪种安全协议、协议的操作模式（传输模式和隧道模式）、加密算法（DES和3DES）、特定流中保护数据的共享密钥以及密钥的生存周期等。
4. 安全联盟是单向的，在两个对等体之间的双向通信，最少需要两个安全联盟来分别对两个方向的数据流进行安全保护。入站数据流和出站数据流分别由入站SA和出站SA进行处理。同时，如果希望同时使用AH和ESP来保护对等体间的数据流，则分别需要两个SA，一个用于AH，另一个用于ESP。
5. 安全联盟由一个三元组来唯一标识，这个三元组包括安全参数索引（SPI, Security Parameter Index）、目的IP地址、安全协议号（AH 或ESP）。SPI 是为唯一标识SA而生成的一个32比特的数值，它在IPSec头中传输。
6. IPSec设备会把SA的相关参数放入**SPD（Security Policy Database）**里面，SPD里面存放着“什么数据应该进行怎样的处理”这样的消息，在IPSec数据包出站和入站的时候会首先从SPD数据库中查找相关信息并做下一步处理。

**IKE和AH/ESP之前的联系**
IKE是UDP之上的一个应用层协议，是IPsec的信令协议。IKE为IPsec协商产生秘钥，供AH/ESP加解密和验证使用。AH协议和ESP协议有自己的协议号，分别为51和50。

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904202337.png)

**数据传输阶段**
数据传输阶段是通过AH或者ESP通信协议进行数据的传输。
数据传输建立在网络层。

**VPN隧道黑洞**
可能情况：
对端的VPN连接已经断开而我方还处在SA的有效生存期时间内，从而形成了VPN隧道的黑洞。
另外一端如果之前SA没有释放，异常重启的对端又来连接，是不会接受新的连接协商的。

DPD解决VPN隧道黑洞：

DPD：死亡对等体检测（Dead Peer Detection），检查对端的ISAKMP SA是否存在。当VPN隧道异常的时候，能检测到并重新发起协商，来维持VPN隧道。
DPD 只对第一阶段生效，如果第一阶段本身已经超时断开，则不会再发DPD包。
DPD包并不是连续发送，而是采用空闲计时器机制。每接收到一个IPSec加密的包后就重置这个包对应IKE SA的空闲定时器；
如果空闲定时器计时开始到计时结束过程都没有接收到该SA对应的加密包，那么下一次有IP包要被这个SA加密发送或接收到加密包之前就需要使用DPD来检测对方是否存活。
DPD检测主要靠超时计时器，超时计时器用于判断是否再次发起请求，默认是发出5次请求（请求->超时->请求->超时->请求->超时）都没有收到任何DPD应答就会删除SA。

检查对端的ISAKMP SA是否存在两种工作模式：

1. 周期模式：每隔一段时间，向对端发送DPD包探测对等体是否仍存在，如果收到回复则证明正常。如果收不到回复，则会每隔2秒发送一次DPD，如果发送七次仍收不到回复，则自动清除本地对应的ISAKMP SA和IPSEC SA。
2. 按需模式：这是默认模式，当通过IPSEC VPN发送出流量而又收不到回程的数据时，则发出DPD探测包，每隔2秒发送一次，七次都收不到回应则清除本地对应的ISAKMP SA和IPSEC SA。注意，如果IPSEC通道上如果跑的只有单向的UDP流量，则慎用这个模式，尽管这种情况极少。
DPD很实用，应该开启。至于选择哪个模式，则根据实际需要，周期模式可以相对快地找出问题peer，但较消耗带宽；按需模式，较节约带宽，但只有当发出加密包后收不到解密包才会去探测。

# 参考资料：
+ [https://www.youtube.com/watch?v=oGIYzguqoJM&amp;t=12s](https://www.youtube.com/watch?v=oGIYzguqoJM&t=12s)
+ [网络安全之IPsec详解](https://blog.csdn.net/qq_43561370/article/details/111814842)
+ [IPSec介绍](https://blog.csdn.net/NEUChords/article/details/92968314)
+ [NAT-T：IPsec穿越NAT之道](https://blog.csdn.net/u014023993/article/details/86634339)
+ [GFW原理和Shadowsocks/ShadowsocksR/V2ray/Trojan又是如何突破封锁的？](https://www.youtube.com/watch?v=k80cu16M-rw)
+ [IPsec Wikipedia](https://en.wikipedia.org/wiki/IPsec#Authentication_Header)
