---
title: docker 教程
published: 2025-09-15
description: docker 教程
image: ./images/cover1.png
tags: [docker, 教程]
category: 容器
draft: false
---
## Docker 命令
#### images
```sh
    shell
    docker images -aq
    docker search mysql -f=STARS=3000
    docker pull mysql[:0.8.5]
    docker pull docker.io/library/mysql:laster

    docker rmi -f <IMAGE ID>
    docker rmi -f $(docker images -aq)

    # 导出​​镜像到一个 tar 归档文件。
    docker save -o ubuntu.tar ubuntu:22.04
    docker load -i ubuntu.tar
```
#### container
```sh
    docker run [] image
    --name="my_Nginx_01"
    -d   # 以后台方式运行
    -it  # 交互
    -p 8080:25565 # (主机:容器)
    -v /host/data:/container/data nginx # 挂载数据卷(主机:容器)
    -e "" / --env-file # 环境变量
    -u 1000:1000 
    -m 512m --cpus 1.5
    --rm # ​​退出时自动删除
    --restart [option] # 重启策略​​
    --health-cmd "" --health-interval=5s --health-timeout=3s --health-retries=3
    --platform linux/amd64 ubuntu
    --network my_app_net # 加入自定义网络
    --hostname my_container
    --add-host myhost:192.168.1.100 # myhost 将解析到 192.168.1.100

    # 开启一个新的交互式终端
    dorker exec -it <CONTAINER_ID> /bin/bash
    dorker attach <CONTAINER_ID>

    exit
    Ctrl+P+Q # 退出但不停止

    docker rm -f <CONTAINER_ID>
    docker rm -f $(docker ps -aq)
    docker start/restart/stop/kill <CONTAINER_ID>

    docker cp <CONTAINER_ID>:/home/file ./host_dir/

    # 查看容器日志等
    docker ps
    docker ps -aq | xargs docker rm
    docker port <CONTAINER_ID>
    docker top <CONTAINER_ID>
    docker logs -tf --tail 10 <CONTAINER_ID>
    docker inspect --format='{{.State.Pid}}' <CONTAINER_ID> # JSON格式底层元数据
    docker stats --all
    docker info

    # 镜像构建​
    docker build -t helloworld . # --build-arg 
    docker buildx build -t my_actix_web:v0.1 .
    docker buildx build --platform linux/amd64,linux/arm64 -t my_actix_web:v0.1 .

    # 配置
    docker system df
    docker system prune
    lsof -i :8080
    docker history <CONTAINER_ID>

    sudo vim /etc/docker/daemon.json
    $ cat /etc/docker/daemon.json
    {
        "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn/","https://registry.docker-cn.com"]
    }
    docker info | grep "Registry Mirrors" -A 5
```
#### hub.docker.com
```sh
docker login -u <name> # hub.scutosc.cn 可指定 hub
docker tag mygo:latest alinche/mygo-dev:latest
docker push alinche/mygo-dev:latest

docker builder prune # 仅清理构建缓存
```

#### network
```sh
    docker network ls
    docker network create my_net
    docker run -d --net=my_net --name app1 nginx
    docker run -it --net=my_net alpine ping app1 # (Docker内置DNS)
```


从最高层面来看，Docker 的体系结构主要由以下三个部分组成：
## Docker 拆解
- **Docker 客户端 (Client)**: 这是用户与 Docker 进行交互的主要界面。我们日常使用的 docker 命令，实际上就是 Docker 客户端程序。它接收用户的指令，并将其发送给 Docker 守护进程进行处理。客户端可以通过本地 Unix 套接字或网络与守护进程通信，这意味着你可以在本地机器上通过客户端操作远程服务器上的 Docker。
- **Docker 守护进程 (Daemon / Server)**: Docker 守护进程(`dockerd`)是 Docker 架构的核心。它是一个在后台持续运行的服务，负责接收并处理来自客户端的请求。构建(Build)、运行(Run)、分发(Ship) 容器的所有繁重工作都由它来完成。守护进程管理着 Docker 的所有对象，包括镜像、容器、网络和存储卷。
- **Docker 仓库 (Registry)**: 这是集中存储和分发 Docker 镜像的地方。一个 Docker Registry 可以包含多个仓库(Repository)，每个仓库可以包含多个标签(Tag)的镜像。最著名的公共 Registry 就是 Docker Hub，但用户也可以搭建自己的私有 Registry。当你执行 docker pull ubuntu 或 docker push my-app 命令时，守护进程就是与 Docker Registry 进行通信。

