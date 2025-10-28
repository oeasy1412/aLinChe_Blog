---
title: epoll
published: 2025-10-28
description: 现代网络编程基石：深入解析 I/O 多路复用及 select、poll、epoll
tags: [OS, net]
category: OS
draft: false
---

### **现代网络编程基石：深入解析 I/O 多路复用及 select、poll、epoll**
在高性能网络编程领域，如何高效地处理成千上万的并发连接是一个核心挑战。传统的阻塞 I/O 模型（Blocking I/O）一个线程只能处理一个连接，当连接数增多时，为每个连接创建一个线程或进程会消耗大量的系统资源，导致上下文切换开销巨大，严重影响服务器性能。为了解决这个问题，I/O 多路复用（`I/O Multiplexing`）技术应运而生，它成为了构建高并发网络服务的基石。

本文将深入探讨 I/O 多路复用的核心思想，并详细解析三种主流的实现方式：`select`、`poll` 和 `epoll`，并结合代码示例，帮助你理解它们的工作原理、优缺点以及适用场景。

#### **一、 什么是 I/O 多路复用？**
I/O 多路复用的核心思想是：**用一个或少数几个线程来监视多个文件描述符(File Descriptor, FD)**，一旦某个或某些文件描述符就绪（例如，有数据可读或可写），就通知应用程序进行相应的 I/O 操作。

这种模型的优势在于，单个线程可以同时管理大量的连接，线程并不需要阻塞在某个具体的 I/O 操作上，而是在一个集中的点（如 `select`、`poll`、`epoll_wait`）上等待，直到有事件发生。这样极大地减少了系统开销，提高了服务器的并发处理能力。

#### **二、 经典模型：`select`**
`select` 是最早出现的 I/O 多路复用模型，也是最经典的一种。它的基本工作模式是，程序员需要维护一个文件描述符集合，然后调用 `select` 函数来监听这个集合中所有 FD 的状态。

