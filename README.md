# Mio – Metal I/O(阅读笔记)

当前版本 `1.0.2`

## kqueue

在macOS和freebsd中, mio的底层使用kqueue实现.

```c
int kq = kqueue();  // 创建 kqueue 实例, kq是一个文件描述符
struct kevent event;
EV_SET(&event, fd, EVFILT_READ, EV_ADD | EV_ENABLE, 0, 0, NULL); // 初始化event
kevent(kq, &event, 1, NULL, 0, NULL);  // 注册事件
struct kevent events[10];
int nev = kevent(kq, NULL, 0, events, 10, NULL);  // 等待事件
close(kq);  // 关闭 kqueue
```

```c
int kevent(int kq, const struct kevent *changelist, int nchanges,
           struct kevent *eventlist, int nevents,
           const struct timespec *timeout);
```

- `int kq`: `kq` 是通过 `kqueue()` 创建的 `kqueue` 描述符，用于标识该事件队列。
- `const struct kevent *changelist`:`changelist` 是一个指向 `kevent` 结构数组的指针，用于指定要添加、修改或删除的事件。可以通过 `EV_SET()` 宏来初始化这些 `kevent` 结构体。 可以将其设为 `NULL`，如果不想注册新的事件，仅想获取当前事件列表。
- `int nchanges`: `nchanges` 是 `changelist` 中的事件个数，表示希望添加或修改的事件的数量。如果不需要注册或修改事件，将此值设置为 0。
- `struct kevent *eventlist`: `eventlist` 是一个指向 `kevent` 结构数组的指针，用于接收发生的事件。当有事件触发时，这里会返回相应的事件信息。 如果不想获取事件，将其设为 `NULL`。
- `int nevents`: `nevents` 是 `eventlist` 中可接收的最大事件个数，表示返回的事件数量。如果只需要查询事件状态并不关心实际的事件，设置为 0。
- `const struct timespec *timeout`: `timeout` 是一个指向 `timespec` 结构的指针, 用于指定等待事件的超时时间。

`timeout`的可选值：
- `NULL`：表示无限等待，直到有事件发生。
- 指定超时时间：例如 `timespec ts = {1, 0};` 表示等待 1 秒。
- `timeout->tv_sec == 0` 且 `timeout->tv_nsec == 0` 时，`kevent()` 是非阻塞的。

```c
struct kevent {
    uintptr_t ident;     // 事件的标识符，通常是文件描述符（如套接字、管道等），但也可以是其他类型的标识符，具体取决于事件类型。
    int16_t   filter;    // 事件过滤器，指定要监控的事件类型，如读、写等。
    uint16_t  flags;     // 事件的状态标志, 用来控制事件的添加、修改、删除等操作。
    uint32_t  fflags;    // 事件的额外标志，根据过滤器类型的不同有不同的含义。例如在监控文件系统事件时，fflags 可能会包含标识具体变化类型的标志位。
    intptr_t  data;      // 事件相关的数据，事件产生的相关数据，例如读写事件时表示可读或可写的数据字节数。
    void      *udata;    // 用户自定义数据，可以在设置事件时使用。在事件返回时会保留这个值，用来区分不同事件源或进行事件处理时的上下文传递。
};
```

kevent的`filter`类型:
1. `EVFILT_READ`: 监控文件描述符上的可读事件。当文件描述符上有数据可读取时触发，例如套接字、管道、文件等。
2. `EVFILT_WRITE`: 监控文件描述符上的可写事件。当文件描述符可以写入数据时触发。
3. `EVFILT_TIMER`: 基于定时器的事件。允许设置相对或周期性的定时器，当时间到达时触发。
4. `EVFILT_SIGNAL`: 监听系统信号（如 `SIGINT`、`SIGTERM` 等），当收到指定的信号时触发事件。
5. `EVFILT_VNODE`: 监控文件系统事件。当文件或目录的状态发生变化时触发，例如文件被修改、删除等。需要设置 `fflags` 来指定监控的具体事件类型。
6. `EVFILT_PROC`: 监控进程状态的变化，例如进程退出、挂起、恢复等。需要在 fflags 中指定感兴趣的进程状态。
7. `EVFILT_USER`: 用于自定义事件。用户可以通过触发机制手动触发事件，用于进程内或者线程间的自定义事件处理。
8. `EVFILT_AIO`: 监控异步 I/O 操作完成事件（仅在支持异步 I/O 的系统中有效）。
9. `EVFILT_EMPTY`: 一个虚拟过滤器，当前没有实际使用。可以用于将空事件加入 kqueue，一般没有实际应用。
10. `EVFILT_EXCEPT`: 监控文件描述符上发生的异常事件，比如异常关闭等。
11. `EVFILT_FS`: 监控文件系统事件，例如挂载或卸载文件系统时触发（特定的系统实现可能会有所不同）。
12. `EVFILT_MACHPORT`: 监控 Mach 端口的状态变化，用于 Mach IPC（进程间通信）。
13. `EVFILT_SOCK`: 监控套接字的状态变化。通常与其他过滤器结合使用，如监听套接字的建立、断开等状态变化。

`fflags`类型:

`EVFILT_VNODE(文件事件监控时)`，fflags 相关标志:
- `NOTE_DELETE`: 文件被删除。
- `NOTE_WRITE`: 文件内容被修改。
- `NOTE_EXTEND`: 文件大小被扩展。
- `NOTE_ATTRIB`: 文件属性被修改。
- `NOTE_LINK`: 文件链接计数被改变。
- `NOTE_RENAME`: 文件被重命名。
- `NOTE_REVOKE`: 文件权限被撤销。