### 守护进程的内部拆解
- **Docker Daemon (dockerd)**: 这是最高层的服务，它暴露了 Docker API，并负责处理来自客户端的请求，同时管理镜像、容器、网络、存储卷等上层对象。
- **containerd**: 这是从 Docker Daemon 中分离出来的一个更底层的容器管理器。dockerd 不再直接负责容器的生命周期管理（如创建、启动、停止），而是将这些任务委托给 containerd。这样做的好处是，即使 dockerd 守护进程崩溃，正在运行的容器也不会受到影响，因为它们的实际管理者是 containerd。containerd 专注于管理容器的完整生命周期：从镜像拉取和推送到存储管理，再到容器的执行和监控。
- **runC**: 这是 containerd 的下一步。containerd 也不直接创建容器，而是使用一个符合 OCI 规范的底层容器运行时（Low-level Runtime）来完成这个任务，runc 就是最常用的一个。runc 是一个轻量级的命令行工具，它的唯一职责就是根据 OCI 规范来创建和运行容器。它负责设置 Linux 内核提供的隔离机制，如命名空间（Namespaces）和控制组（Cgroups），然后运行容器内的进程。创建完成后，runc 进程就会退出，容器进程会成为孤儿进程并被系统接管。
- **shim (containerd-shim)**: runc 退出后，谁来负责收集容器的状态、处理 STDIN/STDOUT 的I/O流呢？这就是 shim 的作用。containerd 会为每个运行的容器启动一个 containerd-shim 进程。这个 shim 进程作为容器进程的父进程，负责报告容器状态给 containerd，并允许在不影响容器进程的情况下重新连接到容器的I/O流。它也是实现 dockerd 或 containerd 重启后仍能管理现有容器的关键。

## 隔离的基石：Linux 命名空间 (Namespaces)

### Pid Namespace —— 独立的进程树
- 进程树: 每个 PID 命名空间都拥有自己独立的进程树，以init进程(PID 1)为根的树形结构，每个PID命名空间都有自己独立的进程树。
- 通过clone()或unshare() 系统调用，添加 CLONE_NEWPID flags，告诉内核为新创建的进程创建一个全新的PIDns
- 进程退出后进程状态变为 EXIT_ZOMBIE ，等待父进程wait()或waitpid()回收资源，
- Linux内核主要采用顺序递增的方式分配PID，维护一个全局计数器其最大值由/proc/sys/kernel/pid_max定义(e.g.32768)，然后回绕，通过一个PID位图(PID bitmap)，
```cpp
// 在内核中，每个进程的 task_struct 结构体都通过 nsproxy 指针指向其所属的命名空间集合，从而实现了进程与特定 PID 命名空间的关联。
struct task_struct {
    pid_t pid;                  // 进程ID
    pid_t tgid;                 // 线程组ID（主线程PID）
    struct task_struct *parent; // 指向父进程，构成了进程树中向上的链接
    struct list_head children;  // 子进程双向链表链表头
    struct list_head sibling;   // 兄弟进程链表节点
    struct nsproxy *nsproxy;    // 命名空间代理（含PID命名空间）
};
```
```sh
# 获取容器在宿主机上的 PID
docker inspect --format '{{.State.Pid}}' my-container
# CONTAINER_PID=$(docker inspect --format '{{.State.Pid}}' my-container)
cat /proc/$CONTAINER_PID/status | grep Pid
pstree -p $CONTAINER_PID
sudo ls -l /proc/$CONTAINER_PID/ns/pid # 或者 sudo readlink /proc/$CONTAINER_PID/ns/pid
    lrwxrwxrwx 1 root root 0 Sep 17 17:10 /proc/10086/ns/pid -> 'pid:[4025532233]' # PID命名空间ID (本质是inode号)
sudo lsns -t pid -p $CONTAINER_PID
            NS TYPE NPROCS   PID USER COMMAND
    4025532233 pid       3 10086 root fwatchdog
# 进入容器的命名空间进行观察
```

