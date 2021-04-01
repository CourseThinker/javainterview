看到这个问题，我想起当初玩魔兽世界的时候，25H难度的脑残吼的血量已经超过了21亿，所以那时候副本的BOSS都设计成了转阶段、回血的模式，因为魔兽的血量是int型，不能超过2^32大小。

估计暴雪的设计师都没想到几个资料片下来血量都超过int上限了，以至于大家猜想才会有后来的属性压缩。

这些都是题外话，只是告诉你数据量大了是有可能达到上限的而已，回到Mysql自增ID上限的问题，可以分为两个方面来说。

### 1.有主键

如果设置了主键，并且一般会把主键设置成自增。

我们知道，Mysql里int类型是4个字节，如果有符号位的话就是[-2^31,2^31-1]，无符号位的话最大值就是2^32-1，也就是4294967295。

创建一张表试试：

```sql
CREATE TABLE `test1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
	`name` varchar(32) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2147483647 DEFAULT CHARSET=utf8mb4;
```

然后执行插入

```sql
insert into test1(name) values('qq');
```

这样表里就有一条达到有符号位的最大值上限的数据。

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gmapn396dqj30ba02s0su.jpg)

如果再次执行插入语句：

```sql
insert into test1(name) values('ww');
```

就会看到错误提示：`1062 - Duplicate entry '2147483647' for key 'PRIMARY', Time: 0.000000s`。

也就是说，如果设置了主键并且自增的话，达到自增主键上限就会报错重复的主键key。

解决方案，mysql主键改为bigint，也就是8个字节。

设计的时候要考虑清楚值的上限是多少，如果业务频繁插入的话，21亿的数字其实还是有可能达到的。

### 2.没有主键

如果没有设置主键的话，InnoDB则会自动帮你创建一个6个字节的row_id，由于row_id是无符号的，所以最大长度是2^48-1。

同样创建一张表作为测试：

```sql
CREATE TABLE `test2` (
	`name` varchar(32) NOT NULL DEFAULT ''
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4;
```

通过`ps -ef|grep mysql`拿到mysql的进程ID，然后执行命令，通过gdb先把row_id修改为1

```sql
sudo gdb -p 2584 -ex 'p dict_sys->row_id=1' -batch
```

然后插入几条数据：

```sql
insert into test2(name) values('1');
insert into test2(name) values('2');
insert into test2(name) values('3');
```

再次修改row_id为2^48，也就是281474976710656

```sql
sudo gdb -p 2584 -ex 'p dict_sys->row_id=281474976710656' -batch
```

再次插入数据

```sql
insert into test2(name) values('4');
insert into test2(name) values('5');
insert into test2(name) values('6');
```

然后查询数据会发现3条数据是4，5，6，3。

因为我们先设置row_id=1开始，所以1，2，3的row_id也是1，2，3。

修改row_id为上限值之后，row_id会从0重新开始计算，所以4，5，6的row_id就是0，1，2。

由于1，2数据已经存在，数据则是会被覆盖。



### 总结

自增ID达到上限用完了之后，分为两种情况：

1. 如果设置了主键，那么将会报错主键冲突。
2. 如果没有设置主键，数据库则会帮我们自动生成一个全局的row_id，新数据会覆盖老数据

解决方案：

表尽可能都要设置主键，主键尽量使用bigint类型，21亿的上限还是有可能达到的，比如魔兽，虽然说row_id上限高达281万亿，但是覆盖数据显然是不可接受的。

