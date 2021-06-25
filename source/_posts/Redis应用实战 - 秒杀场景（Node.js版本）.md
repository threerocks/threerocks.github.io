---
title: Redis应用实战 - 秒杀场景（Node.js版本）
date: 2021-06-25 17:07:25
tags:
- Node.js
- Javascript
- Mysql
- Redis
- 秒杀
---

## 写在前面
公司随着业务量的增加，最近用时几个月时间在项目中全面接入`Redis`，开发过程中发现市面上缺少具体的实战资料，尤其是在`Node.js`环境下，能找到的资料要么过于简单入门，要么名不副实，大部分都是属于初级。因此决定把公司这段时间的成果进行分享，会用几篇文章详细介绍`Redis`的几个使用场景，期望大家一起学习、进步。   
下面就开始第一篇，秒杀场景。
## 业务分析
实际业务中，秒杀包含了许多场景，具体可以分为秒杀前、秒杀中和秒杀后三个阶段，从开发角度具体分析如下：

1. 秒杀前：主要是做好缓存工作，以应对用户频繁的访问，因为数据是固定的，可以把商品详情页的元素静态化，然后用`CDN`或者是浏览器进行缓存。
1. 秒杀中：主要是库存查验，库存扣减和订单处理，这一步的特点是
    - 短时间内大量用户同时进行抢购，系统的流量突然激增，服务器压力瞬间增大（瞬时并发访问高）
    - 请求数量大于商品库存，比如10000个用户抢购，但是库存只有100
    - 限定用户只能在一定时间段内购买
    - 限制单个用户购买数量，避免刷单
    - 抢购是跟数据库打交道，核心功能是下单，库存不能扣成负数
    - 对数据库的操作读多写少，而且读操作相对简单
    
1. 秒杀后：主要是一些用户查看已购订单、处理退款和处理物流等等操作，这时候用户请求量已经下降，操作也相对简单，服务器压力不大。

根据上述分析，本文把重点放在秒杀中的开发讲解，其他部分感兴趣的小伙伴可以自己搜索资料，进行尝试。
    
## 开发环境
数据库：`Redis 3.2.9` + `Mysql 5.7.18`
服务器：`Node.js v10.15.0`
测试工具：`Jmeter-5.4.1`

<!--more-->

## 实战
### 数据库准备
![tables](media/16239877134495/tables.jpg)
如图所示，`Mysql`中需要创建三张表，分别是

- 产品表，用于记录产品信息，字段分别为Id、名称、缩略图、价格和状态等等
- 秒杀活动表，用于记录秒杀活动的详细信息，字段分别为Id、参与秒杀的产品Id、库存量、秒杀开始时间、秒杀结束时间和秒杀活动是否有效等等
- 订单表，用于记录下单后的数据，字段分别为Id、订单号、产品Id、购买用户Id、订单状态、订单类型和秒杀活动Id等等

下面是创建`sql`语句，以供参考

```sql
CREATE TABLE `scekill_goods` (
	`id` INTEGER NOT NULL auto_increment,
	`fk_good_id` INTEGER,
	`amount` INTEGER,
	`start_time` DATETIME,
	`end_time` DATETIME,
	`is_valid` TINYINT ( 1 ),
	`comment` VARCHAR ( 255 ),
	`created_at` DATETIME NOT NULL,
	`updated_at` DATETIME NOT NULL,
PRIMARY KEY ( `id` ) 
) ENGINE = INNODB DEFAULT CHARSET = utf8mb4;
```


```sql
CREATE TABLE `orders` (
	`id` INTEGER NOT NULL auto_increment,
	`order_no` VARCHAR ( 255 ),
	`good_id` INTEGER,
	`user_id` INTEGER,
	`status` ENUM ( '-1', '0', '1', '2' ),
	`order_type` ENUM ( '1', '2' ),
	`scekill_id` INTEGER,
	`comment` VARCHAR ( 255 ),
	`created_at` DATETIME NOT NULL,
	`updated_at` DATETIME NOT NULL,
PRIMARY KEY ( `id` ) 
) ENGINE = INNODB DEFAULT CHARSET = utf8mb4;
```


```sql
CREATE TABLE `goods` (
	`id` INTEGER NOT NULL auto_increment,
	`name` VARCHAR ( 255 ),
	`thumbnail` VARCHAR ( 255 ),
	`price` INTEGER,
	`status` TINYINT ( 1 ),
	`stock` INTEGER,
	`stock_left` INTEGER,
	`description` VARCHAR ( 255 ),
	`comment` VARCHAR ( 255 ),
	`created_at` DATETIME NOT NULL,
	`updated_at` DATETIME NOT NULL,
PRIMARY KEY ( `id` ) 
) ENGINE = INNODB DEFAULT CHARSET = utf8mb4;
```
  