- **工作流程解读:**
`select` 代码示例：
```cpp
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <netinet/in.h>
#include <sys/select.h>
#include <sys/socket.h>
#include <unistd.h>

#define MAXBUF      1024
#define MAX_CLIENTS 5

int main() {
    int sockfd;
    struct sockaddr_in server_addr, client_addr;
    socklen_t addrlen;
    char buffer[MAXBUF];
    int fds[MAX_CLIENTS], max_fd;
    memset(&fds, -1, sizeof(fds)); // -1表示该位置未使用
    // 创建TCP套接字
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        perror("socket创建失败");
        exit(1);
    }
    // 设置套接字选项，避免地址占用错误
    int opt = 1;
    if (setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)) < 0) {
        perror("setsockopt失败");
        close(sockfd);
        exit(EXIT_FAILURE);
    }
    // 设置服务器地址
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(8000);
    server_addr.sin_addr.s_addr = INADDR_ANY;
    // 绑定套接字到地址并监听
    if (bind(sockfd, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("绑定失败");
        close(sockfd);
        exit(1);
    }
    if (listen(sockfd, 5) < 0) {
        perror("监听失败");
        close(sockfd);
        exit(1);
    }
    printf("基于select的服务器启动，监听端口8000...\n");

    // 初始化客户端文件描述符数组
    max_fd = sockfd;
    int i;
    fd_set rset; // 关键数据 bitmap
    while (true) {
        FD_ZERO(&rset);        // 每次select都需要手动清空rset
        FD_SET(sockfd, &rset); // 监听服务器套接字
        // 将所有的客户端套接字加入监听集合rset
        for (i = 0; i < MAX_CLIENTS; ++i) {
            if (fds[i] > 0) {
                FD_SET(fds[i], &rset);
                if (fds[i] > max_fd) {
                    max_fd = fds[i];
                }
            }
        }
        printf("📡等待客户端连接或数据...\n");
        // 使用select监听多个文件描述符，我们只关心从 0 到 max_fd 这个范围内的所有文件描述符
        int ready = select(max_fd + 1, &rset, NULL, NULL, NULL);
        if (ready < 0) {
            perror("select错误");
            continue;
        }
        // 检查是否有新的客户端连接
        if (FD_ISSET(sockfd, &rset)) {
            memset(&client_addr, 0, sizeof(client_addr));
            addrlen = sizeof(client_addr);
            int new_client = accept(sockfd, (struct sockaddr*)&client_addr, &addrlen); // 返回套接字文件描述符
            if (new_client >= 0) {
                printf("✅新客户端连接, fd: %d\n", new_client);
                // 将新客户端加入到空闲位置
                bool added = false;
                for (i = 0; i < MAX_CLIENTS; ++i) {
                    if (fds[i] < 0) {
                        fds[i] = new_client;
                        added = true;
                        break;
                    }
                }
                if (!added) {
                    printf("❌客户端数量已达上限，拒绝连接\n");
                    close(new_client);
                }
            }
        }
        // 检查客户端是否有数据可读
        for (i = 0; i < MAX_CLIENTS; ++i) {
            if (fds[i] > 0 && FD_ISSET(fds[i], &rset)) {
                memset(buffer, 0, MAXBUF);
                ssize_t bytes_read = read(fds[i], buffer, MAXBUF - 1);
                if (bytes_read > 0) {
                    printf("📨从客户端%d接收: %s", fds[i], buffer);
                    // 回声功能：将数据发回客户端
                    write(fds[i], buffer, bytes_read);
                } else {
                    // 客户端断开连接
                    printf("🔌客户端%d断开连接\n", fds[i]);
                    close(fds[i]);
                    // 若关闭的是最大的fd，则重新计算max_fd，减少监听性能开销
                    if (fds[i] == max_fd) {
                        int new_max = sockfd;
                        for (int fd : fds)
                            if (fd > new_max)
                                new_max = fd;
                        max_fd = new_max;
                    }
                    fds[i] = -1; // 标记为可用
                }
            }
        }
    }
    close(sockfd);
    return 0;
}
```
```sh
> curl localhost:8000 # 当然你也可以直接浏览器访问

基于select的服务器启动，监听端口8000...
📡等待客户端连接或数据...
✅新客户端连接, fd: 4
📡等待客户端连接或数据...
📨从客户端4接收: GET / HTTP/1.1
Host: localhost:8000
User-Agent: curl/7.81.0
Accept: */*

📡等待客户端连接或数据...
🔌客户端4断开连接
📡等待客户端连接或数据...

```

1.  **创建和初始化：**
    *   首先，程序通过 `socket`、`bind`、`listen` 创建一个监听套接字 `sockfd`。
    *   然后通过循环 `accept` 客户端连接，并将接受到的新连接的 FD 存储在一个 `fds` 数组中。
2.  **核心循环与监听：**
    *   在一个 `while(true)` 循环中，首先使用 `FD_ZERO(&rset)` 清空一个名为 `rset` 的文件描述符集合。
    *   接着，通过 `for` 循环和 `FD_SET(fds[i], &rset)` 将所有需要监听的 FD 添加到 `rset` 集合中。
    *   调用 `select(max+1, &rset, NULL, NULL, NULL)` 函数。这是 `select` 模型的核心，该函数会**阻塞**，直到 `rset` 集合中至少有一个 FD 变为可读。`max+1` 是所有被监听的 FD 的最大值+1。
3.  **处理就绪事件：**
    *   当 `select` 返回后（即有事件发生），程序再次通过一个 `for` 循环和 `FD_ISSET(fds[i], &rset)` 来**遍历所有**的 FD，检查哪一个 FD 是真正就绪的。
    *   对于就绪的 FD，执行 `read` 操作读取数据。

**`select` 的优缺点:**
*   **优点:**
    *   **跨平台性好**：几乎所有的主流操作系统都支持 `select`，兼容性极佳。
