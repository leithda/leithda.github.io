---
title: docker_镜像导入导出
category: docker
tag: docker
abbrlink: 2652267918
date: 2020-01-01 12:00:00
---

{% cq %}

当目标服务器无网络或希望从一台服务器拷贝docker镜像到目标服务器时。可以使用docker的命令完成镜像的导入导出。

{% endcq %}

<!-- more -->

# docker_镜像导入导出

## 导出

### 查看镜像列表

```bash
# dev @ localhost in ~ [10:03:04] 
$ docker images                           
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
mysql         5.7       09361feeb475   2 weeks ago    447MB
hello-world   latest    d1165f221234   4 months ago   13.3kB
```



### 导出镜像到文件

```bash
# dev @ localhost in ~ [10:03:10]
$ docker save -o ~/mysql_5.7.tar mysql:5.7
```

- `docker save -o {FILE_NAME} {REPOSITORY:TAG}`





## 导入

将导出的镜像文件上传至服务器，使用命令`docker load < {FILE_NAME}`加载

```bash
# 查看镜像
# dev @ localhost in ~ [10:08:14] 
$ docker images
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
hello-world   latest    d1165f221234   4 months ago   13.3kB

# 导入镜像
# dev @ localhost in ~ [10:08:50] C:1
$ docker load < mysql_5.7.tar 
764055ebc9a7: Loading layer [==================================================>]  72.53MB/72.53MB
71a14cc55692: Loading layer [==================================================>]  338.4kB/338.4kB
50854886015e: Loading layer [==================================================>]  9.557MB/9.557MB
1952fb2b0eb4: Loading layer [==================================================>]  4.202MB/4.202MB
893f6aea2ce2: Loading layer [==================================================>]  2.048kB/2.048kB
b8d0aeaeeee8: Loading layer [==================================================>]  53.77MB/53.77MB
d7cde20f3f68: Loading layer [==================================================>]  5.632kB/5.632kB
12c8996d19a8: Loading layer [==================================================>]  3.584kB/3.584kB
8b092d2f4bcf: Loading layer [==================================================>]  311.9MB/311.9MB
4f20a66508d4: Loading layer [==================================================>]  17.92kB/17.92kB
4723a691b7d9: Loading layer [==================================================>]  1.536kB/1.536kB
Loaded image: mysql:5.7

# 查看镜像
$ docker images
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
mysql         5.7       09361feeb475   2 weeks ago    447MB
hello-world   latest    d1165f221234   4 months ago   13.3kB

```

- 导入成功

