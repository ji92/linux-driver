![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829223050.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829223108.png)
 ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829223149.png)
 ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829223212.png)
 ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829223225.png)
 ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829223347.png)
 ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829223424.png)
 ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829223447.png)
 ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829223621.png)
 ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829223658.png)
 ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829223925.png)
 ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829224058.png)
 ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829224259.png)
 ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829224317.png)
 ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829224617.png)

### Hybrid接口是华为设备特有，思科的就没有

 ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829224404.png)
 ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829224815.png)

### access接口只有一个vid，trunk接口有多个vid

### access端口收到帧时，如果是无标签的，那就打上自己的PVID；如果是有标签的，那就查看是否与自己的PVID相同，相同则放过，不相同则丢弃。（简而言之，就是access端口放行后的帧就是打上自己PVID标签的帧）。

### access端口发送帧时，去掉标签，发送给主机。

### trunk端口收到帧时，（在允许通过的情况下）如果有标签，那就放行；如果没标签，打上默认的VID（也就是PVID）。

### trunk端口发送帧时，（肯定有标签）如果标签与PVID不一样，那就直接发送（透传），如果标签与PVID一样，那就去掉标签。（简而言之，trunk链路上传送的帧要么有标签，要么没有标签，有标签的一定不是默认VID，没有标签的一定是默认VID）。

### 万一trunk之间接了不支持vlan的设备

### vlan修建：华为设备trunk默认全部不放行，思科设备trunk默认全部放行，都需要配置vlan list

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829230359.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829230413.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829230429.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829230642.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829230519.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829230656.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829230714.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829230725.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829230736.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829230745.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829230808.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829230824.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829230839.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829230852.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829231049.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829231115.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829231219.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829231242.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829231253.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829231302.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829231315.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829231325.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829231334.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829231405.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230829231414.png)

# 参考资料

[https://zhuanlan.zhihu.com/p/384845768](https://zhuanlan.zhihu.com/p/384845768)
[https://www.bilibili.com/video/BV1ha4y177nH/?spm_id_from=333.337.search-card.all.click&vd_source=00c7bb189a105f317a347bc7d83911b5](https://www.bilibili.com/video/BV1ha4y177nH/?spm_id_from=333.337.search-card.all.click&vd_source=00c7bb189a105f317a347bc7d83911b5)