*   **缺点:**
    *   **最大连接数限制**：`select` 使用一个位图（bitmap）来存储文件描述符，这个位图的大小通常由 `FD_SETSIZE` 宏定义，在大多数系统上是 1024。这意味着单个进程默认最多只能监听 1024 个 FD。
    *   **两次拷贝性能开销大**：每次调用 `select` 之前，都需要将所有fd_set（本质是一个 bitmap）要监听的 FD 从用户空间**拷贝**到内核空间。当连接数很多时，这个拷贝开销非常大。
    *   **两次线性扫描**：进入内核态后，内核需要遍历传入的所有 FD，逐一检查它们的状态，如果某个 FD 就绪，内核就会在它拷贝过来的那份 fd_set 的对应位上进行置位操作。`select` 返回后，只能知道有 FD 就绪了，但不知道是哪几个。因此，程序需要**遍历**整个 FD 集合来找到就绪的 FD，这个时间复杂度是 O(n)，当连接数巨大而活跃连接数很少时，效率很低。

#### **三、 `select` 的改进版：`poll`**
`poll` 的出现主要是为了解决 `select` 的一些限制，特别是最大连接数的限制。

- **工作流程解读:**
`poll` 代码示例：
```cpp
#include <arpa/inet.h>
#include <netinet/in.h>
#include <poll.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>

#define MAXBUF      1024
#define MAX_CLIENTS 5

int main() {
    int sockfd;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_len;
    char buffer[MAXBUF];
    // 创建pollfd结构体数组
    struct pollfd fds[MAX_CLIENTS + 1]; // +1 给服务器套接字
    int nfds = 1;                       // 当前监控的文件描述符数量（初始只有服务器）
    for (int i = 0; i <= MAX_CLIENTS; ++i) {
        fds[i].fd = -1;         // -1表示该位置未使用
        fds[i].events = POLLIN; // 监控可读事件
        fds[i].revents = 0;     // 返回的事件
    }
    // 创建TCP套接字
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        perror("socket创建失败");
        exit(EXIT_FAILURE);
    }
    // 设置套接字选项，避免地址占用错误
    int opt = 1;
    if (setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)) < 0) {
        perror("setsockopt失败");
        close(sockfd);
        exit(EXIT_FAILURE);
    }
    // 设置服务器地址
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(8000);
    // 绑定套接字到地址并监听
    if (bind(sockfd, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("绑定失败");
        close(sockfd);
        exit(EXIT_FAILURE);
    }
    if (listen(sockfd, 5) < 0) {
        perror("监听失败");
        close(sockfd);
        exit(EXIT_FAILURE);
    }
    printf("基于poll的服务器启动，监听端口8000...\n");
    printf("📡等待客户端连接...\n");

    // 将服务器套接字添加到poll监控
    fds[0].fd = sockfd;
    fds[0].events = POLLIN;
    int i;
    while (true) {
        // 调用poll等待事件发生
        int ready = poll(fds, nfds, 10000); // 10s
        if (ready < 0) {
            perror("poll错误");
            break;
        } else if (ready == 0) {
            printf("poll等待超时，继续监听...\n");
            continue;
        }
        // 检查所有被监控的文件描述符
        int cur_nfds = nfds;
        for (i = 0; i < cur_nfds; ++i) {
            if (fds[i].fd == -1)
                continue;
            // 检查是否有事件发生
            if (fds[i].revents & POLLIN) {
                if (fds[i].fd == sockfd) {
                    client_len = sizeof(client_addr);
                    int new_client = accept(sockfd, (struct sockaddr*)&client_addr, &client_len);
                    if (new_client < 0) {
                        perror("接受连接失败");
                    } else if (nfds <= MAX_CLIENTS) {
                        // 将新客户端添加到poll监控
                        fds[nfds].fd = new_client;
                        fds[nfds].events = POLLIN;
                        fds[nfds].revents = 0;
                        char client_ip[INET_ADDRSTRLEN];
                        inet_ntop(AF_INET, &client_addr.sin_addr, client_ip, INET_ADDRSTRLEN);
                        printf("✅新客户端连接: fd=%d, IP=%s:%d\n", new_client, client_ip, ntohs(client_addr.sin_port));
                        printf("当前连接数: %d/%d\n", nfds, MAX_CLIENTS);
                        ++nfds;
                    } else {
                        printf("客户端数量已达上限，拒绝连接\n");
                        close(new_client);
                    }
                } else {
                    // 客户端套接字有数据可读
                    int client_fd = fds[i].fd;
                    memset(buffer, 0, MAXBUF);
                    ssize_t bytes_read = read(client_fd, buffer, MAXBUF - 1);
                    if (bytes_read > 0) {
                        buffer[strcspn(buffer, "\r\n")] = 0;
                        printf("📨从客户端%d接收: %s\n", client_fd, buffer);
                        // 回声功能
                        if (write(client_fd, buffer, bytes_read) < 0) {
                            perror("发送数据失败");
                        }
                    } else {
                        // 客户端断开连接或错误
                        if (bytes_read == 0) {
                            printf("🔌客户端%d断开连接\n", client_fd);
                        } else {
                            perror("读取数据错误");
                        }
                        close(client_fd);
                        fds[i].fd = -1;
                        // （将后面的元素前移）
                        for (int j = i; j < nfds - 1; ++j) {
                            fds[j] = fds[j + 1];
                        }
                        --nfds;
                        fds[nfds].fd = -1;
                        --i; // 重新检查当前位置
                    }
                }
            }
            // 检查其他事件（如错误）
            if (fds[i].fd != -1 && (fds[i].revents & (POLLERR | POLLHUP))) {
                printf("❌客户端%d发生错误或挂断\n", fds[i].fd);
                close(fds[i].fd);
                fds[i].fd = -1;
            }
        }
    }
    // 清理资源
    for (int i = 0; i < nfds; ++i) {
        if (fds[i].fd != -1)
            close(fds[i].fd);
    }
    close(sockfd);
    return 0;
}
```