### Mount Namespace —— 隔离的文件系统视图
Mount 命名空间为进程提供了一个独立的、隔离的文件系统挂载点视图。这是实现容器拥有自己根文件系统 (/) 的关键。
```cpp
struct path {
    struct vfsmount *mnt;    // 挂载点信息
    struct dentry *dentry;   // 目录项（含文件系统位置）
};
struct fs_struct {
    int users;               // 引用计数
    spinlock_t lock;         // 自旋锁
    struct path root;        // 该进程认为的根目录 "/"
    struct path pwd;         // 该进程的当前工作目录 "."
};
struct task_struct {
    struct fs_struct *fs;    // 指向进程的文件系统视图
    struct nsproxy *nsproxy; // 指向进程所属的命名空间集合
    ...
};
struct nsproxy {
    struct mnt_namespace *mnt_ns; // 指向所属的 Mount Namespace
    // ... 指向其他类型的命名空间 (uts, ipc, net, pid, ...)
};
struct mnt_namespace {
    struct vfsmount *root;   // 该命名空间的根挂载点
    // ... 其他信息
};
```
- chroot("/new_root") 进程级操作：仅修改了 `current->fs->root`，完全没有触碰 `current->nsproxy->mnt_ns`
- pivot_root("new_root", "put_old") 命名空间级操作：直接作用于Mount Namespace，同时修改`current->fs->root` 和 `current->nsproxy->mnt_ns->root`
```sh
# 查看容器进程的根目录链接到了哪里
sudo ls -l /proc/$CONTAINER_PID/root
    lrwxrwxrwx 1 root root 0 Sep 17 17:08 /proc/10086/root -> /home/username/MyDocker/my-rootfs
```
```sh
# 通过 nsenter 以容器的文件系统视角执行命令
sudo nsenter -t 25615 -m ls
sudo nsenter -t 25615 -m ps -ef
sudo nsenter -t 25615 -m top
```

### UTS (Unix Time-sharing System) Namespace —— 隔离主机名和域名
UTS 命名空间的核心且唯一的功能就是：隔离 hostname (主机名) 和 domainname (NIS 域名)
- 身份标识: 主机名是网络中一台计算机最基本的身份标识。对于容器来说，拥有一个独立的主机名，而不是继承宿主机的名字（，对于服务发现、日志记录、配置管理和网络识别都至关重要。它让容器在逻辑上看起来更像一台独立的机器。
- 配置隔离: 许多应用程序和服务在启动时会读取系统的主机名来生成默认配置、注册自己或进行其他初始化操作。UTS 命名空间确保了容器内的应用获取到的是容器专属的主机名，从而避免了配置混乱。
- 满足应用期望: 一些软件被设计为在独立的主机上运行。UTS 命名空间满足了这些软件的运行环境期望，使得它们可以无缝地被容器化。

