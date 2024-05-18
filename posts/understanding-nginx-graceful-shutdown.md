# Understanding Nginx graceful shutdown

Nginx 是目前最流行的反向代理和 Web 服务器，它的性能非常高，单机可处理 10 万 RPS，C10k 仅占用 2.5MB 内存。Nginx 被广泛应用于代理、负载均衡、HTTP 缓存、CDN、API Gateway 等不同领域。

Nginx 流行的原因之一还包括它本身支持零停机的配置热更新(reload)。

### 什么是配置热更新

在修改 Nginx 配置文件后，通过 `nginx -s reload` 命令应用新配置而不需要重启，称为配置热更新。这行命令向 master 进程发送 `HUP` 信号，master 进程收到信号会校验配置是否合法，并启动新的 worker 进程，再向旧 worker 发送 `QUIT` 信号请求其执行优雅退出(graceful shutdown)。当 worker 收到信号后，会首先停止接收新请求，但不会中断当前正在处理的请求，在所有请求处理完毕后，进程才会关闭，这个过程称为优雅退出。

### 优雅退出机制

要理解 Nginx 的优雅退出，离不开阅读源码来理解它的底层实现。好在这部分逻辑并不复杂，即使没有丰富的 C 语言经验也不妨碍窥探它的原理。Nginx 的一个特点是事件循环，所有 worker 进程被创建后都会进入 `ngx_worker_process_cycle` 函数，process_cycle 顾名思义就是处理循环(aka 事件循环) —— 在一个 for ( ;; ) 循环中读取并处理 event，也包括执行优雅退出。


### ngx_worker_process_cycle 函数解析

函数代码如下

```c
// ngx_process_cycle.c
static void ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data) {
    ngx_int_t worker = (intptr_t) data;

    ngx_process = NGX_PROCESS_WORKER;
    ngx_worker = worker;

    ngx_worker_process_init(cycle, worker);

    ngx_setproctitle("worker process");

    for ( ;; ) {
        if (ngx_exiting) {
            if (ngx_event_no_timers_left() == NGX_OK) {
                ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");
                ngx_worker_process_exit(cycle);
            }
        }

        ngx_process_events_and_timers(cycle); // 事件处理入口函数

        if (ngx_terminate) {
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");
            ngx_worker_process_exit(cycle);
        }

        if (ngx_quit) {
            ngx_quit = 0;
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "gracefully shutting down");
            ngx_setproctitle("worker process is shutting down");

            if (!ngx_exiting) {
                ngx_exiting = 1;
                ngx_set_shutdown_timer(cycle);
                ngx_close_listening_sockets(cycle);
                ngx_close_idle_connections(cycle);
                ngx_event_process_posted(cycle, &ngx_posted_events);
            }
        }

        // ... ngx_reopen
    }
}
```

`ngx_process_events_and_timers` 是 Nginx 事件处理的入口函数，内部包括处理像 HTTP 请求的解析和响应的生成。不过这跟 graceful shutdown 无关，因此不做赘述。

需要我们关注的只有 `for ( ;; )` 块的代码

```c
for ( ;; ) {
    if (ngx_exiting) {
        if (ngx_event_no_timers_left() == NGX_OK) {
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");
            ngx_worker_process_exit(cycle);
        }
    }

    ngx_process_events_and_timers(cycle); // 事件处理入口函数
		
    if (ngx_terminate) {
        ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");
        ngx_worker_process_exit(cycle);
    }

    if (ngx_quit) {
        ngx_quit = 0;
        ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "gracefully shutting down");
        ngx_setproctitle("worker process is shutting down");

        if (!ngx_exiting) {
            ngx_exiting = 1;
            ngx_set_shutdown_timer(cycle);
            ngx_close_listening_sockets(cycle);
            ngx_close_idle_connections(cycle);
            ngx_event_process_posted(cycle, &ngx_posted_events); 
        }
    }
}
```

这里先简单回顾一下执行 `nginx -s reload` 时， master 进程发生了什么

1.   master 进程接收到 `HUP` 信号
2.   master 进程检查配置文件的语法，并打开日志文件和新的 listening socket
3.   master 进程启动新的 worker，向旧 worker 发送信号请求执行优雅退出

worker 进程里通过解析信号后将 `ngx_quit` 置为 1，在 `for ( ;; )` 里对应
```c
for ( ;; ) {
    // ...

    // QUIT signal (graceful shutdown)
    if (ngx_quit) {
        ngx_quit = 0; 
        ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "gracefully shutting down");
        ngx_setproctitle("worker process is shutting down");
    
        if (!ngx_exiting) {
            ngx_exiting = 1; // 标记 worker 为正在退出状态
            ngx_set_shutdown_timer(cycle);
            ngx_close_listening_sockets(cycle);
            ngx_close_idle_connections(cycle);
            ngx_event_process_posted(cycle, &ngx_posted_events); 
        }
    }
    
    // ...
}
```

**1. ngx_set_shutdown_timer(cycle)**

在 1.11.11 (Mar 2017)  版本，Nginx 新增指令 `worker_shutdown_timeout` 用于控制优雅退出的最长超时时间，默认值在 0，表示不设超时时间。内部是通过 nginx timer 实现的。

**2. ngx_close_listening_sockets(cycle)**

关闭监听 socket，确保不会再产生新的客户端连接。

**3. ngx_close_idle_connections(cycle)**

Nginx 中的 Connection 是指客户端和服务器之间的通信通道。主要类型有客户端连接(client connection)和服务端连接(upstream connection)两种。比如在 HTTP 协议中，客户端和服务器之间可以通过建立长连接(Connection: keep-alive)来避免频繁建立连接的开销，在 Nginx 每次处理完连接上的请求时都会将 Connection 的 `idle` 属性设置为 1，表示处于空闲状态。所以 `ngx_close_idle_connections` 的目的是关闭所有空闲的长连接(比如和客户端的长连接)，底层通过调用 socket close 函数关闭套接字。



由于 `ngx_exiting`  被置为 1，那么下一次循环会进入 `for ( ;; )`  里开头的 if (ngx_exiting) 分支

```c
if (ngx_exiting) {
    if (ngx_event_no_timers_left() == NGX_OK) {
        ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");
        ngx_worker_process_exit(cycle);
    }
}
```

如果当前已经没有需要处理的 event 和 timer，则调用 `ngx_worker_process_exit` 关闭 worker，否则继续调用 `ngx_process_events_and_timers` 函数处理所有未处理完成的请求。



以上就是 Nginx 优雅退出的实现机制。我们做个简单的总结，worker 要执行优雅退出，先通过关闭 listening socket 来停止和客户端建立新连接(新连接会由新 worker 处理)，正在处理的请求不会中断，而是等待处理完成后才会退出 worker 进程。



文章结尾，笔者留下几个问题供有兴趣的读者思考

-   `nginx -s reload` 真的是零停机吗？
-   Nginx reload 之后，客户端先前通过 `Connection: keep-alive` 保持的长连接是否还有效呢？