产品表在此次业务中不是重点，以下逻辑都以`id=1`的产品为示例，请悉知。   
秒杀活动表中创建一条库存为200的记录，作为秒杀测试数据，参考下面语句：

```sql
INSERT INTO `redis_app`.`seckill_goods` (
	`id`,
	`fk_good_id`,
	`amount`,
	`start_time`,
	`end_time`,
	`is_valid`,
	`comment`,
	`created_at`,
	`updated_at` 
)
VALUES
	(
		1,
		1,
		200,
		'2020-06-20 00:00:00',
		'2023-06-20 00:00:00',
		1,
		'...',
		'2020-06-20 00:00:00',
		'2021-06-22 10:18:16' 
	);
```

### 秒杀接口开发
首先，说一下`Node.js`中的具体开发环境:

- `web`框架使用`Koa2`
- `mysql`操作使用基于`promise`的`Node.js` `ORM`工具`Sequelize`
- `redis`操作使用`ioredis`库
- 封装`ctx.throwException`方法用于处理错误，封装`ctx.send`方法用于返回正确结果，具体实现参考文末完整代码

其次，分析一下接口要处理的逻辑，大概步骤和顺序如下：

1. 基本参数校验
1. 判断产品是否加入了抢购
1. 判断秒杀活动是否有效
1. 判断秒杀活动是否开始、结束
1. 判断秒杀商品是否卖完
1. 获取登录用户信息
1. 判断登录用户是否已抢到
1. 扣库存
1. 下单

最后，根据分析把以上步骤用代码进行初步实现，如下：

```js
// 引入moment库处理时间相关数据
const moment = require('moment');
// 引入数据库model文件
const seckillModel = require('../../dbs/mysql/models/seckill_goods');
const ordersModel = require('../../dbs/mysql/models/orders');
// 引入工具函数或工具类
const UserModule = require('../modules/user');
const { random_String } = require('../../utils/tools/funcs');

class Seckill {
  /**
   * 秒杀接口
   * 
   * @method post
   * @param good_id 产品id
   * @param accessToken 用户Token
   * @param path 秒杀完成后跳转路径
   */
  async doSeckill(ctx, next) {
    const body = ctx.request.body;
    const accessToken = ctx.query.accessToken;
    const path = body.path;

    // 基本参数校验
    if (!accessToken || !path) { return ctx.throwException(20001, '参数错误！'); };
    // 判断此产品是否加入了抢购
    const seckill = await seckillModel.findOne({
      where: {
        fk_good_id: ctx.params.good_id,
      }
    });
    if (!seckill) { return ctx.throwException(30002, '该产品并未有抢购活动！'); };
    // 判断是否有效
    if (!seckill.is_valid) { return ctx.throwException(30003, '该活动已结束！'); };
    // 判单是否开始、结束
    if(moment().isBefore(moment(seckill.start_time))) {
      return ctx.throwException(30004, '该抢购活动还未开始！');
    }
    if(moment().isAfter(moment(seckill.end_time))) {
      return ctx.throwException(30005, '该抢购活动已经结束！');
    }
    // 判断是否卖完
    if(seckill.amount < 1) { return ctx.throwException(30006, '该产品已经卖完了！'); };

    //获取登录用户信息(这一步只是简单模拟验证用户身份，实际开发中要有严格的accessToken校验流程)
    const userInfo = await UserModule.getUserInfo(accessToken);
    if (!userInfo) { return ctx.throwException(10002, '用户不存在！'); };

    // 判断登录用户是否已抢到（一个用户针对这次活动只能购买一次）
    const orderInfo = await ordersModel.findOne({
      where: {
        user_id: userInfo.id,
        seckill_id: seckill.id,
      },
    });
    if (orderInfo) { return ctx.throwException(30007, '该用户已抢到该产品，无需再抢！'); };

    // 扣库存
    const count = await seckill.decrement('amount');
    if (count.amount <= 0) { return ctx.throwException(30006, '该产品已经卖完了！'); };

    // 下单
    const orderData = {
      order_no: Date.now() + random_String(4), // 这里就用当前时间戳加4位随机数作为订单号，实际开发中根据业务规划逻辑 
      good_id: ctx.params.good_id,
      user_id: userInfo.id,
      status: '1', // -1 已取消, 0 未付款， 1 已付款， 2已退款
      order_type: '2', // 1 常规订单 2 秒杀订单
      seckill_id: seckill.id, // 秒杀活动id
      comment: '', // 备注
    };
    const order = ordersModel.create(orderData);

    if (!order) { return ctx.throwException(30008, '抢购失败!'); };

    ctx.send({
      path,
      data: '抢购成功!'
    });

  }

}

module.exports = new Seckill();
```

