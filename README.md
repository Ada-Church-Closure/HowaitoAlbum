# Howaito-Album

​	项目名称来源于著名游戏「ホワイト　アルバム(White Album)」,是一个用来做动漫点评(类似于Bangumi),并且进行相关的周边抢购(类似于Bilibili会员购),同好交流,相互关注的软件.

## 技术栈使用

> Spring Boot 2.x Spring MVC 
**数据存储层**：
> MySQL：存储数据 
> MyBatis Plus：数据访问框架 
> **Redis 相关**：
> spring-data-redis：操作
> Redis Lettuce：操作 Redis 的高级客户端
> Apache Commons Pool：用于实现 Redis 连接池 
> Redisson：基于 Redis 的分布式数据网格
**工具库**：
> HuTool：工具库合集 Lombok：注解式代码生成工具



## 库表结构分析

> 必须搞清楚这里的关系!

先看一下结构,有几张table,分析一些table的基本结构.

```sql
MariaDB [hmdp]> show tables;
+--------------------+
| Tables_in_hmdp     |
+--------------------+
| tb_blog            |
| tb_blog_comments   |
| tb_follow          |
| tb_seckill_voucher |
| tb_shop            |
| tb_shop_type       |
| tb_sign            |
| tb_user            |
| tb_user_info       |
| tb_voucher         |
| tb_voucher_order   |
+--------------------+
11 rows in set (0.001 sec)
```

### tb_user

> 存储用户基本信息,我们怎么利用密码.
>
> 这里关注的是用户的基本登陆信息.

```sql
tb_user | CREATE TABLE `tb_user` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `phone` varchar(11) NOT NULL COMMENT '手机号码',
  `password` varchar(128) DEFAULT '' COMMENT '密码，加密存储',
  `nick_name` varchar(32) DEFAULT '' COMMENT '昵称，默认是用户id',
  `icon` varchar(255) DEFAULT '' COMMENT '人物头像',
  `create_time` timestamp NOT NULL DEFAULT current_timestamp() COMMENT '创建时间',
  `update_time` timestamp NOT NULL DEFAULT current_timestamp() ON UPDATE current_timestamp() COMMENT '更新时间',
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE KEY `uniqe_key_phone` (`phone`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=1020 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci ROW_FORMAT=COMPACT                                  
```

### tb_user_info

> 这是用户使用我们服务的基本信息.

```sql
| tb_user_info | CREATE TABLE `tb_user_info` (
  `user_id` bigint(20) unsigned NOT NULL COMMENT '主键，用户id',
  `city` varchar(64) DEFAULT '' COMMENT '城市名称',
  `introduce` varchar(128) DEFAULT NULL COMMENT '个人介绍，不要超过128个字符',
  `fans` int(8) unsigned DEFAULT 0 COMMENT '粉丝数量',
  `followee` int(8) unsigned DEFAULT 0 COMMENT '关注的人的数量',
  `gender` tinyint(1) unsigned DEFAULT 0 COMMENT '性别，0：男，1：女',
  `birthday` date DEFAULT NULL COMMENT '生日',
  `credits` int(8) unsigned DEFAULT 0 COMMENT '积分',
  `level` tinyint(1) unsigned DEFAULT 0 COMMENT '会员级别，0~9级,0代表未开通会员',
  `create_time` timestamp NOT NULL DEFAULT current_timestamp() COMMENT '创建时间',
  `update_time` timestamp NOT NULL DEFAULT current_timestamp() ON UPDATE current_timestamp() COMMENT '更新时间',
  PRIMARY KEY (`user_id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci ROW_FORMAT=COMPACT                                                                  |
```

### tb_shop_type

> 商户的基本类型.

```sql
| tb_shop_type | CREATE TABLE `tb_shop_type` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `name` varchar(32) DEFAULT NULL COMMENT '类型名称',
  `icon` varchar(255) DEFAULT NULL COMMENT '图标',
  `sort` int(3) unsigned DEFAULT NULL COMMENT '顺序',
  `create_time` timestamp NULL DEFAULT current_timestamp() COMMENT '创建时间',
  `update_time` timestamp NULL DEFAULT current_timestamp() ON UPDATE current_timestamp() COMMENT '更新时间',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci ROW_FORMAT=COMPACT                   |

