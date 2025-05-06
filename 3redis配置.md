我们在之前介绍过可以通过 redis-server 的启动参数 port 设置了 Redis 服务的端口号，除此之外 Redis 还支持其他配置选项，如是否开启持久化、日志级别等。  
  
通过命令行传递参数  
最简单的方式就是在启动 `redis-server` 的时候直接传递命令参数。  
​
```sh
redis-server \--port 6380 \--host 127.0.0.1
```
  
## 配置文件  
由于可以配置的选项较多，通过启动参数设置这些选项并不方便，所以 Redis 支持通过配置文件来设置这些选项。  
Redis 提供了一个配置文件的模板 redis.conf，位于源代码目录的根目录中。  
  
我们建议把该文件放到 `/etc/redis `目录中（该目录需要手动创建），以端口号命令，例如 `6379.conf`。  
  
启用配置文件的方法是在启动时将配置文件的路径作为启动参数传递给 redis-server：  
​
```sh
redis-server 配置文件路径
```

  
通过启动参数传递同名的配置选项会覆盖配置文件中的相应的参数，就像这样：  

```sh
redis-server 配置文件路径 \--port 3000
```
  
## 在服务器运行时更改 Redis 配置  
还可以在 Redis 运行时通过 CONFIG SET 命令在不重新启动 Redis 的清空下动态修改部分 Redis 配置。就像这样：  

```sh
CONFIG SET logLevel warning
```

  
同样在运行的时候也可以使用 CONFIG GET 命令获得 Redis 当前的配置情况：  

```sh
CONFIG GET logLevel
```


## 使用 CONFIG SET requirepass 设置密码

默认情况下，Redis 不需要密码，这使其容易受到攻击。设置密码是保护你的 Redis 服务器的第一步，也是最重要的一步。我们将使用 `CONFIG SET requirepass` 命令来完成此操作。

**使用 `CONFIG SET requirepass` 命令设置密码：**
`CONFIG SET` 命令允许你动态更改 Redis 的配置设置。`requirepass` 设置指定客户端连接到服务器时必须提供的密码。
```sh
127.0.0.1:6379[15]> CONFIG SET requirepass test123
OK
```

**尝试在没有身份验证的情况下执行命令：**
尝试执行一个简单的命令，如 `PING`：
```sh
127.0.0.1:6379> ping
(error) NOAUTH Authentication required.
```
这表明现在需要身份验证。

## 使用 `AUTH` 命令进行身份验证，及登录

**使用 `AUTH` 命令进行身份验证：**

使用 `AUTH` 命令，后跟你在之前设置的密码：

```bash
127.0.0.1:6379> auth wzh123
ok
```

**在身份验证后执行命令：**

现在你已经通过身份验证，再次尝试 `PING` 命令：

```bash
127.0.0.1:6379> ping
PONG  # 收到预期的响应
```

## 使用 `CONFIG SET` 禁用命令

Redis 提供了许多命令，但有些命令在某些环境中可能有风险。禁用这些命令可以提升安全性。我们将使用 `CONFIG SET disable-command` 来禁用 `FLUSHALL` 命令作为示例。「FLUSHALL」会删除所有数据库中的所有数据，因此禁用它可以防止意外数据丢失。

**使用 `CONFIG SET disable-command` 禁用 `FLUSHALL` 命令：**

```bash
CONFIG SET disable-command FLUSHALL
ok
```

**方法 1：通过配置文件永久禁用**
打开 `redis.conf` 文件。
添加以下配置（例如禁用 `FLUSHDB` 和 `FLUSHALL`）：
```sh
rename-command FLUSHDB ""
rename-command FLUSHALL ""
```
重启 Redis 服务使配置生效：

redis-server redis.conf

**方法 2：通过命令行动态禁用（需 Redis >= 2.6.0）**

```sh
# 动态禁用命令（重启后失效）
CONFIG SET rename-command FLUSHDB ""
CONFIG SET rename-command FLUSHALL ""
```