至此，秒杀接口用传统的关系型数据库就实现完成了，代码并不复杂，注释也很详细，不用特别的讲解大家也都能看懂，那它能不能正常工作呢，答案显然是否定的   
通过`Jmeter`模拟以下测试：

- 模拟5000并发下2000个用户进行秒杀，会发现`mysql`报出`timeout`错误，同时`seckill_goods`表`amount`字段变成负数，`orders`表中同样产生了多于200的记录（具体数据不同环境下会有差异），这就代表产生了超卖，跟秒杀规则不符
- 模拟10000并发下单个用户进行秒杀，`orders`表中产生了多于1条的记录（具体数据不同环境下会有差异），这就说明一个用户针对这次活动买了多次，跟秒杀规则不符


分析下代码会发现这其中的问题：

- 步骤2，判断此产品是否加入了抢购   

    直接在`mysql`中查询，因为是在秒杀场景下，并发会很高，大量的请求到数据库，显然mysql是扛不住的，毕竟`mysql`每秒只能支撑千级别的并发请求
- 步骤7，判断登录用户是否已抢到   

    在高并发下同一个用户上个订单还没有生成成功，再次判断是否抢到依然会判断为否，这种情况下代码并没有对扣减和下单操作做任何限制，因此就产生了单个用户购买多个产品的情况，跟一个用户针对这次活动只能购买一次的要求不符
- 步骤8，扣库存操作   
    
    假设同时有1000个请求，这1000个请求在步骤5判断产品是否秒杀完的时候查询到的库存都是200，因此这些请求都会执行步骤8扣减库存，那库存肯定会变成负数，也就是产生了超卖现象



### 解决方案
经过分析得到三个问题需要解决：

1. 秒杀数据需要支持高并发访问
2. 一个用户针对这次活动只能购买一次的问题，也就是限购问题
3. 减库存不能扣成负数，订单数不能超过设置的库存数，也就是超卖问题

`Redis`作为内存型数据库，本身高速处理请求的特性可以支持高并发。针对超卖，也就是库存扣减变负数情况，`Redis`可以提供`Lua`脚本保证原子性和分布式锁两个解决高并发下数据不一致的问题。针对一个用户只能购买一次的要求，`Redis`的分布式锁可以解决问题。   
因此，可以尝试用Redis解决上述问题，具体操作：

- 为了支撑大量高并发的库存查验请求，需要用`Redis`保存秒杀活动数据（即`seckill_goods`表数据），这样一来请求可以直接从`Redis`中读取库存并进行查询，完成查询之后如果还有库存余量，就直接从`Redis`中扣除库存
- 扣减库存操作在`Redis`中进行，但是因为`Redis`扣减这一操作是分为读和写两个步骤，也就是必须先读数据进行判断再执行减操作，因此如果对这两个操作没有做好控制，就导致数据被改错，依然会出现超卖现象，为了保证并发访问的正确性需要使用原子操作解决问题，`Redis`提供了使用`Lua`脚本包含多个操作来实现原子性的方案   
以下是`Redis`官方文档对`Lua`脚本原子性的解释

    >Atomicity of scripts
    Redis uses the same Lua interpreter to run all the commands. Also Redis guarantees that a script is executed in an atomic way: no other script or Redis command will be executed while a script is being executed. This semantic is similar to the one of MULTI / EXEC. From the point of view of all the other clients the effects of a script are either still not visible or already completed.
    
- 使用`Redis`实现分布式锁，对扣库存和写订单操作进行加锁，以保证一个用户只能购买一次的问题。
    
### 接入Redis
首先，不再使用`seckill_goods`表，新增秒杀活动逻辑变为在`Redis`中插入数据，类型为`hash`类型，`key`规则为`seckill_good_ + 产品id`，现在假设新增一条`key`为`seckill_good_1`的记录，值为

```js
{
    amount: 200,
    start_time: '2020-06-20 00:00:00',
    end_time: '2023-06-20 00:00:00',
    is_valid: 1,
    comment: '...',
  }
```
其次，创建`lua`脚本保证扣减操作的原子性，脚本内容如下

```lua
if (redis.call('hexists', KEYS[1], KEYS[2]) == 1) then
  local stock = tonumber(redis.call('hget', KEYS[1], KEYS[2]));
  if (stock > 0) then
    redis.call('hincrby',  KEYS[1], KEYS[2], -1);
    return stock
  end;
  return 0
end;
```
最后，完成代码，完整代码如下：

