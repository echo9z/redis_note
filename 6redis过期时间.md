在实际开发中经常会遇到一些有时效的数据，比如限时优惠活动、缓存或验证码等，过了一定时间就需要删除这些数据。在关系数据库中一般需要额外的一个字段记录到期时间，然后定期检测删除过期数据。而在 Redis 中可以设置一个键的过期时间，到时间后 Redis 会自动删除它。

## 设置键的过期时间

```sh
127.0.0.1:6379> zadd score 100 'tom' 58 'jack' 78 'joe'
(integer) 3
127.0.0.1:6379> zrange score 0 -1 withscores
4) "jack"
5) "58"
6) "joe"
7) "78"
8) "tom"
9) "100"
127.0.0.1:6379> expire score 10 # 10秒钟过期
(integer) 1
127.0.0.1:6379> zrange score 0 -1 withscores
(empty array)

# 和 EXPIRE 一样，但是它以毫秒为单位
PEXPIRE key milliseconds
127.0.0.1:6379> set name tom
OK
127.0.0.1:6379> pexpire name 5000 # 5秒后过期
(integer) 1
127.0.0.1:6379> get name
(nil)

# EXPIREAT 的作用和 EXPIRE 类似，都用于为 key 设置生存时间。
# 不同在于 EXPIREAT 命令接受的时间参数是 UNIX 时间戳(unix timestamp)。
EXPIREAT key timestamp

# 这个命令和 EXPIREAT 命令类似，但它以毫秒为单位设置 key 的过期 unix 时间戳，而不是像 EXPIREAT 那样，以秒为单位。
PEXPIREAT key milliseconds-timestamp
```

上面这4个命令只是单位和表现形式上的不同，但实际上 `EXPIRE`、`PEXPIRE` 以及 `EXPIREAT` 命令的执行最后都会使用 `PEXPIREAT` 来实行。