### User Namespace —— 隔离的用户与权限
隔离用户和组ID + 隔离权能 (Capabilities)
- 每个进程的cred结构体都明确地指向它所属的 User 命名空间。当一个进程尝试执行任何需要权限检查的操作时，内核会查看它的cred，并特别是它所属的user_ns，来决定这个操作是否被允许。
```cpp
// include/linux/user_namespace.h
struct user_namespace {
    struct uid_gid_map uid_map;    // UID 映射规则
    struct uid_gid_map gid_map;    // GID 映射规则
    struct user_namespace *parent; // 父命名空间
    unsigned int level;            // 命名空间层级深度
    kuid_t owner;                  // 创建者用户ID
    kgid_t group;                  // 创建者组ID
    struct ns_common ns;           // 命名空间公共结构
    struct work_struct work;       // 异步工作队列
    struct ucounts *ucounts;       // 资源计数器
    // ... 其他字段（能力集、进程限制等）
};
```
- docker 的 默认行为其实是不做 User Namespace 映射的（）
```sh
 ~/> sudo ls -l /proc/10086/ns/
total 0
lrwxrwxrwx 1 root root 0 Sep 18 15:55 cgroup -> 'cgroup:[4026532361]' # new
lrwxrwxrwx 1 root root 0 Sep 18 15:55 ipc -> 'ipc:[4026532359]' # new
lrwxrwxrwx 1 root root 0 Sep 18 15:55 mnt -> 'mnt:[4026532357]' # new
lrwxrwxrwx 1 root root 0 Sep 18 15:55 net -> 'net:[4026532362]' # new
lrwxrwxrwx 1 root root 0 Sep 18 15:55 pid -> 'pid:[4026532360]' # new
lrwxrwxrwx 1 root root 0 Sep 18 15:56 pid_for_children -> 'pid:[4026532360]' # new
lrwxrwxrwx 1 root root 0 Sep 18 15:56 time -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Sep 18 15:56 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Sep 18 15:56 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Sep 18 15:55 uts -> 'uts:[4026532358]' # new
 ~/> sudo ls -l /proc/1/ns/
total 0
lrwxrwxrwx 1 root root 0 Sep 18 15:16 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Sep 18 15:16 ipc -> 'ipc:[4026532206]'
lrwxrwxrwx 1 root root 0 Sep 18 15:16 mnt -> 'mnt:[4026532217]'
lrwxrwxrwx 1 root root 0 Sep 18 15:16 net -> 'net:[4026531840]'
lrwxrwxrwx 1 root root 0 Sep 18 15:16 pid -> 'pid:[4026532219]'
lrwxrwxrwx 1 root root 0 Sep 18 15:16 pid_for_children -> 'pid:[4026532219]'
lrwxrwxrwx 1 root root 0 Sep 18 15:16 time -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Sep 18 15:16 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Sep 18 15:16 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Sep 18 15:16 uts -> 'uts:[4026532218]'
# P.S. docker 的 默认行为是不做 User Namespace 映射的（）
# 对开启 User Namespace 的容器，当容器内的root进程（在宿主机上其实是UID 1001）尝试写入 /host/data 时，它会因为权限不足而被拒绝 (Permission Denied)，因为它不是文件所有者。
# 为了解决这个问题，Docker Daemon 必须采取一个非常暴力的措施：它会自动递归 chown 挂载点目录，将其在宿主机上的所有权递归地修改为那个映射后的高编号 UID (1001)
# 1. 性能雪崩: 对于包含大量文件的数据卷，执行 chown -R 是一个极其缓慢的操作，它会大大延长容器的启动时间。
# 2. 破坏宿主机权限: 原本属于你普通用户（UID 1000）的 /host/data 目录，现在被改成了 UID 1001 用户所有。你自己在宿主机上无法访问这些文件了！ 这是一个非常糟糕且出乎意料的副作用，对于开发者来说是不可接受的。 一些特定的用户才能访问的硬件设备（如USB），在启用了 User Namespace 后，容器内的非特权用户可能无法获得访问这些设备的权限。
# 4. 性能开销: 每次文件操作需查询映射表 `/proc/<pid>/uid_map` ，进行一次 UID/GID 的转换

## 结论就是：Docker 将 “开箱即用的易用性” 和 “最大的向后兼容性” 放在了比 “默认的最高安全性” 更高的优先级上。
```

### Network Namespace
```sh
sudo nsenter -t 25615 -n ip addr
sudo nsenter -t 25615 -m -n ping 8.8.8.8 -c 1 # 没有 -n 将会使用宿主机的网络命名空间，这一点可以通过 tshark 证明
```

## Q & A:
- Q1: 为什么我不能通过docker隔离
  - A1: 驱动通过硬件描述信息初始化之后就是一个内存中的程序，感觉可以做隔离，但别忘了内核是共享的，这个内存其实是内核空间中的内存，常见的驱动是一种​​内核模块​​。docker 本质是 **​​共享单一宿主机内核**​ ​的进程隔离环境。设备驱动是暴露read()write()ioctl()等接口供容器使用的，容器就一个用户空间的沙盒盒，没有驱动这一说。所以哪怕是--privileged，insmod也是到宿主机上，容器什么都没有