```js
// 引入相关库
const moment = require('moment');
const Op = require('sequelize').Op;
const { v4: uuidv4 } = require('uuid');
// 引入数据库model文件
const seckillModel = require('../../dbs/mysql/models/seckill_goods');
const ordersModel = require('../../dbs/mysql/models/orders');
// 引入Redis实例
const redis = require('../../dbs/redis');
// 引入工具函数或工具类
const UserModule = require('../modules/user');
const { randomString, checkObjNull } = require('../../utils/tools/funcs');
// 引入秒杀key前缀
const { SECKILL_GOOD, LOCK_KEY } = require('../../utils/constants/redis-prefixs');
// 引入避免超卖lua脚本
const { stock, lock, unlock } = require('../../utils/scripts');

class Seckill {
  async doSeckill(ctx, next) {
    const body = ctx.request.body;
    const goodId = ctx.params.good_id;
    const accessToken = ctx.query.accessToken;
    const path = body.path;

    // 基本参数校验
    if (!accessToken || !path) { return ctx.throwException(20001, '参数错误！'); };
    // 判断此产品是否加入了抢购
    const key = `${SECKILL_GOOD}${goodId}`;
    const seckill = await redis.hgetall(key);
    if (!checkObjNull(seckill)) { return ctx.throwException(30002, '该产品并未有抢购活动！'); };
    // 判断是否有效
    if (!seckill.is_valid) { return ctx.throwException(30003, '该活动已结束！'); };
    // 判单是否开始、结束
    if(moment().isBefore(moment(seckill.start_time))) {
      return ctx.throwException(30004, '该抢购活动还未开始！');
    }
    if(moment().isAfter(moment(seckill.end_time))) {
      return ctx.throwException(30005, '该抢购活动已经结束！');
    }
    // 判断是否卖完
    if(seckill.amount < 1) { return ctx.throwException(30006, '该产品已经卖完了！'); };

    //获取登录用户信息(这一步只是简单模拟验证用户身份，实际开发中要有严格的登录注册校验流程)
    const userInfo = await UserModule.getUserInfo(accessToken);
    if (!userInfo) { return ctx.throwException(10002, '用户不存在！'); };

    // 判断登录用户是否已抢到
    const orderInfo = await ordersModel.findOne({
      where: {
        user_id: userInfo.id,
        good_id: goodId,
        status: { [Op.between]: ['0', '1'] },
      },
    });
    if (orderInfo) { return ctx.throwException(30007, '该用户已抢到该产品，无需再抢！'); };
    
    // 加锁，实现一个用户针对这次活动只能购买一次
    const lockKey = `${LOCK_KEY}${userInfo.id}:${goodId}`; // 锁的key有用户id和商品id组成
    const uuid = uuidv4();
    const expireTime = moment(seckill.end_time).diff(moment(), 'minutes'); // 锁存在时间为当前时间和活动结束的时间差
    const tryLock = await redis.eval(lock, 2, [lockKey, 'releaseTime', uuid, expireTime]);
    
    try {
      if (tryLock === 1) {
        // 扣库存
        const count = await redis.eval(stock, 2, [key, 'amount', '', '']);
        if (count <= 0) { return ctx.throwException(30006, '该产品已经卖完了！'); };

        // 下单
        const orderData = {
          order_no: Date.now() + randomString(4), // 这里就用当前时间戳加4位随机数作为订单号，实际开发中根据业务规划逻辑 
          good_id: goodId,
          user_id: userInfo.id,
          status: '1', // -1 已取消, 0 未付款， 1 已付款， 2已退款
          order_type: '2', // 1 常规订单 2 秒杀订单
          // seckill_id: seckill.id, // 秒杀活动id, redis中不维护秒杀活动id
          comment: '', // 备注
        };
        const order = ordersModel.create(orderData);

        if (!order) { return ctx.throwException(30008, '抢购失败!'); };
      }
    } catch (e) {
      await redis.eval(unlock, 1, [lockKey, uuid]);
      return ctx.throwException(30006, '该产品已经卖完了！');
    }

    ctx.send({
      path,
      data: '抢购成功!'
    });
  }

}

module.exports = new Seckill();
```

这里代码主要做个四个修改：

1. 步骤2，判断产品是否加入了抢购，改为去`Redis`中查询
2. 步骤7，判断登录用户是否已抢到，因为不在维护抢购活动`id`，所以改为使用用户`id`、产品`id`和状态`status`判断
3. 步骤8，扣库存，改为使用`lua`脚本去`Redis`中扣库存
4. 对扣库存和写入数据库操作进行加锁