`EVFILT_PROC(进程事件监控时)`，fflags 相关标志:
- `NOTE_EXIT`: 进程退出。
- `NOTE_FORK`: 进程创建子进程。
- `NOTE_EXEC`: 进程执行新程序。
- `NOTE_SIGNAL`: 进程收到信号。
- `NOTE_TRACK`: 跟踪进程的创建和退出（适用于子进程）。

`EVFILT_USER（用户自定义事件)`，fflags 相关标志:
- `NOTE_TRIGGER`: 手动触发用户事件。

`flags`类型:
- `EV_ADD`: 将事件添加到 `kqueue` 中。如果该事件已经存在，则更新该事件的配置。
- `EV_DELETE`: 从 `kqueue` 中移除事件。
- `EV_ENABLE`: 启用已经存在的事件。如果事件被禁用，则重新启用它。
- `EV_DISABLE`: 禁用一个事件，禁用后不再收到该事件的通知，但事件仍然保留在 `kqueue` 中。
- `EV_ONESHOT`: 事件触发一次后，自动将该事件从 `kqueue` 中移除。
- `EV_CLEAR`: 清除触发状态。默认情况下，当一个事件触发时，会继续保持该状态，直到事件被处理(水平触发)。如果使用 `EV_CLEAR`，则需要每次事件触发后重置状态，等待下一次触发(边沿触发)。
- `EV_RECEIPT`: 使用此标志时，`kevent()` 只返回操作的结果状态，可以查看`changelist`中的`data`检查是否出错, 如果`data`不是0, 表示有错误。
- `EV_DISPATCH`: 该事件触发后，将会被禁用，直到手动使用 EV_ENABLE 重新启用。
- `EV_EOF`: 指示在该事件的文件描述符上发生了 EOF（即文件结束）。此标志通常在读取数据时出现，表示已到达数据的末尾。
- `EV_ERROR`: 表示在事件操作中发生了错误。此标志用于向用户传递有关事件处理中的错误信息。检查 `kevent()` 操作时发生的错误，并通过 `data` 字段查看具体的错误码。
- `EV_FLAG1`: 这是一个保留标志，通常在某些特定的系统调用或实现中使用，并且不适用于普通事件操作。
- `EV_SYSFLAGS`: 系统保留的内部标志。此标志在用户操作中通常不需要设置。
- `EV_POLL`: 用于轮询`(polling)`模式下的 `kqueue`。此标志主要应用于部分平台上实现的 `poll()` 函数的兼容性模式。

## epoll

在linux中, mio的底层使用epoll实现.

```c
int epoll_create(int size); // size已经不再使用, 设置为大于 0 的任意值
int epoll_create1(int flags); // flags可以设置0或EPOLL_CLOEXEC(表示在执行 exec 系列系统调用时自动关闭该 epoll 文件描述符)
```

```c
struct epoll_event {
    uint32_t events;  // 事件类型
    epoll_data_t data;  // 关联的数据
};
```
常见的`events`类型包括：
- `EPOLLIN`: 表示对应的文件描述符可以读（包括普通文件、管道、网络套接字等）。
- `EPOLLOUT`: 表示对应的文件描述符可以写。
- `EPOLLRDHUP`: 表示套接字的远端关闭（半关闭状态）。
- `EPOLLPRI`: 表示有紧急数据可读（带外数据，通常用于 socket）。
- `EPOLLERR`: 表示发生错误，通常是描述符错误状态。
- `EPOLLHUP`: 表示挂起事件，表示文件描述符被挂起或终止。
- `EPOLLET`: 表示使用边缘触发（Edge Triggered）模式。
- `EPOLLONESHOT`: 表示一次性触发，即触发一次事件后，文件描述符将被禁用，必须再次通过 epoll_ctl 重新添加才能继续监控。

```c
typedef union epoll_data {
    void *ptr;        // 指向用户定义数据的指针
    int fd;           // 文件描述符
    uint32_t u32;     // 32 位的无符号整数
    uint64_t u64;     // 64 位的无符号整数
} epoll_data_t;
```

```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

1. `epfd`：由 `epoll_create` 或 `epoll_create1` 返回的 `epoll` 实例的文件描述符。
2. `op`：指定要执行的操作，可以是以下三种操作之一：
   - `EPOLL_CTL_ADD`：向 `epoll` 实例中添加一个新的文件描述符及其事件。
   - `EPOLL_CTL_MOD`：修改已经存在的文件描述符的事件。
   - `EPOLL_CTL_DEL`：从 `epoll` 实例中删除一个文件描述符。
3. `fd`：要被管理的目标文件描述符（如套接字或管道的文件描述符）。
4. `event`：指向 `epoll_event` 结构体的指针，用于指定要监听的事件类型（可读、可写等）。当 `op` 是 `EPOLL_CTL_ADD` 或 `EPOLL_CTL_MOD` 时，`event` 不能为空；
当 `op` 是 `EPOLL_CTL_DEL` 时，`event` 可以为 `NULL`，因为删除操作不需要指定事件。

```c
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```
1. `epfd`：由 `epoll_create` 或 `epoll_create1` 返回的 `epoll` 实例的文件描述符。
2. `events`：指向一个 `epoll_event` 结构体数组的指针，该数组用于存储已触发的事件。调用 `epoll_wait` 时，操作系统会将已经发生事件的文件描述符和事件信息填入此数组。
3. `maxevents`：指定 `events` 数组的最大容量。这个值必须是大于 0 的数字，表示程序最多可以处理的事件数量。
4. `timeout`：等待事件发生的超时时间，单位是毫秒：
   - `0`：立即返回，即使没有任何事件发生（非阻塞模式）。
   - `负值`：无限期等待，直到有事件发生（阻塞模式）。
   - `正值`：等待指定的毫秒数，如果在此时间内没有事件发生，函数将返回 0。

