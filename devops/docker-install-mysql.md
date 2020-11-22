# Docker安装Mysql

### 1. 运行Mysql容器
```bash
docker run -d --name some-mysql mysql:latest 
```

### 2. copy出配置文件
```bash
# 创建挂载目录
mkdir conf
mkdir logs
mkdir data

# 从容器中复制出配置文件
docker cp mysql:/etc/mysql/conf.d /conf
```

### 3. 删除容器
```
docker stop mysql
docker rm mysql
```
### 4. 启动mysql
```bash
docker run -d -p 3306:3306 --restart always --privileged=true --name some-mysql \
-v $PWD/conf:/etc/mysql/conf.d \
-v $PWD/logs:/logs \
-v $PWD/data:/var/lib/mysql \
-v /etc/localtime:/etc/localtime \
-e MYSQL_ROOT_PASSWORD=123456 \
mysql:latest
```
* `-v $PWD/conf:/etc/mysql/conf.d` 挂载配置文件
* `-v $PWD/logs:/logs` 挂载日志文件
* `-v $PWD/data:/var/lib/mysql` 挂载数据文件
* `-v /etc/localtime:/etc/localtime` 容器时间与宿主机同步
* `-e MYSQL_ROOT_PASSWORD=123456` root 账户密码

### windows 命令
```bash
docker run -d -p 3306:3306 --restart always --privileged=true --name some-mysql `
-v $PWD/conf:/etc/mysql/conf.d `
-v $PWD/logs:/logs `
-v $PWD/data:/var/lib/mysql `
-e MYSQL_ROOT_PASSWORD=123456 `
mysql:latest
```