1.  **数据结构变化：**
    *   `poll` 不再使用位图，而是定义了一个 `struct pollfd` 数组。这个结构体包含了 `fd`（文件描述符）、`events`（要监听的事件，如 `POLLIN` 表示可读）和 `revents`（实际发生的事件）。
    *   由于使用了数组而非位图，`poll` **没有了 1024 的连接数限制**，其最大连接数受限于系统的内存大小。
2.  **核心循环与监听：**
    *   在 `accept` 连接后，将新的 FD 和关心的事件 `POLLIN` 填入 `pollfds` 数组。
    *   调用 `poll(pollfds, 5, 50000)` 函数进行阻塞监听。
3.  **处理就绪事件：**
    *   当 `poll` 返回后，同样需要**遍历**整个 `pollfds` 数组。
    *   通过检查 `pollfds[i].revents & POLLIN` 来判断对应的 FD 是否就绪。

**`poll` 的优缺点:**
*   **优点:**
    *   **解决了最大连接数限制**：它突破了 `select` 的 1024 限制。
*   **缺点:**
    *   **性能开销问题依然存在**：和 `select` 一样，每次调用 `poll` 仍然需要将整个 `pollfds` 数组从用户空间**拷贝**到内核空间。
    *   **线性扫描问题依然存在**：`poll` 返回后，依然需要**遍历**整个数组来找到就绪的 FD，时间复杂度仍为 O(n)。

总的来说，`poll` 只是对 `select` 的一个简单改进，并没有从根本上解决性能开销和线性扫描的问题。