订单的操作仍然在`Mysql`数据库中进行，因为大部分的请求都在步骤5被拦截了，剩余请求`Mysql`是完全有能力处理的。   

再次通过`Jmeter`进行测试，发现订单表正常，库存量扣减正常，说明超卖问题和限购已经解决。


## 其他问题
1. 秒杀场景的其他技术
    基于`Redis`支持高并发、键值对型数据库和支持原子操作等特点，案例中使用`Redis`来作为秒杀应对方案。在更复杂的秒杀场景下，除了使用`Redis`外，在必要的的情况下还需要用到其他一些技术：
    
    - 限流，用漏斗算法、令牌桶算法等进行限流
    - 缓存，把热点数据缓存到内存里，尽可能缓解数据库访问的压力
    - 削峰，使用消息队列和缓存技术使瞬间高流量转变成一段时间的平稳流量，比如客户抢购成功后，立即返回响应，然后通过消息队列异步处理后续步骤，发短信，写日志，更新一致性低的数据库等等
    - 异步，假设商家创建一个只针对粉丝的秒杀活动，如果商家的粉丝比较少（假设小于1000），那么秒杀活动直接推送给所有粉丝，如果用户粉丝比较多，程序立刻推送给排名前1000的用户，其余用户采用消息队列延迟推送。（1000这个数字需要根据具体情况决定，比如粉丝数2000以内的商家占99%，只有1%的用户粉丝超过2000，那么这个值就应该设置为2000）
    - 分流，单台服务器不行就上集群，通过负载均衡共同去处理请求，分散压力
  
    这些技术的应用会让整个秒杀系统更加完善，但是核心技术还是`Redis`，可以说用好`Redis`实现的秒杀系统就足以应对大部分场景。
    
2. `Redis`健壮性
    案例使用的是单机版`Redis`，单节点在生产环境基本上不会使用，因为
    - 不能达到高可用
    - 即便有着`AOF`日志和`RDB`快照的解决方案以保证数据不丢失，但都只能放在`master`上，一旦机器故障，服务就无法运行，而且即便采取了相应措施仍不可避免的会造成数据丢失。
    
    因此，`Redis`的主从机制和集群机制在生产环境下是必须的。
3. `Redis`分布式锁的问题
    - 单点分布式锁，案例提到的分布式锁，实际上更准确的说法是单点分布式锁，是为了方便演示，但是，单点`Redis`分布式锁是肯定不能用在生产环境的，理由跟第2点类似
    - 以主从机制（多机器）为基础的分布式锁，也是不够的，因为`redis`在进行主从复制时是异步完成的，比如在`clientA`获取锁后，主`redis`复制数据到从`redis`过程中崩溃了，导致锁没有复制到从`redis`中，然后从`redis`选举出一个升级为主`redis`，造成新的主`redis`没有`clientA`设置的锁，这时`clientB`尝试获取锁，并且能够成功获取锁，导致互斥失效。
    
    针对以上问题，`redis`官方设计了`Redlock`，在`Node.js`环境下对应的资源库为`node-redlock`，可以用`npm`安装，至少需要3个独立的服务器或集群才能使用，提供了非常高的容错率，在生产环境中应该优先采用此方案部署。

## 总结
秒杀场景的特点可以总结为瞬时并发访问、读多写少、限时和限量，开发中还要考虑避免超卖现象以及类似黄牛抢票的限购问题，针对以上特点和问题，分析得到开发的原则是：数据写入内存而不是写入硬盘，异步处理而不是同步处理，扣库存操作原子执行以及对单用户购买进行加锁，而`Redis`正好是符合以上全部特点的工具，因此最终选择`Redis`来解决问题。

秒杀场景是一个在电商业务中相对复杂的场景，此篇文章只是介绍了其中最核心的逻辑，实际业务可能更加复杂，但只需要在此核心基础上进行扩展和优化即可。   
秒杀场景的解决方案不仅仅适合秒杀，类似的还有抢红包、抢优惠券以及抢票等等，思路都是一致的。   
解决方案的思路还可以应用在单独限购、第二件半价以及控制库存等等诸多场景，大家要灵活运用。

## 项目地址

[https://github.com/threerocks/redis-seckill](https://github.com/threerocks/redis-seckill)
## 参考资料
[https://time.geekbang.org/column/article/307421](https://time.geekbang.org/column/article/307421)
[https://redis.io/topics/distlock](https://redis.io/topics/distlock)




