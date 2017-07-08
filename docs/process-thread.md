# 操作系统 - 进程与线程

标签（空格分隔）： 操作系统

---
[TOC]

## 进程与线程之间的联系与区别

进程是系统资源分配的基本单位例如内存资源，两个进程之间内存是隔离的，而线程是 CPU 调度的基本单位；
进程体现的是系统的并发能力，线程体现系统的并行能力；
线程是进程的一部分，一个进程中可以包含多个进程；
[进程和线程的区别](https://www.zhihu.com/question/44087187) 一些回答也可以简要的说明。
[Linus 在邮件中的讨论说明](https://www.zhihu.com/question/44087187/answer/96885903?from=profile_answer_card)，这个回答中提到的 `执行环境`（context of execute, COE）可以了解。

---

## 进程控制

文章中函数参数的具体信息可以从《Unix 环境高级编程》中得知，在此不详述。
《Unix 环境高级编程》好难看，好难看……

### 进程标识符
> 每一个进程都有一个非负整性表示唯一的进程 ID， 因为进程 ID 标识符总是唯一，常将其用来作其他标识符的一部分以保证其唯一性
> 虽然是唯一的，但是进程 ID 可以重用，当一个进程终止后，其进程 ID 就可以再次使用
> 《Unix 环境高级编程》

### fork函数

一个现有进程可以调用 `fork()` 创建新进程；
`fork` 函数原型如下：

```c
#include<unistd.h>
pid_t fork(void);
```

`fork` 一次调用有两个返回值，其中父进程的返回值是子进程的 `pid`， 其返回值大于 0 ，因为没有函数可以提供子进程的进程 ID ，这样设计可以提供父进程子进程的 `pid`。

> 子进程和父进程继续执行 fork 调用之后的指令，子进程是父进程的副本。注意子进程所拥有的副本，父、子进程并不共享这些存储空间部分。父子进程共享正文段

[一个fork的面试题](http://coolshell.cn/articles/7965.html) 涉及 fork 的特性。

### wait 和 waitpid 函数

一个已经终止的进程但是其父进程未获取子进程的终止状态，释放进程占用资源的进程称为僵死进程；当一个子进程在父进程之前终止，父进程调用 wait 或 waitpid 函数获取子进程的状态。
函数原型如下：

```c
#include<sys/wait.h>
pid_t wait(int *statloc);
pid_t wait(pid_t pid, int * statloc, int options);
```

区别如下：
1. 子进程未终止，调用 `wait` 使调用者阻塞， `waitpid` 有一选项可以不阻塞
2. `waitpid` 有若干选项可以控制它所等待的进程
参数信息可以查看《Unix 环境高级编程》

---

## 进程间通信 (IPC)

### 匿名管道（pipe）

pipe 管道存在以下的缺陷：
1. 数据流单向流动
2. 只能用于具有公共祖先的进程

函数原型如下：

```c
#include<unistd.h>
int pipe(int filedes[2])
```

其中 `filedes[0]` 是读端口， `filedes[1]` 是写端口。

Demo：
```c
int main()
{
    int fd[2];
    pid_t pid;

    if(pipe(fd) < 0)
    {
        fprintf(stderr, "pipe error");
        exit(EXIT_FAILURE);
    }
    if( (pid= fork()) < 0)
    {
        fprintf(stderr, "fork error");
        exit(EXIT_FAILURE);
    }
    else if(pid > 0)
    {// parent
        wait(NULL);//子进程未终止，则阻塞
        printf("Parent process\n");
        char buf[BUFSIZ];
        int cnt;
        cnt = read(fd[0], buf, BUFSIZ);
        buf[cnt] = '\0';
        printf(" read msg <%s> from pipe\n", buf);
        close(fd[0]);
        exit(EXIT_SUCCESS);
    }
    else
    {//child
        printf("Child process\n");
        write(fd[1], "Hello world", 11);
        close(fd[1]);
        exit(EXIT_SUCCESS);
    }
    return 0;
}
```

### 命名管道（FIFO）
FIFO 也被称为命名管道，弥补了匿名管道只能在具有公共祖先进程中使用的缺陷。FIFO是一种类似文件结构，所以可以用 `read／write` 函数读写。
创建 FIFO 管道
```c
#include<sys/stat.h>
mkfifo(const char *file_path, int modes)
```

Demo
```cplusplus
#define FILE_PATH "./fifo"

int main()
{
    pid_t pid;
    //创建 FIFO 管道，进程只需要知道文件路径，就可以通信
    mkfifo(FILE_PATH, 0777);
    if((pid = fork()) < 0)
    {
        fprintf(stderr, "fork error");
        exit(EXIT_FAILURE);
    }
    else if( pid > 0)
    {//parent
        wait(NULL);
        printf("Parent process\n");
        // 类文件结构，可以用有关文件函数操作 FIFO
        int fd = open(FILE_PATH, O_RDONLY);
        char buf[BUFSIZ];
        read(fd, buf, BUFSIZ);
        printf("read msg <%s> from fifo\n", buf);
        close(fd);
        exit(EXIT_SUCCESS);
    }
    else{//child
        printf("Child process\n");
        int fd = open(FILE_PATH, O_WRONLY);
        write(fd, "Hello world", 11);
        close(fd);
        exit(EXIT_SUCCESS);
    }
    return 0;
}
```

当打开一个 FIFO 管道时，可以设置非阻塞 O_NONBLOCK 模式, 这个标志位，open的时候可以立即返回，一般不设置。
一个FIFO，可能有多个写程序；一般应该做好写进程间的同步，以及写操作的原子化，要么全写，要么不写。

### 消息队列

与消息队列有关函数

```c
#include<sys/msg.h>

int msgget(key_t key, int flag);
int msgsnd(int msqid, const void *prt, size_t nbytes, int flag);
int msgrcv(int msqid, void *prt, size_t nbytes, long type, int flag);
int msgctl(int msqid, int cmd, struct msqid_ds *buf);
//消息结构
struct mymesg{
    long type; //消息类型
    char text[BUFSIZ]; //消息内容
};
```

Demo
```
int main()
{
    pid_t pid;
    int qid;
    key_t key;
    //创建消息队列
    qid = msgget(key, IPC_CREAT|0777);

    if( (pid=fork()) < 0)
    {
        fprintf(stderr, "fork error\n");
        exit(EXIT_FAILURE);
    }
    else if (pid > 0)
    {//parent
        wait(NULL);
        struct mymsg msg_r;
        // 接收第一个类型为 10 的消息，并设置为非阻塞，立即返回
        int len = msgrcv(qid, &msg_r, BUFSIZ, 10, IPC_NOWAIT);
        msg_r.text[len] = '\0';
        printf("recieve type %ld, msg <%s> from msg queue\n", msg_r.type, msg_r.text);
        // 删除消息队列
        msgctl(qid, IPC_RMID, NULL);
        exit(EXIT_SUCCESS);
    }
    else
    {//child
        struct mymsg msg_w;
        msg_w.type = 10;
        strcpy(msg_w.text, "Hello world");
        msgsnd(qid, &msg_w, strlen(msg_w.text), IPC_NOWAIT);
        exit(EXIT_SUCCESS);
    }
    return 0;
}
```

消息队列并不是粗暴的先进先出的获取消息，可以根据消息的类型信息获取消息，由此我们可以制定消息的优先级别。同时不同的进程可以共享这个消息队列，共享的方式是传递队列的ID, 或者是创建消息队列的 key。

### 信号量

信号量其实是操作系统中的同步原语，控制进程之间的同步关系， 是 PV 操作的实际实现。同时
与信号量有关的函数：

```c
#include<sys/sem.h>
int semget(key_t key, int nsems, int flag);
int semop(int semid, struct sembuf sems[], size_t nops);
int semctl(int semid, int semums, int cmd);
struct sembuf{
    unsigned short sem_num;
    short sem_op;
    short sem_flg;
};
```

系统中信号量的定义是一个信号量集合，如 `semget` 创建指定 `nsems` 数量的信号量，所以当操作信号量时需要指定 `sembuf.sem_num` 编号。 `sembuf.sem_op` 值体现信号量操作。

Demo

```c
int main()
{
    pid_t pid;
    key_t key;
    int semid = semget(key, 1, IPC_CREAT|0777);
    if((pid = fork()) < 0)
    {
        fprintf(stderr, "fork error\n");
        exit(EXIT_FAILURE);
    }
    else if (pid >0)
    {//parent 
        int i  = 3;
        struct sembuf sb;
        sb.sem_num = 0;
        sb.sem_op = 1;
        sb.sem_flg = 0;
        semop(semid, &sb, 1);
        while( i> 0)
        {
            sb.sem_op = -1;
            semop(semid, &sb, 1);
            printf("Parent process\n");
            i -= 1;
            sb.sem_op = 1;
            semop(semid, &sb, 1);
        }
        semctl(semid, 0, IPC_RMID);
        exit(EXIT_SUCCESS);
    }
    else
    {
        int i = 2;
        struct sembuf sb;
        while( i > 0)
        {
            sb.sem_num = 0;
            sb.sem_op = -1;
            sb.sem_flg = 0;
            semop(semid, &sb, 1);
            printf("Child process\n");
            i -= 1;
            sb.sem_op = 1;
            semop(semid, &sb, 1);
        }
        exit(EXIT_SUCCESS);
    }
}
```

### 共享内存

共享内存允许多个进程可以使用共同的一块内存区域，因为不需要在服务端客户端之间复制，所以共享内存是最快的 IPC 之一。就如 FIFO 一样，我们需要利用信号量或者其他进程间同步的手段，控制进程的读写操作。

共享内存相关的函数如下：

```c
#include<sys/shm.h>
int shmget(key_t key, size_t size, int flag);
int shmctl(int shmid, int cmd, struct shmid_ds *buf);
void *shmat(int shmid, const void * addr, int flag);
```

Demo

```c
int main()
{
    pid_t pid;
    key_t key;
    int shmid = shmget(key, BUFSIZ,IPC_CREAT | 0666);
    if( (pid = fork()) < 0)
    {
        fprintf(stderr, "fork error");
        exit(EXIT_FAILURE);
    }
    else if (pid > 0)
    {//parent
        wait(NULL);
        char *addr;
        addr = shmat(shmid,0, 0);
        printf(" msg <%s>\n", addr);
        shmctl(shmid, IPC_RMID, NULL);
        exit(EXIT_SUCCESS);
    }
    else
    {//child
        char* addr;
        addr = shmat(shmid, 0, 0);
        strcpy(addr,"Hello world");
        exit(EXIT_SUCCESS);
    }

}
```

 以上几种进程间通讯方式（除管道外），都需要 `key_t key` 作为键，这个值将会被内核修改，使得 IPC 对象与键对应，进程间利用这个键可以访问到 IPC 对象， Demo 程序示例是父子进程之间共享 ID 的方法获取对象的， `***get` 函数是可以创建一个对象，也可以打开一个已有对象。`***ctl` 函数都是根据命令做相应操作的函数。

---

## 线程同步与互斥

由于线程是 CPU 的调度单位， 进程才是资源的分配单位，所以同一个进程之间的线程资源是共享的；线程则主要了解数据的线程安全性，线程之间的同步以及锁的概念。
线程之间同步方法有以下几种：
1. 互斥量（mutex）
2. 读写锁
3. 条件变量

---