#### **四、 高性能的终极选择：`epoll`**
`epoll` 是 Linux 内核为了解决 `select` 和 `poll` 的性能瓶颈而提出的一套全新的 I/O 多路复用机制，是目前公认的 Linux 平台下性能最好的方案。Redis、Nginx 等著名的高性能中间件都广泛采用了 `epoll`。


- **`epoll` 的核心改进:**
`epoll` 的设计哲学与 `select`/`poll` 完全不同。它将监听的 FD 集合的管理和事件的等待分离开来。
1.  **`epoll_create()`**: 创建一个 `epoll` 实例。在内核中，这会创建一个 `eventpoll` 对象，该对象内部包含两个关键数据结构：
    *   **红黑树(Red-Black Tree)**: 用于高效地存储和管理所有被监听的文件描述符。
    *   **就绪列表(Ready List)**: 用于存放已经就绪（有事件发生）的文件描述符。

2.  **`epoll_ctl()`**: 对 `epoll` 实例进行管理，可以**添加**（`EPOLL_CTL_ADD`）、**修改**（`EPOLL_CTL_MOD`）或**删除**（`EPOLL_CTL_DEL`）被监听的 FD。当添加一个新的 FD 时，内核会创建一个 `epitem` 结构体来代表这个监控项，并将需要这个epitem插入到红黑树rbr中进行管理。这个FD信息只需要在此时从用户态拷贝到内核**一次**，之后除非被删除，否则会一直存在于内核的红黑树中，无需反复拷贝。同时，它会向这个FD对应的内核文件结构一个​**​回调函数**（通常是`ep_poll_callback`），注册在内核中与该 socket 事件相关的等待队列 (wait queue) 上，事件就绪时，内核通过这个回调函数来采取行动。

3.  **`epoll_wait()`**: 阻塞等待就绪列表中的事件。（当一个 FD 接收到数据而就绪时，内核会执行预先注册好的回调函数 `ep_poll_callback`，这个函数会将对应的 epitem 添加到 eventpoll 对象的就绪列表 rdllist 中。）与 `select`/`poll` 不同，`epoll_wait()` 只需检查就绪列表是否为空，若不为空，则将就绪列表中的 `epitem` 所携带的事件信息**从内核空间拷贝到用户空间**，然后返回就绪事件的数量，而不需要去遍历整个集合，从而实现了 `O(1)` 的时间复杂度（严格来说是 O(k)，k为就绪FD数量）。

- **底层结构体**
```cpp
// epoll_create() 内核会在内存中创建了一个 eventpoll 结构体实例
struct eventpoll {
	struct mutex mtx;
	wait_queue_head_t wq;        // 等待队列​​：让调用epoll_wait()的进程在此排队休眠。
	wait_queue_head_t poll_wait; // poll等待队列​​：当eventpoll文件描述符本身被监控时（例如被另一个epoll监控）使用的等待队列
	struct list_head rdllist;    // “就绪队列”（双向链表）：epoll_wait()只需检查这个链表是否为空，即可知道有多少事件就绪并拷贝到用户空间，复杂度为O(1)
	rwlock_t lock;               // 主要用于保护就绪链表rdllist
	struct rb_root_cached rbr;   // 红黑树的根节点：存储所有通过epoll_ctl()注册的待监控事件
	struct epitem *ovflist;      // 临时中转清单​​：在将事件从内核向用户空间传输时，作为一个临时链表使用，以避免在持有mtx锁的情况下操作rdllist，从而优化性能
	struct file *file;           // 关联文件​​：指向内核中代表这个epoll实例的文件对象。
};

struct epitem {
	union {
		struct rb_node rbn;       // 红黑树节点
		struct rcu_head rcu;
	};
	struct list_head rdllink;     // 就绪链表节点
	struct epitem *next;
	struct epoll_filefd ffd;      // 存储了该 epitem对应的文件描述符信息，是红黑树中查找和比较的关键依据。
	struct eppoll_entry *pwqlist; // 指向一个列表，该列表维护了与这个 fd 相关的等待队列条目。这是实现 epoll ​​回调机制​​的关键，确保事件发生时能准确通知到对应的 epoll 实例
	struct eventpoll *ep;         // 指向该 epitem所属的 eventpoll实例
	struct hlist_node fllink;     // 反向链接​​：用于将本 epitem链接到被监视的 struct file对象中的一个列表。这样，当文件被关闭时，内核可以遍历此列表，清理所有相关的 epoll 监视项
	struct epoll_event event;     // 监视清单​​：保存了用户通过 epoll_ctl注册的感兴趣事件
};

struct epoll_event {
	__poll_t events;
	__u64 data;
} EPOLL_PACKED;
```

