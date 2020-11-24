---
title: Nginx学习笔记
date: 2019-08-04 09:04:05
tags:
  - nginx
photos:
  - ["http://sysummerblog.club/nginx_cover.jpeg"]
---
Nginx 是一个免费、开源、高性能、轻量级的 HTTP 和反向代理服务器，也是一个电子邮件(IMAP/POP3)代理服务器，其特点是占有内存少，并发能力强。
<!--more-->

## nginx的架构
Nginx 里有一个 master 进程和多个 worker 进程。master 进程并不处理网络请求，主要负责调度工作进程：加载配置、启动工作进程及非停升级。worker 进程负责处理网络请求与响应。

![](http://sysummerblog.club/nginx_process.png)

master进程主要用来管理worker进程，具体包括如下4个主要功能：

接收来自外界的信号。

向各worker进程发送信号。

监控woker进程的运行状态。

当woker进程退出后（异常情况下），会自动重新启动新的woker进程。

woker进程主要用来处理基本的网络事件：

多个worker进程之间是对等且相互独立的，他们同等竞争来自客户端的请求。

一个请求，只可能在一个worker进程中处理，一个worker进程，不可能处理其它进程的请求。

worker进程的个数是可以设置的，一般我们会设置与机器cpu核数一致。同时，nginx为了更好的利用多核特性，具有cpu绑定选项，我们可以将某一个进程绑定在某一个核上，这样就不会因为进程的切换带来cache的失效。
## 建立连接的步骤

1. 最开始master进程会建立、绑定、监听一个socket。
2. 然后fork出多个work进程，work进程共享master监听的文件描述符
3. 当socket上有数据的时候，work进程会通过锁来竞争资源
4. 竞争到资源的work进程调用accept方法接收连接，accept方法会返回一个新的socket，之后就是这个新的socket与客户端建立3次握手和数据的交换。需要注意的是新建立的这个socket所监听的端口号还是80或443。唯一确定一个socket（只考虑tcp协议的）是四元组：客户端ip, 客户端端口, 服务器ip, 服务器端口。只要有一个元素不一样，就表示不同的连接。因此一单建立了连接，下次再通信的时候数据就会直接到第一次accept的那个socket，并不会再到达master进程最开始监听的那个socket了。


## 处理事件
每个work进程可以处理两类事件

1. 处理已监听的socket的读或写就绪事件
2. 处理新的连接

多个进程间通过争夺一块锁来决定谁处理新的连接。同一时刻只能有一个进程获取到锁。也因此不会产生“惊群”现象。

获取到锁的进程调用epoll_wait方法返回的事件列表中既有已监听的socket的读或写就绪事件，也有要处理新的连接事件。新的连接事件的优先级比较高。

没有获取到锁的进程调用epoll_wait方法只会获取到该进程监听的socket上的读或写就绪事件。

每个work进程在启动后都会调用内核的epoll_create方法。该方法返回一个文件描述符，并在内核维护一个epoll模块。之后work进程通过这个文件描述符来实现对自己感兴趣的事件的添加、删除、修改、监听。

当work进程获得一个新的连接的时候，会把这个socket上的读就绪事件通过epoll_ctl函数挂载在epoll模块的红黑树上。

work进程通过epoll_wait函数获取到了socket上有可读事件后就开始处理该事件，处理的过程很短，就是把该请求根据用户的配置交到下游处理模块中。

## 模块
Nginx 由内核和一系列模块组成，内核提供 Web 服务的基本功能，如启用网络协议，创建运行环境，接收和分配客户端请求，处理模块之间的交互。

Nginx 的各种功能和操作都由模块来实现。Nginx 的模块从结构上分为：

* 核心模块：HTTP 模块、EVENT 模块和 MAIL 模块。
* 基础模块：HTTP Access 模块、HTTP FastCGI 模块、HTTP Proxy 模块和 HTTP Rewrite 模块。
* 第三方模块：HTTP Upstream Request Hash 模块、Notice 模块和 HTTP Access Key 模块及用户自己开发的模块。

## 配置
nginx的配置有三大模块

1. 全局块
2. event块
3. http块

### 全局块
全局块负责一些全局的配置，部分配置如下

1. user 规定运行nginx的用户和用户组
2. worker_processes worker进程的数量
3. worker_rlimit_nofile 一个worker进程最大的打开文件的数量

### event块
event块里的配置是有关事件的，部分配置如下

1. worker_connections 一个work进程能够处理的最大的连接数
2. accept_mutex 当一个新连接到达时，如果激活了accept_mutex，那么多个Worker将以串行方式来处理，其中有一个Worker会被唤醒，其他的Worker继续保持休眠状态；如果没有激活accept_mutex，那么所有的Worker都会被唤醒，不过只有一个Worker能获取新连接，其它的Worker会重新进入休眠状态
3. use epoll 使用epoll来管理io事件

### http块
http块中的配置包含两部分，公共部分和server部分。公共部分是所有server都采用的，server部分是每个server单独采用的。

#### 公共配置

1. client_header_buffer_size 请求头缓冲区大小
2. large_client_header_buffers 当请求头太大的时候client_header_buffer_size放不下了，会向large_client_header_buffers申请一块额外的内存
3. client_max_body_size 请求body的最大
4. client_body_timeout 读取请求body的超时时间
5. client_header_timeout 读取请求header的超时时间
6. log_format access_log的日志格式
7. sendfile 支持“零拷贝”
8. keepalive_timeout 长连接的超时时间。超时后会nginx会释放socket资源。如果客户端再想通信需重新3次握手

#### server配置
server配置中一般会有多个location，把不同的url分配给不同的下游服务

nginx中以$开头的都是nginx的变量。nginx在把请求发给fastcgi之前会把nginx的变量转化成fastcgi变量。fastcgi的变量由大写字母和下划线组成。在php的进程中，fastcgi的变量都在SERVERA全局数组中。

##### rewrite
```sh
if (!-e $request_filename) {
    rewrite ^(.*)$ /index.php?_url=$1 last;
}
```
$request_filename是根路径拼接的uri(域名后面的一段)。如果用户请求的不是一个文件那么久将进行重写。重写的规则是请求根域名下的index.php,而uri变成了_url参数

rewrite最后会有一个标识符

* break url重写后，直接使用当前资源，不再执行location里余下的语句，完成本次请求，地址栏url不变 
* last url重写后，马上发起一个新的请求，再次进入server块，重试location匹配，超过10次匹配不到报500错误，地址栏url不变
* redirect 返回302临时重定向，地址栏显示重定向后的url，爬虫不会更新url（因为是临时）
* permanent 返回301永久重定向, 地址栏显示重定向后的url，爬虫更新url

##### location
在匹配location的时候

* =  精确匹配
* ~  表示大小写敏感的正则匹配
* ~* 表示大小写不敏感的正则匹配

以php为例。以.php为结尾的请求都转给fastcgi
```sh
location ~ \.php$ {
    set $real_script_name $fastcgi_script_name;
    if ($fastcgi_script_name ~ /lianjia(/.*)$) {
        set $real_script_name $1;
    }
    fastcgi_pass    127.0.0.1:18181;
    fastcgi_index  index.php;
    fastcgi_param  SCRIPT_FILENAME $document_root$real_script_name;
    include        tengine.fastcgi_params;
}
```

## 负载均衡
nginx的负载均衡策略有三种

1. 平均分配
2. 根据权重
3. 根据客户端的ip

无论哪一种，如果其中的一台机器出问题了，nginx会自动剔除，用起来很方便

### 平均负载
```sh
http{
    # 待选服务器列表
    upstream myproject{
        server 125.219.42.4;
        server 172.31.2.183;
    }

    server{
        listen 80; 
        location / {            
            proxy_pass http://myproject;
        }

   }
}
```

### 根据权重负载
```sh
http{
    # 待选服务器列表
    upstream myproject{
        server 125.219.42.4 weight 1;
        server 172.31.2.183 weight 2;
    }

    server{
        listen 80; 
        location / {            
            proxy_pass http://myproject;
        }

   }
}
```

### 根据用户ip负载 
```sh
http{
    # 待选服务器列表
    upstream myproject{
        ip_hash;
        server 125.219.42.4;
        server 172.31.2.183;
    }

    server{
        listen 80; 
        location / {            
            proxy_pass http://myproject;
        }

   }
}
```
