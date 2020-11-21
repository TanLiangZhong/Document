# WSL发行版迁移

> 以迁移docker-desktop-data为示例   
> Windows10 docker 运行数据、镜像文件都存储在$Home\AppData\Local\Docker\wsl\data 需占用C很大空间,C盘不够的小伙伴可将其迁移至指定目录  


1. 关闭所有发行版
    ``` shell
    wsl --shutdown
    ```

2. 导出发行版

    ```shell
    wsl --export docker-desktop-data D:\wsl\docker-desktop-data\docker-desktop-data.tar
    ```
    * `D:\wsl\docker-desktop-data\docker-desktop-data.tar` 导出至的目录，文件目录不能有空格
3. 注销发行版
    ```
    wsl --unregister docker-desktop-data
    ```

4. 重新导入发行版
    ```
    wsl --import docker-desktop-data D:\wsl\docker-desktop-data\ D:\wsl\docker-desktop-data\docker-desktop-data.tar --version 2
    ```
    * `D:\wsl\docker-desktop-data\` 安装位置
    * `D:\wsl\docker-desktop-data\docker-desktop-data.tar` 导出的发行版文件名