```

### tb_shop

> 我们持有哪些基本商户?卖周边的**shop**.

```sql
| tb_shop | CREATE TABLE `tb_shop` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `name` varchar(128) NOT NULL COMMENT '商铺名称',
  `type_id` bigint(20) unsigned NOT NULL COMMENT '商铺类型的id',
  `images` varchar(1024) NOT NULL COMMENT '商铺图片，多个图片以'',''隔开',
  `area` varchar(128) DEFAULT NULL COMMENT '商圈，例如陆家嘴',
  `address` varchar(255) NOT NULL COMMENT '地址',
  `x` double unsigned NOT NULL COMMENT '经度',
  `y` double unsigned NOT NULL COMMENT '维度',
  `avg_price` bigint(10) unsigned DEFAULT NULL COMMENT '均价，取整数',
  `sold` int(10) unsigned zerofill NOT NULL COMMENT '销量',
  `comments` int(10) unsigned zerofill NOT NULL COMMENT '评论数量',
  `score` int(2) unsigned zerofill NOT NULL COMMENT '评分，1~5分，乘10保存，避免小数',
  `open_hours` varchar(32) DEFAULT NULL COMMENT '营业时间，例如 10:00-22:00',
  `create_time` timestamp NULL DEFAULT current_timestamp() COMMENT '创建时间',
  `update_time` timestamp NULL DEFAULT current_timestamp() ON UPDATE current_timestamp() COMMENT '更新时间',
  PRIMARY KEY (`id`) USING BTREE,
  KEY `foreign_key_type` (`type_id`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=15 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci ROW_FORMAT=COMPACT |
```











## 短信登陆

### 使用session来实现登陆

​	这里就是把session存放在redis内部,返回一个token给client,其实也就是cookie(这里是是**authorization**的一个字段,原理和cookie机制是类似的),之后用户每次访问的时候都需要携带这个token和redis里面:

```java
 // 把这个登陆的信息存放到redis内部
        String tokenKey = LOGIN_USER_KEY + token;
        stringRedisTemplate.opsForHash().putAll(tokenKey, userMap);
        // 这个登陆也设置有效期,30min
        stringRedisTemplate.expire(tokenKey, LOGIN_USER_TTL, TimeUnit.MINUTES);
```

​	那么就其实是**redis**内部维护了这样一个session的对象,第一层拦截器可以刷新token,第二层拦截器可以直接利用redis内部存储的数据来进行登陆校验.

下面就是分布式的实现:

### 服务器集群的session共享的问题

nginx会进行负载均衡的处理,不同服务器之间没有保存相同的session怎么处理?

> 这个问题的意思就是,在点击发送验证码请求之后,和后续验证验证码登陆,如果这个两个请求不在一个服务器上怎么说.

> 把这些session全部存储到redis内部.

## 使用redis缓存存储用户访问的数据

redis本身有**内存淘汰**机制--->内存不足的时候直接去掉这部分内存.(一致性需求很低,就是基本不会改动.)

**超时剔除**,缓存的数据增加TTL时间,到时自动删除缓存.

**主动更新**--->每次修改数据库的时候,都修改redis.(对于一致性需求很高.)



缓存写回,有其他线程异步进行缓存的持久化操作.

**Cache Aside Pattern**

> 缓存的调用者,在**更新数据库的同时去更新缓存**.

1.更新数据库直接让缓存失效,再次查询的时候才进行加载.

2.缓存数据库同时更新原子性--->事务!

3.操作顺序?--->**先操作数据库,再删除缓存,这样带来的并发性问题更小.**

### 缓存穿透

> 客户端的请求在**redis**和**mysql**中都不存在,所有的请求都会直接到达数据库.

1.缓存null对象(控制TTL),短期的不一致性(就是数据库中没有,我们就先**暂时存放一个null**对象,但是如果这期间更新了数据库,那么就会出现**不一致**的情况,但是时间比较短)

2.**布隆过滤**--->算法

内存占用少,二进制保存.

实现复杂.存在误判的可能. 

3.增强id的复杂度,难以被预测规律.

4.基础格式的校验--->主动防范.

5.热点参数**限流**.

### 缓存雪崩

同一时间内,大量的key失效或者redis服务宕机,大量请求直接到达数据库导致的.

1.TTL随机值,不要大家都一样.

2.redis集群.

3.缓存业务的降级,限流策略.

4.业务添加**多级缓存**.(nginx缓存,jvm缓存).

### 缓存击穿(热点key问题)

**高并发访问 或者 缓存重建业务比较复杂** 的 **key** 突然失效了,许多访问会给数据库带来冲击.

> 就是很多线程都在同时尝试创建一个缓存.

**1.mutex** 互斥锁

获取mutex--->重建缓存,写入缓存--->释放mutex.(线程要进行等待,性能有影响.)

**setnx**就可以达到这样的效果.

**2.逻辑过期**

性能很好,直接返回过期数据/或者拿到互斥锁,开一个新线程进行重建的操作.

## redis实现优惠券秒杀

### 全局ID生成器(订单号)

唯一性 递增 随机(安全)

Long类型

### 超卖问题

比如很多人一起抢订单,就会出现问题,把数据库减成负数.

> 指定数据库是非负的是否是解决问题的方式,不可以解决问题.

#### 乐观锁

> 不实际加锁,只是一种逻辑.

每次修改前先检查,如果可以认为数据没有被修改过,就直接修改.

被修改的话就不来执行.

1.使用**版本号**解决问题.

其实就是**CAS**--->**compare and set**:在修改之前和查询时**value**进行比较,看有没有发生变化.

> 你也可以和0进行比较,但是即使是0,那个瞬间还是可能会出问题.

这样的失败概率也会提升,很简单,因为只要有一个线程成功修改了,这期间的其余线程都会直接无效.

> 分段锁方案,就是一次锁的资源变少,分到多个数据库来减少压力.

### *一人一单

> 不能让一个人抢太多单,这样就成黄牛了,这是本项目的核心难点.

```conf
   upstream backend {
        server 127.0.0.1:8081 max_fails=5 fail_timeout=10s weight=1;
        server 127.0.0.1:8082 max_fails=5 fail_timeout=10s weight=1;
    }  
```

启用**nginx**的**反向代理**和**负载均衡**.

### 负载均衡/并发问题解决

​	我们在上面启动多个服务,在nginx上做负载均衡,来的请求就会分配到这两个服务上,来同时对于数据库进行操作,这里是采用轮询的规则.

> ​	那么很明显,上面的这样负载均衡,解决不了一人一单的问题,因为**lock**是单独的,就是一个**jvm**锁监视器维护了这样的一把**lock**,当有多个**jvm**的时候,就会产生更多的并行线程.
>
> ​	那么我们就要实现,跨进程,或者说跨**jvm**的锁来解决这个问题.

## 分布式lock

> **分布式**或者**集群**下,**多进程同时可见**的**互斥**lock.
>
> 这个核心目的是**防止一人多单**.
>
> **高可用/高性能/安全性**

让多个**jvm**使用相同的锁监视器.

​	这里是利用redis的**setnx**等互斥命令,并且设置一个key的过期时间,防止获取lock之后,没有及时释放,导致问题.

​	非阻塞式获取lock,获取失败的话直接返回false,**我们不等**,用户抢失败了可以再发送请求进行抢夺,而不是进行阻塞式的等待.

### lua脚本实现

```lua
-- 锁的key,实现原子性的基于redis的分布式lock,整个lua脚本是具有原子性的
local key = KEYS[1]
-- 当前线程的标识
local threadId = ARGV[1]

-- 获取lock中的线程的标识
local id = redis.call('get', key)

-- 比较线程标识与lock中的标识是否一致
if(id == threadId) then
    -- 释放lock
    return redis.call('del', key)
end
return 0
```

因为lua脚本执行一系列的redis指令具有原子性.

## 利用Redisson

分布式系统工具的集合.

> 之前的**lock**是**不可重入**的,不可重入lock的**重复调用**会导致死锁.
>
> 不可重试.
>
> 超时释放问题.
>
> 主从一致性.

### Redisson可重入lock的原理

> 这也就是Redisson的底层实现的原理,也是利用lua脚本.

​	把原来的**lock**对应的**value**字段更改成hash的结构(**threadID:counter**--->这样的对应即可),每获取一次lock,增加一次引用计数,释放一次lock减少一次引用计数即可,这个过程可以使用一个lua脚本解决.

### 重试机制

利用信号量,不是忙等待,并且有通知机制.

leaseTime不断地更新有效期,**看门狗机制**不断重置超时的时间.

> WatchDog解决的问题就是业务代码的执行长度大于了这个过期时间,就会产生问题.
>
> 先设置默认30s,接下来每隔10s检查lock是否持有,如果持有就会更新这个过期时间.--->直到业务手动释放.
>
> **无参的时候才会启用这个机制.**

### 主从一致性

主要是**主从同步延迟**导致的问题.

> 利用**联锁**机制.

**MultiLock**--->采用**多个主节点**而不是一个主节点,**只有所有主节点上都获取了lock的时候,才算真正获取了lock.**

还有多个lock的同步机制.

## 秒杀优化

> 这是我们为什么引入**消息队列**的机制!

​	当用户请求来的时候.如果有库存以及合法的一人一单的时候,我们就可以确定这个用户可以抢到订单了,不需要等到完全创建和写入mysql数据库内部才能返回,我们可以把这个两个过程解耦,使得更新内存和sql数据库之间是异步进行的即可.--->利用一个阻塞队列来进行异步的下单.

​	jdk阻塞队列,队列可能会导致内存溢出/内存安全问题/异常--->如何解决

## *消息队列

> 基本的生产者,消费者模型,也是在解除耦合.

区别是

1.单独的服务,不受jvm内存的限制.

2.安全,会做消息的持久化.

**redis List**模拟**消息队列**.--->直接用**push** 和**Bpop**阻塞等待来进行模拟.

> 但是消息会丢失,并且单消费者.

**PubSub**(发布订阅模型)--->消息传递模型.多生产,多消费.

> 不支持持久化,丢失,存储上限,纯废物.

### 基于**stream**的消息队列

生成消息ID唯一标识.

> 这是单消费的模式.

> **XADD**和**XREAD**方法,多消费者可读取,但是可能漏读,消息可以回溯.

#### 消费者组

> 利用消费者组,这是多消费者模式.

1.一个组内消息给多消费者**竞争**,处理速度很快.--->分流.

2.维护**最后一个被处理的消息的标识**,即使重启,也可以进行恢复的操作.--->标识.

3.**确认机制**,当消费者拿到消息之后(消息会进入一个pending-list,也就是**等待消息确认**的队列),处理完成之后返回XACK确认已经处理.--->更安全.

​	如果抛出异常,去处理pending-list里面的异常消息.

> 这里的通信机制就比较像计算机网络的部分.

\> 是从下一个未消费的消息开始读取,我们这样就不会造成重复读的问题.

> 要求不高的情况下,**Stream**是可以解决大多数问题的.

我们尝试使用stream来解决:

> 注意每次清空都要创建这个消费者组.

```sh
127.0.0.1:6379> XGROUP CREATE stream.orders g1 0 MKSTREAM
OK
# g1 是消费者组
# 从0开始消费
# MKSTREAM如果流不存在,就自动创建这个流
```

并且直接在之前的**lua**脚本中往这个队列中**输送消息**.

消息相关操作:

|      命令       |        作用        |
| :-------------: | :----------------: |
|     `XADD`      | 向流中添加一条消息 |
|    `XRANGE`     | 按 ID 范围读取消息 |
|     `XREAD`     |    普通读取消息    |
| `XGROUP CREATE` |    创建消费者组    |
|  `XREADGROUP`   |    按组读取消息    |
|     `XACK`      |      确认消息      |
| `XINFO GROUPS`  |     查看组信息     |
| `XINFO STREAM`  |     查看流信息     |
|     `XDEL`      |    删除指定消息    |

这是我们的lua脚本:

```lua
---
--- Generated by EmmyLua(https://github.com/EmmyLua)
--- Created by ada.
--- DateTime: 10/30/25 11:13 AM
---
---
--- 优惠券id 和 用户id 作为参数
--- v1.1 我们这里还要加入消息队列的控制
local voucherId = ARGV[1]
local userId = ARGV[2]
local orderId = ARGV[3]

--- 数据的key
local stockKey = 'seckill:stock:' .. voucherId
local orderKey = 'seckill:order' .. voucherId

--- 脚本业务
if(tonumber(redis.call("get", stockKey)) <= 0) then
    return 1
end

if(redis.call('sismember', orderKey, userId) == 1) then
    return 2
end

redis.call('incrby', stockKey, -1)
redis.call('sadd', orderKey, userId)
--- 发现有资格,我们发送一个消息到队列中
redis.call('xadd', 'stream.orders', '*', 'userId', userId, 'voucherId', voucherId, 'id', orderId)
return 0
```

## 点评功能

> 这是核心的,也有可移植性的功能.
>
> 上传,点赞排行等.

### Note:发布笔记的功能/评价

> 实现一些简单的接口即可.

利用**sortedSet**实现点赞排行榜的功能.

> 这里的分数,也就是排序的score,可以直接用时间戳.

sql查询的小问题:

```java
 // 这里sql查询的问题 in(5, 1) 也会从1开始查询       
 // 利用 ORDER BY FILED (id, 5, 1) 来进行查询 
 List<UserDTO> userDTOS = userService.l
         .stream()                      
         .map(user -> BeanUtil.copyProp
         .collect(Collectors.toList());
```

感觉处理不是很优雅:

```java
  List<Long> ids = top5UserId.stream().map(Long::valueOf).collect(Collectors.toList());         
  String idStr = StrUtil.join(",", ids);
  // 3.用这些id查询出来对应的user
  // 这里sql查询的问题 in(5, 1) 也会从1开始查询
  // 利用 ORDER BY FILED (id, 5, 1) 来进行查询
  // 好复杂
  List<UserDTO> userDTOS = userService.query()
          .in("id", ids)
          .last("ORDER BY FIELD(id," + idStr + ")")
          .list()
          .stream()
          .map(user -> BeanUtil.copyProperties(user, UserDTO.class))
          .collect(Collectors.toList());
```

## 好友/关注的实现

看这张follow表的结构:

```sql
CREATE TABLE `tb_follow` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `user_id` bigint(20) unsigned NOT NULL COMMENT '用户id',
  `follow_user_id` bigint(20) unsigned NOT NULL COMMENT '关联的用户id',
  `create_time` timestamp NOT NULL DEFAULT current_timestamp() COMMENT '创建时间',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci ROW_FORMAT=COMPACT
```

### 关注和取关

直接插入mysql数据.

### 共同关注

求set的交集.--->那么我们每次关注的时候,也要把这个数据放到redis内部才可以.

比如这就是实现的逻辑:

```java
@Override
    public Result followCommons(Long id) {
        Long userId = UserHolder.getUser().getId();
        String myKey = FOLLOWS_KEY + userId;
        String targetKey = FOLLOWS_KEY + id;
        // 求交集
        Set<String> intersectFollows = stringRedisTemplate.opsForSet().intersect(myKey, targetKey);

        if(intersectFollows == null || intersectFollows.isEmpty()){
            return Result.ok(Collections.emptyList());
        }

        List<Long> ids = intersectFollows.stream().map(Long::valueOf).collect(Collectors.toList());
        // 查询交集的关注
        List<UserDTO> userDTOList = userService.listByIds(ids)
                .stream()
                .map(user -> BeanUtil.copyProperties(user, UserDTO.class))
                .collect(Collectors.toList());

        return Result.ok(userDTOList);
    }
```

### 关注推送

> **Feed流**--->持续提供沉浸式的体验,下拉获取新的信息.

#### 智能排序

粘度很高,推送用户感兴趣的信息.

#### Timeline

直接根据时间,不进行筛选,朋友圈.

> 我们实现Timeline,直接推送的关注.

##### 拉模式(读扩散)

先拉取,然后进行排序的操作.

##### 推模式(写扩散)

创作者写的时候,直接写入所有用户的收件箱.--->实现简单,延时比较低.

##### 推拉结合(读写混合模式)

粉丝少,直接推送,推模式写扩散.

大V,一般放在发件箱,也就是拉模式读扩散,对于狂热粉丝,可以直接推模式写扩散.

> 显然我们使用推模式.

feed流数据更新,角标也发生变化,不能采取传统的分页模式.

利用滚动分页,这次的**起始角标是上一次的结束位置**开始的.--->利用sortedset!

> ​	解决的问题就是查询期间,如果有大量的数据插入,角标会发生变化,但是如果本次查询利用的是上一次查询的**最小值**(在这个基础上做偏移,**偏移量**是(**在上一次的查询结果中,和最小值一样的结果的个数**)),就没有什么问题了.

我们经过本次查询之后会返回一些参数,下一次前端就会使用这些参数来进行新的查询.

## 附近商户

### GEO数据结构

> 存储地理坐标信息.

redis内部有的数据结构--->底层还是Zset

这里的业务逻辑太复杂,其实就是按照距离排序然后进行分页的查询.

## 用户签到

sign table的结构:

```sql
CREATE TABLE `tb_sign` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `user_id` bigint(20) unsigned NOT NULL COMMENT '用户id',
  `year` year(4) NOT NULL COMMENT '签到的年',
  `month` tinyint(2) NOT NULL COMMENT '签到的月',
  `date` date NOT NULL COMMENT '签到的日期',
  `is_backup` tinyint(1) unsigned DEFAULT NULL COMMENT '是否补签',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci ROW_FORMAT=COMPACT
```

我们这里使用**redis**中的**BitMap**--->二进制来进行处理.

一个bit对应一个月的某一天.

> 这里前端好像没有接口?

统计:连续签到的次数.

## UV统计

### HyperLogLog

> 这是一种概率算法,我们统计用户的访问量.

UV:Unique Visitor 独立访客量.

PV:Page View 页面的点击量. 











# Frontend

先把前端跑起来

终止nginx服务:

```sh
# 请确保您在项目根目录（~/Project/HowaitoAlbum/HowaitoAlbum-frontend）下执行此命令
sudo nginx -p $(pwd) -c conf/nginx.conf -s stop
```

运行nginx服务:

```sh
sudo nginx -p $(pwd) -c conf/nginx.conf # 在当前的目录下方
```

> 
>
> 简单分析一下前端就可以.

```sh
$tree -L
.
├── conf
│   ├── fastcgi.conf
│   ├── fastcgi_params
│   ├── koi-utf
│   ├── koi-win
│   ├── mime.types
│   ├── nginx.conf
│   ├── scgi_params
│   ├── uwsgi_params
│   └── win-utf
├── contrib
│   ├── geo2nginx.pl
│   ├── README
│   ├── unicode2nginx
│   │   ├── koi-utf
│   │   ├── unicode-to-nginx.pl
│   │   └── win-utf
│   └── vim
│       ├── ftdetect
│       ├── ftplugin
│       ├── indent
│       └── syntax
├── docs
│   ├── CHANGES
│   ├── CHANGES.ru
│   ├── LICENSE
│   ├── OpenSSL.LICENSE
│   ├── PCRE.LICENCE
│   ├── README
│   └── zlib.LICENSE
├── html
│   ├── 50x.html
│   ├── hmdp
│   │   ├── blog-detail.html
│   │   ├── blog-edit.html
│   │   ├── css
│   │   ├── favicon.ico
│   │   ├── imgs
│   │   ├── index.html
│   │   ├── info-edit.html
│   │   ├── info.html
│   │   ├── js
│   │   ├── login2.html
│   │   ├── login.html
│   │   ├── other-info.html
│   │   ├── shop-detail.html
│   │   └── shop-list.html
│   └── index.html
├── logs
│   ├── access.log
│   └── error.log
└── temp
    ├── client_body_temp
    ├── fastcgi_temp
    ├── proxy_temp
    ├── scgi_temp
    └── uwsgi_temp
```

| 目录         | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| `conf/`      | nginx 的配置文件目录，`nginx.conf` 是主配置文件，`mime.types` 定义文件类型 |
| `html/`      | **静态网页根目录**，也就是 nginx 默认展示的内容所在处        |
| `html/hmdp/` | 🚀 前端页面目录（`index.html`、`shop-list.html` 等都在这里）  |
| `logs/`      | nginx 的访问和错误日志                                       |
| `temp/`      | nginx 临时文件目录                                           |
| `docs/`      | 文档和许可证                                                 |
| `contrib/`   | 一些扩展脚本，无需修改                                       |



