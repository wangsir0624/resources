# Kong使用文档

## Kong简介
对于一些传统的大型项目，传统的方式会有一些缺陷，比如说新人熟悉系统成本高（因为整个系统作为一个整体，彼此会有一定的牵连），项目重启时间长，重构困难（对于一个新技术的引入，可能需要对整个项目推到重来），不易于更换新的技术，并且整个项目会慢慢变成巨无霸。所以说就会有微服务这种概念，一个服务实现一个不同的特性或者功能。每一个独立的微服务都是一个小型应用。一些微服务可能会暴露一些api给其他的一些微服务或者是客户。如下图：
![微服务弊端](https://github.com/wangsir0624/resources/blob/master/study/api-gateway/images/weifuwubiduan.png)

随着服务增多，服务地址的管理将变得非常繁重，服务每次修改一次地址，很多个业务线代码可能要随之更改。为了克服这些缺陷，引入了api网关的。api网关作为所有API的公共入口，这有点类似于设计模式里面的Facade模式，这样能大大降低api的使用难度以及维护成本。同时还可以在app网关层做权限控制，安全，负载均衡，请求分发，监控等等。原理图如下：
![api网关原理图](https://github.com/wangsir0624/resources/blob/master/study/api-gateway/images/api_gateway.png)

Kong是由Mashape公司开发的一款基于Lua的Nginx api网关插件，具有如下几个优点：

- 伸缩性好，集群部署方便，只要将Kong节点配置成相同的数据库，就能构成一个Kong集群
- 扩展性好，有丰富的插件，提供各式各样的功能，包括授权，权限控制，流量控制，日志等等
- 移植性好，可以在各种平台上运行

## 安装
Kong支持多种操作系统平台，包括各种Linux系统、Mac等。本文采用的安装环境为Linux Ubuntu，[查看更多](https://konghq.com/install/)

### Kong的安装
以deb安装为例
```bash
#下载deb安装包
sudo wget https://bintray.com/kong/kong-community-edition-deb/download_file?file_path=dists/kong-community-edition-0.11.2.trusty.all.deb

#安装
sudo apt-get update
sudo apt-get install openssl libpcre3 procps perl
sudo dpkg -i *kong-community-edition-0.11.2.*.deb
```

### PostgreSQL安装
Kong支持两种数据库来充当存储层，分别为：[Cassandra](http://cassandra.apache.org/)和[PostgreSQL](https://www.postgresql.org/)，本文以PostgreSQL为例。
```bash
sudo apt-get install postgresql-9.6 
```

### 数据库初始化
PostgreSQL安装完成后，默认会创建一个postgre用户，认证方式为peer，意思就是必须将当前用户切换成postgre用户，才可以使用psql命令连接数据库
>切换账号
```bash
sudo su postgre
```

>连接数据库
```bash
psql
```

>创建kong用户和数据库
```psql
CREATE USER kong; CREATE DATABASE kong OWNER kong;
```

>将Kong认证方式修改为md5

打开/etc/postgresql/9.6/main/pg_hba.conf，修改两行配置
```vim
local all postgres peer
local all all peer
```
修改为
```vim
local all postgres md5
local all all md5
```

重启postgresql
```bash
sudo service postgresql restart
```


>修改kong配置文件
>将kong.conf修改为如下：
```vim
pg_host = 127.0.0.1             # The PostgreSQL host to connect to.
pg_port = 5432                  # The port to connect to.
pg_user = kong                  # The username to authenticate if required.
pg_password = kong                   # The password to authenticate if required.
pg_database = kong              # The database name to connect to.

pg_ssl = off  
```

>执行数据库迁移命令
```bash
sudo kong migrations up -c /etc/kong/kong.conf
```

### 启动
```bash
sudo kong start -c /etc/kong/kong.conf
```

### 测试
kong启动后默认会监听两个端口，8000api转发端口和8001管理端口，我们通过请求kong的admin api来测试kong是否成功安装并启动
```bash
curl -i http://localhost:8001/
```
如果接口返回200，说明安装成功

## API管理

### 相关管理接口

- [获取API列表](https://getkong.org/docs/0.11.x/admin-api/#list-apis)
- [增加API](https://getkong.org/docs/0.11.x/admin-api/#add-api)
- [查看API](https://getkong.org/docs/0.11.x/admin-api/#retrieve-api)
- [更新API](https://getkong.org/docs/0.11.x/admin-api/#update-api)
- [增加或修改API](https://getkong.org/docs/0.11.x/admin-api/#update-or-create-api)
- [删除API](https://getkong.org/docs/0.11.x/admin-api/#delete-api)

>由于篇幅关系，这儿就不对这些API进行全部介绍，大家可以参看官网的文档

### 增加接口
以观音灵签项目获取灵签解签API为例，接口地址为：http://112.124.40.205:9706/v1/lingqians/{lingqian_id}
```bash
curl -i -X POST --url http://localhost:8001/apis/ --data 'name=lingqian.detail' --data 'hosts=lingqian' --data 'upstream_url=http://112.124.40.205:9706' --data 'methods=GET,HEAD'
```
如果返回如下，则说明添加成功
```
HTTP/1.1 201 Created
Date: Tue, 09 Jan 2018 09:24:40 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.11.2

{
    "created_at":1515489880332,
    "strip_uri":true,
    "id":"307a0db6-dc4b-4d4d-8272-836c1dddbe4f",
    "hosts":["lingqian"],
    "name":"lingqian.detail",
    "http_if_terminated":false,
    "preserve_host":false,
    "upstream_url":"http://112.124.40.205:9706",
    "methods":["GET","HEAD"],
    "upstream_send_timeout":60000,
    "upstream_connect_timeout":60000,
    "upstream_read_timeout":60000,
    "retries":5,
    "https_only":false
}
```

>测试
```bash
 curl -i -X GET --url 'http://localhost:8000/v1/lingqians/guanyin1?mmc_code_tag=1.0.0&mmc_channel=QQ&mmc_devicesn=868724413019320&mmc_appid=5002&mmc_package=oms.mmc.fortunetelling.measuringtools.naming&mmc_operate_tag=5.8.2&mmc_lang=zh&mmc_platform=Android&user_id=1' --header 'Host: lingqian'
```
接口返回如下：
```
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
Server: openresty
Date: Tue, 09 Jan 2018 09:29:45 GMT
Vary: Accept-Encoding
X-Powered-By: PHP/7.0.6
X-Kong-Upstream-Latency: 453
X-Kong-Proxy-Latency: 0
Via: kong/0.11.2

{
  "id":"guanyin1",
  "title":"第一簽",
  "grade":"上籤",
  "zodiac":"子宮",
  "shiyue":["開天闢地作良緣，","吉日良時萬物全；","若得此簽非小可，","人行忠正帝王宣。"],
  "shiyi":["此卦盤古初開天地之象。","諸事皆吉也。"],
  "jieyue":["急速兆速。","年未值時。","觀音降筆。","先報君知。"],
  "xianji":["家宅&rarr;祈福","自身&rarr;秋冬大利","求財&rarr;秋冬大利","交易&rarr;成","婚姻&rarr;成","六甲&rarr;生男","行人&rarr;至","田蠶&rarr;好","六畜&rarr;好","尋人&rarr;見","公訟&rarr;吉","移徙&rarr;東南","失物&rarr;東北","疾病&rarr;歿送","山墳&rarr;吉"],
  "jieyi":"這是個開天闢地的好機緣，這是個好時辰，各種條件都具全，求到本簽非同小可，人忠心正直上級召見，由於本身條件優越，且又逢時機良好，因此獲得眾人擁戴或上級賞賜，而得以成就功名。",
  "jingsui":"受擁而發。",
  "zuoshi":"本簽算是個天王簽，能在自己所謀求的領域中稱王。若問事業者，最後能發展成事業之佼佼者。投資若問政治地位，最後是各級部會首長。或是帝王。簡言之，會在各個領域中成為首腦者。",
  "jianyi":"祈福台恭請神明，便可累積無比殊勝的功德，凡事一帆風順。",
  "jianyi_goto":"qifutai"
}
```

### 接口转发规则
增加API的接口包含三个参数：hosts、uris、methods，这三个参数是用来控制转发规则的，在调用增加API接口时，至少要传递其中一个参数。这三个参数都是可以配置多个值的，不同值之间以逗号分割。
考虑如下的API配置：
```json
{
    "name": "my-api",
    "upstream_url": "http://my-api.com",
    "hosts": ["example.com", "service.com"],
    "uris": ["/foo", "/bar"],
    "methods": ["GET"]
}
```

下面列出一些符合规则的请求
```http
GET /foo HTTP/1.1
Host: example.com
```

```http
GET /bar HTTP/1.1
Host: service.com
```

```http
GET /foo/hello/world HTTP/1.1
Host: example.com
```
>注意：第三个API也是符合规则的，因为uri进行匹配的时候，是按照前缀规则进行匹配

下面再列出一些不符合规则的请求
```http
GET / HTTP/1.1
Host: example.com
```

```http
POST /foo HTTP/1.1
Host: example.com
```

```http
GET /foo HTTP/1.1
Host: foo.com
```

>由于篇幅限制，不能覆盖到全部知识点，想了解全部的同学[戳这里](https://getkong.org/docs/0.11.x/proxy/)

## Consumer管理
消费者，顾名思义，就是API的使用者，这个模块主要是用来做一些API授权用的，后面讲到的权限相关插件的权限分配主体都是这儿讲到的Consumer。消费者管理类似于API管理，也是通过调用Kong的admin api进行管理。

### API列表
- [获取Consumer列表](https://getkong.org/docs/0.11.x/admin-api/#list-consumers)
- [创建Consumer](https://getkong.org/docs/0.11.x/admin-api/#create-consumer)
- [查看Consumer](https://getkong.org/docs/0.11.x/admin-api/#retrieve-consumer)
- [更新Consumer](https://getkong.org/docs/0.11.x/admin-api/#update-consumer)
- [增加或修改Consumer](https://getkong.org/docs/0.11.x/admin-api/#update-or-create-consumer)
- [删除Consumer](https://getkong.org/docs/0.11.x/admin-api/#delete-consumer)

>想了解更多的同学，可以参看[官方文档](https://getkong.org/docs/0.11.x/getting-started/adding-consumers/)

## 进阶

### 后端接口负载均衡

当后端接口访问量大的时候，一台服务器可能服务不过来，这时候我们就要对后端接口做一下负载均衡。目前Kong常用的负载均衡方法主要有DNS轮询、加权轮询等

#### DNS轮询
DNS轮询是通过给一个域名添加多条A解析记录来实现负载均衡的。配置的时候非常简单，例如我们有三台后端服务器，IP分别为1.1.1.1、2.2.2.2、3.3.3.3，域名为example.com。在添加API的时候，stream_url参数设置为example.com，然后给域名example.com添加3条A解析记录，分别指向1.1.1.1、2.2.2.2以及3.3.3.3，这样配置就算完成了。虽然配置简单，但是弊端也非常明显，DNS轮询无法给后端服务器分配权重，而且无法对后端服务器进行健康检查，如果有一台服务器宕机了，DNS依然会解析到这台服务器。<br />
添加A记录的时候，有一个很重要的配置参数TTL，这个参数是用来控制域名解析结果刷新频率的，TTL越大，DNS缓存时间越长，更新越慢，TTL越小，更新越快，如果TTL为0，表示每次请求都会刷新DNS解析结果，但是会延长请求响应时间，在实际配置中，必须综合考虑多方面因素，设置合适的值。

#### 加权轮询
类似与Nginx的负载均衡，Kong也是通过配置upstream和target来实现负载均衡的。下面示范upstream和target的配置过程：
```bash
# create an upstream
curl -i -X POST http://localhost:8001/upstreams --data "name=address.v1.service"

# add two targets to the upstream
curl -i -X POST http://localhost:8001/upstreams/address.v1.service/targets --data "target=192.168.34.15:80" --data "weight=100"
curl -i -X POST http://localhost:8001/upstreams/address.v1.service/targets --data "target=192.168.34.16:80" --data "weight=50"
```

执行上面的语句，我们就创建了一个名为address.v1.service的upstream，并且给这个upstream添加了两个target节点，权重分别为100和50。当配置API的时候，只要将stream_url设置为http://address.v1.service即可。当接收到请求时，2/3的请求将会转发到192.168.34.15:80，1/3的请求转发到192.168.34.16:80

#### Nginx负载均衡
就是利用Nginx反向代理做负载均衡，然后将API的upstream_url配置为代理服务器的地址。这个方法具有下面几个优势：

- nginx的upstream模块提供了多种调度算法，例如加权轮询、ip_hash、url_hash、fair等，使用起来更加灵活
- nginx的upstream提供了健康检查机制，如果后端服务器工作不正常，nginx会将这个节点标记为down，大大提高了可用性

>综上所属，个人觉得Nginx负载均衡是最为理想的一种方案

### Kong集群

### 插件



