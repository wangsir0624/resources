# 短链接生成服务

## 项目简介

### 项目背景
短链接，通俗来说，就是将长的URL网址，通过程序计算等方式，转换为简短的网址字符串。开发此项目的初衷是方便短信营销，通过压缩链接的URL长度，避免URL太长导致短信必须分多条进行发送。现在很多公司其实都有提供短信压缩的功能，但是如果这些服务出现一些问题（例如更改域名，服务器宕机等），可能会影响到公司内部业务，为了提高项目的可控性，项目组决定自行研发一套短链接生成服务系统。

### 项目特点
- nginx方向代理 + go后端，只需要改一下配置文件，就可以部署一台服务。集群搭建方便，可以根据不同的业务，使用不同的服务器，通过分流来减轻服务器压力。
- 区分临时短链和永久短链，采用不同的存储层存储，临时短链采用redis存储，永久短链采用mysql存储。防止数据量暴涨，降低吞吐率。
- 错误告警和快速恢复

### 接口列表
```
接口1：POST /api/shortlinks      生成短链接
接口2：GET /{shortlink}          跳转到原始链接
```

### 原理与关键点
![短链接生成服务原理图](https://github.com/wangsir0624/resources/blob/master/images/shortlinks_share/yuanli.jpg)

思考两个问题：
1、是否可以将原始链接作为明文进行加密
2、是否可以使用哈希算法进行加密

#### 如何获取唯一标示符
- mysql中的获取
- redis中的获取

#### 加密算法
![加密算法原理图](https://github.com/wangsir0624/resources/blob/master/images/shortlinks_share/jiami.png)

#### 压力测试
- 测试人员：星辰&隆杰
- 测试服务器：101.37.254.197
- 架构方式：golang+redis，nginx反代
- 测试数据：20W

|    并发量     |  吞吐率   |   平均响应时间（ms）   |
| :---------: | :---: | :----: |
| 500 | 282.90 | 3.535 | 
| 1000 | 275.11 | 3.635 |
| 2000 | 270.40 | 3.698 |
| 3000 | 270.63 | 3.695 |

## 接入指南
- 接口权限验证：接口采用appkey+appsecret的方式进行权限验证，调用接口的时候，需要传递一个timestamp时间戳参数和一个signture签名参数，签名算法md5(appkey+appsecret+timestamp)，这样可以防止重放攻击，保护接口安全。因此，在接入的时候，需要先申请一对app key和app secret
- SDK：[点击跳转](http://git.linghit.com:666/datacenter/datacenter.linghit.com.backend/tree/master/src/Services/ShortLinks)

## 问题与解决
问题：之前redis被清理过一次，导致临时短链全都失效，无法访问
解决：1、单独开一个线程，定时检测redis的运行情况，如果redis再次出现数据丢失的情况，给相关负责人发送邮件提醒。
2、添加快速恢复数据的功能。