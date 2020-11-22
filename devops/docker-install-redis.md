# Docker安装Redis

### 1. 下载`redis.conf`配置,从[官网下载](https://redis.io/)对应版本的Redis压缩包.

### 2. 创建数据目录,配置文件目录,并把下载的配置文件放到 `$pwd/conf` 下
```
mkdir conf
mkdir data
```

### 3. 运行容器
```bash
docker run -d -p 6379:6379 --restart always --name some-redis \
-v $PWD/conf/redis.conf:/etc/redis/redis.conf \
-v $PWD/data:/data \
redis redis-server /etc/redis/redis.conf \
--requirepass "123456" --appendonly yes
```
* `-d` 后台运行
* `-p 6379:6379` 宿主机和容器的端口映射
* `--restart always`  自动启动
* `--privileged=true` container内的root拥有真正的root权限
* `-v $PWD/conf/redis.conf:/etc/redis/redis.conf`  挂载配置文件
* `-v $PWD/data:/data`  数据目录，冒号左边是宿主机目录，冒号右边是容器内部路径
* `--requirepass` 密码，也可通过`redis.conf`配置
* `--appendonly` 持久化，也可通过`redis.conf`配置

### 4. 修改配置`$pwd/conf/redis.conf`
```
requirepass 123456  #默认空, 连接时需要输入的密码
appendonly yes  # redis持久化（可选）
databases 16    # 数据库个数（可选），可以改改看，看看能不能生效
port 6379   # redis监听的端口号
daemonize no    # 默认no，改为yes意为以守护进程方式启动，可后台运行，除非kill进程，改为yes会使配置文件方式启动redis失败
protected-mode no   # 默认yes，开启保护模式会限制为本地访问
bind 127.0.0.1        # 注释掉这部分，这是限制redis只能本地访问
```
### 5. 重启容器生效配置文件
```
docker restart some-redis
```

## 相关文档
* 配置文件详解：https://www.cnblogs.com/metu/p/9609604.html
* 可视化界面：https://github.com/qishibo/AnotherRedisDesktopManager/releases


