---
title: Docker安装MySQL
category: 软件安装
tag:
  - MySQL
  - Docker
abbrlink: 728095789
date: 2021-07-15 22:00:00
---

{% cq %}
CentOS上使用Docker安装MySQL
{% endcq %}

<!-- more -->

# Docker安装MySQL

## 安装docker

1. 安装 `yum-utils`工具包

   ```
   sudo yum install -y yum-utils device-mapper-persistent-data lvm2
   ```

2. 添加docker的yum源(阿里源)

   ```
   sudo yum-config-manager \
       --add-repo \
       http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
   ```

3. 卸载旧版本(如果存在则删除)

   ```
   sudo yum -y remove docker docker-common docker-selinux docker-engine
   ```

4. 查看所有版本，并选择指定版本安装

   ```
   sudo yum list docker-ce --showduplicates | sort -r
   ```

5. 安装docker

   ```
   sudo yum install docker-ce # 默认安装最新版本
   ```

   - `$ yum install docker-ce-<VERSION_STRING> `(指定安装版本)。例：` yum install docker-ce-18.03.1.ce`

6. 验证安装成功

7. 启动并加入开机启动

   ```
   sudo systemctl start docker       # (重启命令  $  systemctl restart docker ) 
   sudo systemctl enable docker   # 开机自启动
   sudo docker version  # 查看docker版本号
   ```

8. 验证安装成功

   ```
   sudo docker run hello-world
   ```

9. 将当前用户(此时可以使用dev登录)加入到`docker`用户组

   - 新增`docker`用户组，已存在可忽略

     ```
     sudo groupadd docker
     ```

   - 将用户加入`docker`用户组

     ```
     sudo usermod -aG docker ${USER}
     ```

   - 重启`docker`服务

     ```
     sudo systemctl restart docker
     ```

   - 重新登录用户

## 安装MySQL

2. 查看可用的MySQL版本

   [Docker-MySQL版本](https://hub.docker.com/_/mysql?tab=tags)

   

3. 拉取docker镜像（可以直接使用第7步操作进行拉取）

   ```bash
   docker pull mysql:5.7
   # docker pull mysql:{tag}
   # 如 docker pull mysql:5.7
   # docker pull mysql 会默认拉取最新版本
   ```

   

3. 查看本地是否安装了MySQL

   ```bash
   docker images
   ```

   

4. 创建本地数据目录及复制MySQL的配置文件

   ```bash
   mkdir -p /home/dev/data/mysql/data /home/dev/data/mysql/logs /home/dev/data/mysql/conf
   ```

   ```bash
   ├── mysql
   │   ├── conf # 拷贝自MySQL
   │   │   ├── docker.cnf
   │   │   ├── mysql.cnf
   │   │   └── mysqldump.cnf
   │   ├── data
   │   └── logs
   
   ```

   - 配置文件挂载到服务器主要为了修改方便，**配置文件可以通过如下方式获取**.

   ```bash
   # 启动获取配置文件的docker容器
   $ docker run -p 23306:3306 --name mysql_config -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7
   
   # 查看容器是否启动
   $ docker ps
   CONTAINER ID   IMAGE       COMMAND                  CREATED          STATUS          PORTS                                                    NAMES
   e454f36eb8c5   mysql:5.7   "docker-entrypoint.s…"   26 seconds ago   Up 24 seconds   33060/tcp, 0.0.0.0:23306->3306/tcp, :::23306->3306/tcp   mysql_config
   
   # 进入容器，查看配置文件
   $ docker exec -it mysql_config /bin/bash
   $ cd /etc/mysql/conf.d/
   $ ls -rlt
   total 12
   -rw-r--r-- 1 root root 55 Aug  3  2016 mysqldump.cnf
   -rw-r--r-- 1 root root  8 Aug  3  2016 mysql.cnf
   -rw-r--r-- 1 root root 43 Apr 19 18:57 docker.cnf
   
   $ exit # 退出容器
   
   # 拷贝容器中文件到本机， `.` 表示拷贝到当前目录
   $ docker cp mysql_config:/etc/mysql/conf.d/ .
   
   # 查看拷贝结果
   $ ls
   conf.d  data  front  install_package  java  middle
   
   # 将conf.d中文件拷贝到指定的文件夹即可
   $ mv conf.d/* ~/data/mysql/conf
   
   # 删除空文件夹
   $ rm -rf conf.d
   
   # 关闭容器并删除
   $ docker stop mysql_config
   $ docker rm mysql_config
   
   ```

   

5. 启动脚本

   ```bash
   docker run -p 3306:3306 --name mysql -v /home/dev/data/mysql/conf:/etc/mysql/conf.d -v /home/dev/data/mysql/logs:/logs -v /home/dev/data/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7
   ```

   - `-p 主机端口：容器内端口`：将容器内的MySQL使用的3306端口映射到主机的3306端口
   - `-v 主机路径：容器内路径`：将容器内路径映射到主机路径
   - `-e VAR={value}`：指定容器内环境变量的值
   - `-d`：指定容器后台运行