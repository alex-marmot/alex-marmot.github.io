---
layout: post
title: "USing Docker on AWS EC2"
date: 2019-01-21
categories:
---

# 在 AWS 上使用 EC2

因为公司使用 AWS 所以就其上尝试了 Docker，操作过程如下：

1. 创建实例 我选的是 Ubuntu 免费实例

2. `sudo apt update` 刷新下源

3. 安装 docker `sudo apt install docker docker.io`

4. 安装之后 键入 docker 还是会提示没有docker 原因待查， 我是根据提示 `sudo snap install docker` 解决的

5. `sudo usermod -aG docker ${USER}` 将当前用户添加入 docker 组

6. `sudo service docker restart` 重启 docker服务

7. `dockerd` 确定 守护进程是否已经启动

8. 拉取代码 然后 `docker-compose up` 进行测试

9. 这个时候通常执行 `docker-compose down` 会报错，我选择重启。重启大法赛高！！！

10. 重新进入之后就一切 OK。