- **工作流程解读:**
`epoll` 示例代码：
```cpp
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <netinet/in.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <unistd.h>

#define MAXBUF     1024
#define MAX_EVENTS 10

int main() {
    int sockfd;
    struct sockaddr_in server_addr, client_addr;
    socklen_t addrlen;
    char buffer[MAXBUF];
    struct epoll_event ev, events[MAX_EVENTS];
    int epfd, nfds;
    // 创建TCP套接字
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        perror("socket创建失败");
        exit(1);
    }
    // 设置套接字选项，避免地址占用错误
    int opt = 1;
    if (setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)) < 0) {
        perror("setsockopt失败");
        close(sockfd);
        exit(EXIT_FAILURE);
    }
    // 设置服务器地址
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(8000);
    server_addr.sin_addr.s_addr = INADDR_ANY;
    // 绑定套接字到地址并监听
    if (bind(sockfd, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("绑定失败");
        close(sockfd);
        exit(1);
    }
    if (listen(sockfd, 5) < 0) {
        perror("监听失败");
        close(sockfd);
        exit(1);
    }
    printf("基于epoll的服务器启动，监听端口8000...\n");

    // 创建epoll实例
    epfd = epoll_create1(0);
    if (epfd < 0) {
        perror("epoll创建失败");
        close(sockfd);
        exit(1);
    }
    // 将服务器套接字添加到epoll监听
    ev.events = EPOLLIN;
    ev.data.fd = sockfd;
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev) < 0) {
        perror("epoll_ctl添加服务器套接字失败");
        close(epfd);
        close(sockfd);
        exit(1);
    }

    int i;
    while (true) {
        printf("📡等待事件发生...\n");
        // 等待事件发生，超时时间10秒
        nfds = epoll_wait(epfd, events, MAX_EVENTS, 10000);
        if (nfds < 0) {
            perror("epoll_wait错误");
            continue;
        } else if (nfds == 0) {
            printf("epoll等待超时，继续监听...\n");
            continue;
        }

        // 处理所有就绪的事件
        for (i = 0; i < nfds; ++i) {
            // 新的客户端连接
            if (events[i].data.fd == sockfd) {
                memset(&client_addr, 0, sizeof(client_addr));
                addrlen = sizeof(client_addr);
                int client_fd = accept(sockfd, (struct sockaddr*)&client_addr, &addrlen);
                if (client_fd >= 0) {
                    printf("✅新客户端连接: %d\n", client_fd);
                    // 设置新客户端为非阻塞模式（可选）
                    ev.events = EPOLLIN | EPOLLET; // 边缘触发模式
                    ev.data.fd = client_fd;
                    if (epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev) < 0) {
                        perror("epoll_ctl添加客户端失败");
                        close(client_fd);
                    }
                }
            } else {
                // 客户端套接字有数据可读
                int client_fd = events[i].data.fd;
                memset(buffer, 0, MAXBUF);
                ssize_t bytes_read = read(client_fd, buffer, MAXBUF - 1);
                if (bytes_read > 0) {
                    printf("📨从客户端%d接收: %s", client_fd, buffer);
                    // 回声功能
                    write(client_fd, buffer, bytes_read);
                } else {
                    // 客户端断开连接
                    printf("🔌客户端%d断开连接\n", client_fd);
                    epoll_ctl(epfd, EPOLL_CTL_DEL, client_fd, NULL);
                    close(client_fd);
                }
            }
        }
    }
    close(epfd);
    close(sockfd);
    return 0;
}
```
```sh
> curl localhost:8000 # 当然你也可以直接浏览器访问

基于epoll的服务器启动，监听端口8000...
📡等待事件发生...
✅新客户端连接: 5
📡等待事件发生...
📨从客户端5接收: GET / HTTP/1.1
Host: localhost:8000
User-Agent: curl/7.81.0
Accept: */*

📡等待事件发生...
🔌客户端5断开连接
📡等待事件发生...

```

