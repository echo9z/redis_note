在实际开发中经常会遇到一些有时效的数据，比如限时优惠活动、缓存或验证码等，过了一定时间就需要删除这些数据。在关系数据库中一般需要额外的一个字段记录到期时间，然后定期检测删除过期数据。而在 Redis 中可以设置一个键的过期时间，到时间后 Redis 会自动删除它。

**`SET mykey hello EX 10`** 表示：
1. **`SET`**：Redis 的写入命令，用于设置键值对。
2. **`mykey`**：键（Key）的名称。
3. **`hello`**：键对应的值（Value）。
4. **`EX 10`**：设置键的过期时间为 **10 秒**（`EX` 是 "expire" 的缩写，单位为秒）。

```sh
127.0.0.1:6379> set uname tom ex 10
OK
127.0.0.1:6379> get uname
"tom"
127.0.0.1:6379> get uname # 10秒后删除
(nil)
```

 类似用法：
- **`PX`**：以毫秒为单位设置过期时间，例如 `SET mykey hello PX 10000`（10,000 毫秒 = 10 秒）。
- **`NX`**：仅当键不存在时设置值（类似新增操作）。
- **`XX`**：仅当键已存在时设置值（类似更新操作）。
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
# Date.parse('2025/05/09 09:35') js返回时间戳1746754500000
127.0.0.1:6379> expireat name 1746754500000
(integer) 1

# 这个命令和 EXPIREAT 命令类似，但它以毫秒为单位设置 key 的过期 unix 时间戳，而不是像 EXPIREAT 那样，以秒为单位。
PEXPIREAT key milliseconds-timestamp
```

上面这4个命令只是单位和表现形式上的不同，但实际上 `EXPIRE`、`PEXPIRE` 以及 `EXPIREAT` 命令的执行最后都会使用 `PEXPIREAT` 来实行。

比如使用 `EXPIRE` 来设置 KEY 的生存时间为 N 秒，那么后台是如何运行的呢：
- 它会调用 `PEXPIRE` 命令把 N 秒转换为M毫秒
- 然后获取当前的 UNIX 时间单位也是毫秒
- 把当前 UNIX 时间加上 M 毫秒传递给 `PEXPREAT`
另外给键设置了过期时间，这个时间保存在一个字典里，也是键值结构，键是一个指针，指向真实的键，而值这是一个长整型的 UNIX 时间。

## 获取键的过期时间

```sh
# 以秒为单位，返回给定 key 的剩余生存时间(TTL, time to live)。
TTL key
127.0.0.1:6379> zadd score 100 'tom' 58 'jack' 78 'joe'
(integer) 3
127.0.0.1:6379> expire score 60
(integer) 1
127.0.0.1:6379> ttl score
(integer) 51

# 类似于 TTL，但它以毫秒为单位返回 key 的剩余生存时间。
PTTL key
127.0.0.1:6379> pttl score
(integer) 31740

127.0.0.1:6379> pttl score
(integer) -2
```

过期时间返回值说明：

| **值** | **说明**          |
| ----- | --------------- |
| -2    | 过期且已删除          |
| -1    | 没有过期时间设置，即永不过期  |
| >0    | 表示距离过期还有多少秒或者毫秒 |
## 清除键的过期时间
```sh
# 移除给定 key 的生存时间，将这个 key 从『易失的』(带生存时间 key )转换成『持久的』(一个不带生存时间、永不过期的 key )。
PERSIST key

127.0.0.1:6379> expire score 120
(integer) 1
127.0.0.1:6379> persist score
(integer) 1
```

>注意：使用 SET 或 GETSET 命令为键赋值也会同时清除键的过期时间。
>其它只对键值进行操作的命令（如 INCR、LPUSH、HSET、ZREM）不会影响键的过期时间。




