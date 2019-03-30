---
title: 聊聊网络事件中的惊群效应
type: tags
date: 2019-03-28 22:00:00
tags: [Unix,Python,网络编程,编程]
categories: [编程,Python]
---

关于惊群问题，其实我是在去年开始去关注的。然后向 CPython 提了一个关于解决 `selector` 的惊群问题的补丁 [BPO-35517](https://bugs.python.org/issue35517)。现在大概来聊聊关于惊群问题那点事吧

<!--more-->

## 惊群问题的过去

### 惊群问题是什么？

惊群问题又名惊群效应。简单来说就是多个进程或者线程在等待同一个事件，当事件发生时，所有线程和进程都会被内核唤醒。唤醒后通常只有一个进程获得了该事件并进行处理，其他进程发现获取事件失败后又继续进入了等待状态，在一定程度上降低了系统性能。

可能很多人想问，惊群效应为什么会占用系统资源？降低系统性能？

1. 多进程/线程的唤醒，涉及到的一个问题是上下文切换问题。频繁的上下文切换带来的一个问题是数据将频繁的在寄存器与运行队列中流转。极端情况下，时间更多的消耗在进程/线程的调度上，而不是执行

接下来我们来聊聊我们网络编程中常见的惊群问题。

### 常见的惊群问题

在 Linux 下，我们常见的惊群效应发生于我们使用 `accept` 以及我们 `select` 、`poll` 或 `epoll` 等系统提供的 API 来处理我们的网络链接。
 
#### accept 惊群  

首先我们用一个流程图来复习下我们传统的 `accept` 使用方式

![image](https://user-images.githubusercontent.com/7054676/55270168-e467d600-52d6-11e9-9779-8ba62b0b42e1.png)

那么在这里存在一种情况，即当一个请求到达时，所有进程/线程都开始 accept ，但是最终只有一个获取成功，我们来写段代码看看

```c
#include <arpa/inet.h>
#include <assert.h>
#include <errno.h>
#include <netinet/in.h>
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

#define SERVER_ADDRESS "0.0.0.0"
#define SERVER_PORT 10086
#define WORKER_COUNT 4

int worker_process(int listenfd, int i) {
  while (1) {
    printf("I am work %d, my pid is %d, begin to accept connections \n", i,
           getpid());
    struct sockaddr_in client_info;
    socklen_t client_info_len = sizeof(client_info);
    int connection =
        accept(listenfd, (struct sockaddr *)&client_info, &client_info_len);
    if (connection != -1) {
      printf("worker %d accept success\n", i);
      printf("ip :%s\t", inet_ntoa(client_info.sin_addr));
      printf("port: %d \n", client_info.sin_port);
    } else {
      printf("worker %d accept failed", i);
    }
    close(connection);
  }

  return 0;
}

int main() {
  int i = 0;
  struct sockaddr_in address;
  bzero(&address, sizeof(address));
  address.sin_family = AF_INET;
  inet_pton(AF_INET, SERVER_ADDRESS, &address.sin_addr);
  address.sin_port = htons(SERVER_PORT);
  int listenfd = socket(PF_INET, SOCK_STREAM, 0);
  int ret = bind(listenfd, (struct sockaddr *)&address, sizeof(address));
  ret = listen(listenfd, 5);
  for (i = 0; i < WORKER_COUNT; i++) {
    printf("Create worker %d\n", i + 1);
    pid_t pid = fork();
    /*child  process */
    if (pid == 0) {
      worker_process(listenfd, i);
    }
    if (pid < 0) {
      printf("fork error");
    }
  }

  /*wait child process*/
  int status;
  wait(&status);
  return 0;
}

```

我们来看看运行的结果

![image](https://user-images.githubusercontent.com/7054676/55270562-2135cc00-52db-11e9-8ad8-71efe6e7962a.png)

诶？怎么回事？为什么这里没有出现我们想要的现象（一个进程 accept 成功，三个进程 accept 失败）？原因在于在 Linux 2.6 之后，Accept 的惊群问题从内核上被处理了

好，我们接着往下看

#### select/poll/epoll 惊群

我们以 `epoll` 为例，我们来看看传统的工作模式

![image](https://user-images.githubusercontent.com/7054676/55270670-a1a8fc80-52dc-11e9-96c2-49be1aa78e7f.png)

好了，我们来看段代码