1.  **初始化：**
    *   `epfd = epoll_create(10)` 创建一个 `epoll` 实例。
    *   在 `accept()` 循环中，每当有新连接到来，就创建一个 `epoll_event` 结构体，设置好 `fd` 和关心的事件 `EPOLLIN`。
    *   通过 `epoll_ctl(epfd, EPOLL_CTL_ADD, ev.data.fd, &ev)` 将新的连接 FD **注册**到内核的红黑树中。只有在调用 epoll_ctl() 的时候，才会发生从用户态到内核态的数据拷贝。并且内核会在该文件描述符的内部结构上注册一个回调函数，使其添加到 epoll 实例内部的“就绪队列”中。
2.  **核心等待与处理：**
    *   在 `while(true)` 循环中，调用 `nfds = epoll_wait(epfd, events, 5, 10000)`。这个函数会阻塞，直到内核的**就绪列表**中有事件发生。
    *   `epoll_wait()` 直接返回就绪的 FD 列表，`nfds` 的值就是就绪的 FD 的数量。内核已经将这些就绪的事件填充到了 `events` 数组中。
    *   程序只需要遍历“nfds”次，直接从 `events` 数组中取出就绪的 FD 进行 `read()` 操作。

**`epoll` 的巨大优势:**
*   **高效的管理机制**：FD 集合的管理是通过红黑树实现的，增删改查的效率都很高。FD 只需要注册一次，避免了 `select`/`poll` 反复拷贝的开销。
*   **避免线性扫描**：`epoll_wait()` 直接返回就绪的 FD 列表，程序无需遍历所有 FD，时间复杂度是 O(1)（或者说 O(k)，其中 k 是就绪的 FD 数量），并且 `epoll_ctl()` 注册时会同时注册一个回调函数，无需遍历红黑树。
*   **内存拷贝优化**：`epoll` 使用**共享内存**（mmap）技术来优化用户空间和内核空间的数据交换，进一步减少了拷贝开销。
*   **支持边缘触发（ET）**：除了水平触发（LT），`epoll` 还支持边缘触发模式。ET 模式更加高效，它只在 FD 状态发生变化时通知一次，要求程序员必须一次性将数据读完。这减少了 `epoll_wait()` 被唤醒的次数，是许多高性能服务的首选。

#### **五、 总结与对比**
| 特性 | `select` | `poll` | `epoll` |
| :--- | :--- | :--- | :--- |
| **底层数据结构** | 位图（bitmap） | 数组（struct pollfd） | 红黑树 + 就绪列表 |
| **最大连接数** | 有限（通常 1024） | 无限制（受内存影响） | 无限制（受内存影响） |
| **FD 拷贝** | 每次调用都需两次拷贝 | 每次调用都需两次拷贝 | 仅在 `epoll_ctl()` 时拷贝一次 |
| **就绪 FD 查找** | 线性扫描 O(n) | 线性扫描 O(n) | 直接返回 O(1) |
| **工作模式** | 水平触发 (LT) | 水平触发 (LT) | 水平触发 (LT) + 边缘触发 (ET) |
| **平台** | 跨平台性好 | 跨平台性较好 | Linux Only |
