---
tag: csapp
---

[TOC]

记录下我实现csapp的shell-lab的过程

## 实验背景

实验对应的是csapp的第8章：异常控制流，通过自己实现一个shell，可以充分
了解到进程、子进程、信号传递、信号屏蔽等知识。

学生需要手动实现`tsh.c`这一个文件，而跟之前csapp的实验一样，这次的实验
设计也是非常精巧，提供了一个`tshref`的shell来作为参考。同时，还提供了
循序渐进的`trace*.txt`文件作为慢慢深入的测试用例。

通过一个个完成`trace`文件的测试用例，最后tsh的代码也就一点点都完善起来了。

下面文章也是通过`trace`文件的顺序，慢慢展开`tsh.c`的实现。

## tsh.c文件简介

`tsh.c`已经实现了一些基本的函数，包括`main`函数，解析输入内容的`parseline`
函数，以及`job`数据结构的定义以及一些辅助函数。

学生需要实现的是

- eval: Main routine that parses and interprets the command line. [70 lines]
- builtin_cmd: Recognizes and interprets the built-in commands: quit, fg, bg, and jobs. [25
-ines]
- do_bgfg: Implements the bg and fg built-in commands. [50 lines]
- waitfg: Waits for a foreground job to complete. [20 lines]
- sigchld_handler: Catches SIGCHLD signals. 80 lines]
- sigint_handler: Catches SIGINT (ctrl-c) signals. [15 lines]
- sigtstp_handler: Catches SIGTSTP (ctrl-z) signals. [15 lines]

## trace文件

### trace01

```
#
# trace01.txt - Properly terminate on EOF.
#
CLOSE
WAIT
```

正常退出的功能，不需要修改代码

### trace02

```
#
# trace02.txt - Process builtin quit command.
#
quit
WAIT
```

实现`quit`退出的功能，只需要在builtin_cmd中实现即可，非常简单。

注意C语言中对比字符串是用的`!strcmp(str1, str2)`

```c
int builtin_cmd(char **argv)
{
    if (argv[0] == NULL) {
        // ignore empty line
        return 1;
    }

    if (!strcmp(argv[0], "quit")) {
        // quit
        exit(0);
    }
}
```

### trace03

```
#
# trace03.txt - Run a foreground job.
#
/bin/echo tsh> quit
quit
```

到trace03就要开始实现前台任务的功能了，这里涉及到`eval`、`waitfg`以及`sigchld_handler`。

实现前台的任务，基本的思想先用`fork`是创建一个子进程，然后让父进程
等待当前的子进程结束（`sigsuspend`），直到子进程处理完毕（发送回来`SIGCHLD`），
触发当前的`sigchld_handler`处理回收子进程，然后父进程停止等待，return。

这里有几个关键点：

1. `SIGCHLD`信号的屏蔽

在`fork`之后，需要添加子进程到job队列中。这里因为父子进程并发的原因，
可能**子进程已经退出了**，但是父进程才刚刚加入job队列，开始等待。此时由于子进程
已经退出，`SIGCHLD`信号再也不会发过来，导致**死锁**。

所以父进程必须先屏蔽`SIGCHLD`，直到将子进程加入job队列，然后才开始等待。

> 《深入理解操作系统》8.5.7 显式地等待信号

2. `waitfg`要等待什么？

在`waitfg`函数中，父进程使用`sigsuspend`来先解除`SIGCHLD`的屏蔽，
这样，在`SIGCHLD`信号进来之后，触发`sigchld_handler`的响应。
在`sigchld_handler`中，使用`waitpid`回收了子进程之后，就从job队列中**删除或停止**
该job。

这样，`waitfg`中等待的，其实就是传入的pid代表的job的状态不是`FG`即可。
（删除是`UNDEF`，停止是`ST`。）

```c
void eval(char *cmdline)
{
    char *argv[MAXARGS];
    int bg;
    pid_t pid;

    bg = parseline(cmdline, argv);

    if (builtin_cmd(argv)) {
        return;
    }

    sigset_t sigchld_mask, prev;
    sigemptyset(&sigchld_mask);
    sigaddset(&sigchld_mask, SIGCHLD);
    /* Foreground job */
    if (!bg) {
        // 屏蔽SIGCHLD信号，并把之前的屏蔽集合保存到prev中
        sigprocmask(SIG_BLOCK, &sigchld_mask, &prev);
        if ((pid = Fork()) == 0) { /* child */
            // change child's gid to its pid
            // make child's gid differ from parent's gid
            setpgid(0, 0);
            if (execve(argv[0], argv, environ) < 0) {
                printf("%s: Command not found\n", argv[0]);
                fflush(stdout);
                exit(0);
            }
        }

        /* Parent */
        addjob(jobs, pid, FG, cmdline);
        waitfg(pid);
        return;
    }
}


void waitfg(pid_t pid)
{
    struct job_t *job;
    job = getjobpid(jobs, pid);

    sigset_t empty_mask;
    sigemptyset(&empty_mask);
    while (job->state == FG) {
        // 暂时将屏蔽信号集合设置为空，等待SIGCHLD进来
        sigsuspend(&empty_mask);
    }

    // 等待完成，重置屏蔽信号集合为空
    sigprocmask(SIG_SETMASK, &empty_mask, NULL);
    return;
}

```

### trace04

```
#
# trace04.txt - Run a background job.
#
/bin/echo -e tsh> ./myspin 1 \046
./myspin 1 &
```

后台进程的实现比较简单，我们不需要显式地等待子进程，只需要`fork`
一个子进程，然后加入job队列，打印一条消息即可。

```c

/* background job */
if ((pid = fork()) == 0) {
    setpgid(0, 0);
    if (execve(argv[0], argv, environ) < 0) {
        printf("%s: command not found\n", argv[0]);
        fflush(stdout);
        exit(0);
    }
}

addjob(jobs, pid, bg, cmdline);
printf("[%d] (%d) %s", pid2jid(pid), pid, cmdline);
```

### trace05

```
#
# trace05.txt - Process jobs builtin command.
#
/bin/echo -e tsh> ./myspin 2 \046
./myspin 2 &

/bin/echo -e tsh> ./myspin 3 \046
./myspin 3 &

/bin/echo tsh> jobs
jobs
```

这个trace也比较简单，继续实现`biultin_cmd`：`jobs`即可。
可以直接调用已经实现的`listjobs`的api。

```c
if (!strcmp(argv[0], "jobs")) {
    // listjobs
    listjobs(jobs);
    return 1;
}
```

### trace06

```

#
# trace06.txt - Forward SIGINT to foreground job.
#
/bin/echo -e tsh> ./myspin 4
./myspin 4 

SLEEP 2
INT
```

处理前台进程收到`SIGINT`的情形，这里涉及到`sigint_handler`的实现。

- 这里有几个关键点

1. 收到`SIGINT`信号的是父进程tsh，此时如果子进程不做特殊处理，
那么它们是属于同一个进程组的（`fork`出的子进程默认和父进程同一个进程组）。
此时如果在`sigint_handler`中对信号使用`kill`转发到所有前台的进程组，
那么子进程和父进程都会被kill掉，shell会直接退出。

所以这里要用到`setpgid`函数，通过`setpgid(0,0)`，可以将当前进程的进程组设置为
子进程的pid（和父进程不同），这样再`kill`子进程的时候，就不会影响到父进程了。

> 《深入理解操作系统》8.5.2 发送信号

2. `SIGINT`信号应该由父进程，转发给job队列中，所有状态是前台`FG`的子进程。

```c
void eval(char *cmdline)
{
    ...
    /* Foreground job */
    if (!bg) {
        sigprocmask(SIG_BLOCK, &sigchld_mask, &prev);
        if ((pid = Fork()) == 0) { /* child */
            // change child's gid to its pid
            // make child's gid differ from parent's gid
            setpgid(0, 0);
            if (execve(argv[0], argv, environ) < 0) {
                ...
            }
    }
}

void sigint_handler(int sig)
{
    for (int i = 0; i < MAXJOBS; i++) {
        if (jobs[i].pid != 0 && jobs[i].state == FG) {
            // 将SIGINT转发给所有前台进程（组）
            kill(-jobs[i].pid, sig);
        }
    }
    return;
}
```

### trace07

```
#
# trace07.txt - Forward SIGINT only to foreground job.
#
/bin/echo -e tsh> ./myspin 4 \046
./myspin 4 &

/bin/echo -e tsh> ./myspin 5
./myspin 5 

SLEEP 2
INT

/bin/echo tsh> jobs
jobs
```

同上

### trace08

```

#
# trace08.txt - Forward SIGTSTP only to foreground job.
#
/bin/echo -e tsh> ./myspin 4 \046
./myspin 4 &

/bin/echo -e tsh> ./myspin 5
./myspin 5 

SLEEP 2
TSTP

/bin/echo tsh> jobs
jobs
```

`SIGTSTP`信号的处理和上面`SIGINT`信号的处理完全一致，
转发给所有前台进程组即可。

```c
void sigtstp_handler(int sig)
{
    for (int i = 0; i < MAXJOBS; i++) {
        if (jobs[i].pid != 0 && jobs[i].state == FG) {
            // 将SIGTSTP转发给所有前台进程（组）
            kill(-jobs[i].pid, sig);
        }
    }
    return;
}
```

### trace09

```
#
# trace09.txt - Process bg builtin command
#
/bin/echo -e tsh> ./myspin 4 \046
./myspin 4 &

/bin/echo -e tsh> ./myspin 5
./myspin 5 

SLEEP 2
TSTP

/bin/echo tsh> jobs
jobs

/bin/echo tsh> bg %2
bg %2

/bin/echo tsh> jobs
jobs
```

开始处理`bg`和`fg`命令。`bg`是把前台的命令放到后台，并发送`SIGCONT`信号，
让其在后台继续运行。

有了前面的基础，这个逻辑比较简单了。

```c
void do_bgfg(char **argv)
{
    ...

    if (!strcmp(argv[0], "bg")) {
        job->state = BG;
        kill(-job->pid, SIGCONT);
        printf("[%d] (%d) %s", jid, job->pid, job->cmdline);
    }
    
    ...
}
```

### trace10

```
#
# trace10.txt - Process fg builtin command. 
#
/bin/echo -e tsh> ./myspin 4 \046
./myspin 4 &

SLEEP 1
/bin/echo tsh> fg %1
fg %1

SLEEP 1
TSTP

/bin/echo tsh> jobs
jobs

/bin/echo tsh> fg %1
fg %1

/bin/echo tsh> jobs
jobs

```

`fg`是把一个后台的任务放到前台，然后同样发送`SIGCONT`信号，让它继续执行。

这个和`bg`的区别是，前台的任务需要父进程等待。

这里有两个关键点：

1. 后台进程转到前台后，需要处理`SIGINT`和`SIGTSTP`信号。

因为后台进程需要屏蔽`SIGINT`和`SIGTSTP`信号，所以刚开始我参考了csapp
书本附录代码`shell.c`的实现，采用后台子进程`fork`之后就屏蔽这两个信号的做法。

但是这样一来，需要用`fg`切换到前台的时候，就没办法实现信号的解除屏蔽了。
因为这时的逻辑在父进程这里，父进程没办法改变子进程的信号状态。

经过一段时间的研究，最后想到了使用`tsh.c`本身提供的job的状态属性，来只转发`SIGINT`
和`SIGTSTP`给前台进程，解决了这个麻烦。

2. 父进程需要等待前台进程

继续调用`waitfg`来等待。

```c
void do_bgfg(char **argv)
{
    ...

    if (!strcmp(argv[0], "bg")) {

        ...

    } else {
        job->state = FG;
        kill(-job->pid, SIGCONT);
        waitfg(job->pid);
    }

    ...
}
```

### trace11

```
#
# trace11.txt - Forward SIGINT to every process in foreground process group
#
/bin/echo -e tsh> ./mysplit 4
./mysplit 4 

SLEEP 2
INT

/bin/echo tsh> /bin/ps a
/bin/ps a

```

这个已经实现了。

### trace11

```
#
# trace12.txt - Forward SIGTSTP to every process in foreground process group
#
/bin/echo -e tsh> ./mysplit 4
./mysplit 4 

SLEEP 2
TSTP

/bin/echo tsh> jobs
jobs

/bin/echo tsh> /bin/ps a
/bin/ps a



```

同上

### trace13

```
#
# trace13.txt - Restart every stopped process in process group
#
/bin/echo -e tsh> ./mysplit 4
./mysplit 4 

SLEEP 2
TSTP

/bin/echo tsh> jobs
jobs

/bin/echo tsh> /bin/ps a
/bin/ps a

/bin/echo tsh> fg %1
fg %1

/bin/echo tsh> /bin/ps a
/bin/ps a

```

已经实现了

### trace14

```
#
# trace14.txt - Simple error handling
#
/bin/echo tsh> ./bogus
./bogus

/bin/echo -e tsh> ./myspin 4 \046
./myspin 4 &

/bin/echo tsh> fg
fg

/bin/echo tsh> bg
bg

/bin/echo tsh> fg a
fg a

/bin/echo tsh> bg a
bg a

/bin/echo tsh> fg 9999999
fg 9999999

/bin/echo tsh> bg 9999999
bg 9999999

/bin/echo tsh> fg %2
fg %2

/bin/echo tsh> fg %1
fg %1

SLEEP 2
TSTP

/bin/echo tsh> bg %2
bg %2

/bin/echo tsh> bg %1
bg %1

/bin/echo tsh> jobs
jobs
```

简单的bgfg命令的错误处理，不再赘述。

```c
void do_bgfg(char **argv)
{
    int jid;
    pid_t pid;
    struct job_t *job;

    if (argv[1] == NULL) {
        printf("%s command requires PID or %%jobid argument\n", argv[0]);
        return;
    } else if (!(argv[1][0] == '%' || (argv[1][0] >= '0' && argv[1][0] <= '9'))) {
        printf("%s: argument must be a PID or %%jobid\n", argv[0]);
        return;
    }

    if (argv[1][0] == '%') {
        // jid
        jid = atoi(argv[1] + 1);
        job = getjobjid(jobs, jid);
        if (job == NULL) {
            printf("%s: No such job\n", argv[1]);
            return;
        }
    } else {
        // pid
        pid = atoi(argv[1]);
        job = getjobpid(jobs, pid);
        if (job == NULL) {
            printf("(%d): No such process\n", pid);
            return;
        }
    }

    if (!strcmp(argv[0], "bg")) {
        job->state = BG;
        kill(-job->pid, SIGCONT);
        printf("[%d] (%d) %s", jid, job->pid, job->cmdline);
    } else {
        job->state = FG;
        kill(-job->pid, SIGCONT);
        waitfg(job->pid);
    }

    return;
}
```

### trace15

```
#
# trace15.txt - Putting it all together
#

/bin/echo tsh> ./bogus
./bogus

/bin/echo tsh> ./myspin 10
./myspin 10

SLEEP 2
INT

/bin/echo -e tsh> ./myspin 3 \046
./myspin 3 &

/bin/echo -e tsh> ./myspin 4 \046
./myspin 4 &

/bin/echo tsh> jobs
jobs

/bin/echo tsh> fg %1
fg %1

SLEEP 2
TSTP

/bin/echo tsh> jobs
jobs

/bin/echo tsh> bg %3
bg %3

/bin/echo tsh> bg %1
bg %1

/bin/echo tsh> jobs
jobs

/bin/echo tsh> fg %1
fg %1

/bin/echo tsh> quit
quit

```

全部放在一起测试

### trace16

```
#
# trace16.txt - Tests whether the shell can handle SIGTSTP and SIGINT
#     signals that come from other processes instead of the terminal.
#

/bin/echo tsh> ./mystop 2 
./mystop 2

SLEEP 3

/bin/echo tsh> jobs
jobs

/bin/echo tsh> ./myint 2 
./myint 2

```

已经实现。

## sigchld_handler

`sigchld_handler`的实现差不多是最复杂的，所以单独讲讲。

它要实现的功能是

1. 回收子进程

2. 判断子进程的状态，对应修改job队列

这里的关键点是`waitpid`参数的设置

```
pid_t Waitpid(pid_t pid, int *iptr, int options);
```

第一个参数`pid`设置成-1，代表父进程等待回收所有的子进程（包括前台进程和后台进程）

第二个参数`iptr`为回收状态，需要传入来获取子进程是停止的（`SIGTSTP`）还是终止的
（`SIGINT`或正常退出）。

第三个参数`options`代表默认行为，`WUNTRACED | WNOHANG`代表非阻塞，如果子进程都没有
停止或终止，就立即返回0；否则就返回停止或终止的进程pid。

这里主要的疑问是options为什么不能设置成`WUNTRACED`，而必须是`WUNTRACED | WNOHANG`。

```
子进程状态变化的时候会发送SIGCHLD信号。如果有批量的子进程同时结束，SIGCHLD信号顶多只会
处理一个，保存一个。

所以在回收子进程的时候，需要父进程在收到`SIGCHLD`信号的时候，使用while循环，
尽可能多地回收子进程。同时要用这个options，不要阻塞主进程。
```

> 《深入理解操作系统》8.4.3 回收子进程

```c
void sigchld_handler(int sig)
{
    int status;
    int child_pid;

    // WUNTRACED | WNOHANG：立即返回。如果等待集合中的子进程都没有终止，则返回0
    //                      否则，返回停止或终止的那个子进程的pid。
    while ((child_pid = waitpid(-1, &status, WUNTRACED | WNOHANG)) > 0) {
        // 被停止了，SIGTSTP
        if (WIFSTOPPED(status)) {
            printf("Job [%d] (%d) stopped by signal %d\n", pid2jid(child_pid), child_pid, SIGTSTP);
            struct job_t *job = getjobpid(jobs, child_pid);
            job->state = ST;
        } else {
            // 被终止了，SIGINT
            if (WIFSIGNALED(status)) {
                printf("Job [%d] (%d) terminated by signal %d\n", pid2jid(child_pid), child_pid, SIGINT);
            }
            deletejob(jobs, child_pid);
        }
    }
}
```

## 完整源码

```c
/*
 * tsh - A tiny shell program with job control
 *
 * <Put your name and login ID here>
 */
#include <ctype.h>
#include <errno.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

/* Misc manifest constants */
#define MAXLINE 1024   /* max line size */
#define MAXARGS 128    /* max args on a command line */
#define MAXJOBS 16     /* max jobs at any point in time */
#define MAXJID 1 << 16 /* max job ID */

/* Job states */
#define UNDEF 0 /* undefined */
#define FG 1    /* running in foreground */
#define BG 2    /* running in background */
#define ST 3    /* stopped */

/*
 * Jobs states: FG (foreground), BG (background), ST (stopped)
 * Job state transitions and enabling actions:
 *     FG -> ST  : ctrl-z
 *     ST -> FG  : fg command
 *     ST -> BG  : bg command
 *     BG -> FG  : fg command
 * At most 1 job can be in the FG state.
 */

/* Global variables */
extern char **environ;   /* defined in libc */
char prompt[] = "tsh> "; /* command line prompt (DO NOT CHANGE) */
int verbose = 0;         /* if true, print additional output */
int nextjid = 1;         /* next job ID to allocate */
char sbuf[MAXLINE];      /* for composing sprintf messages */

struct job_t {             /* The job struct */
    pid_t pid;             /* job PID */
    int jid;               /* job ID [1, 2, ...] */
    int state;             /* UNDEF, BG, FG, or ST */
    char cmdline[MAXLINE]; /* command line */
};
struct job_t jobs[MAXJOBS]; /* The job list */
/* End global variables */

/* Function prototypes */

/* Here are the functions that you will implement */
void eval(char *cmdline);
int builtin_cmd(char **argv);
void do_bgfg(char **argv);
void waitfg(pid_t pid);

void sigchld_handler(int sig);
void sigtstp_handler(int sig);
void sigint_handler(int sig);

/* Here are helper routines that we've provided for you */
int parseline(const char *cmdline, char **argv);
void sigquit_handler(int sig);

void clearjob(struct job_t *job);
void initjobs(struct job_t *jobs);
int maxjid(struct job_t *jobs);
int addjob(struct job_t *jobs, pid_t pid, int state, char *cmdline);
int deletejob(struct job_t *jobs, pid_t pid);
pid_t fgpid(struct job_t *jobs);
struct job_t *getjobpid(struct job_t *jobs, pid_t pid);
struct job_t *getjobjid(struct job_t *jobs, int jid);
int pid2jid(pid_t pid);
void listjobs(struct job_t *jobs);

void usage(void);
void unix_error(char *msg);
void app_error(char *msg);
typedef void handler_t(int);
handler_t *Signal(int signum, handler_t *handler);

/* Wrappered system calls */
pid_t Fork(void);

/*
 * main - The shell's main routine
 */
int main(int argc, char **argv)
{
    char c;
    char cmdline[MAXLINE];
    int emit_prompt = 1; /* emit prompt (default) */

    /* Redirect stderr to stdout (so that driver will get all output
     * on the pipe connected to stdout) */
    dup2(1, 2);

    /* Parse the command line */
    while ((c = getopt(argc, argv, "hvp")) != EOF) {
        switch (c) {
        case 'h': /* print help message */
            usage();
            break;
        case 'v': /* emit additional diagnostic info */
            verbose = 1;
            break;
        case 'p':            /* don't print a prompt */
            emit_prompt = 0; /* handy for automatic testing */
            break;
        default:
            usage();
        }
    }

    /* Install the signal handlers */

    /* These are the ones you will need to implement */
    Signal(SIGINT, sigint_handler);   /* ctrl-c */
    Signal(SIGTSTP, sigtstp_handler); /* ctrl-z */
    Signal(SIGCHLD, sigchld_handler); /* Terminated or stopped child */

    /* This one provides a clean way to kill the shell */
    Signal(SIGQUIT, sigquit_handler);

    /* Initialize the job list */
    initjobs(jobs);

    /* Execute the shell's read/eval loop */
    while (1) {

        /* Read command line */
        if (emit_prompt) {
            printf("%s", prompt);
            fflush(stdout);
        }
        if ((fgets(cmdline, MAXLINE, stdin) == NULL) && ferror(stdin))
            app_error("fgets error");
        if (feof(stdin)) { /* End of file (ctrl-d) */
            fflush(stdout);
            exit(0);
        }

        /* Evaluate the command line */
        eval(cmdline);
        fflush(stdout);
        fflush(stdout);
    }

    exit(0); /* control never reaches here */
}

/*
 * eval - Evaluate the command line that the user has just typed in
 *
 * If the user has requested a built-in command (quit, jobs, bg or fg)
 * then execute it immediately. Otherwise, fork a child process and
 * run the job in the context of the child. If the job is running in
 * the foreground, wait for it to terminate and then return.  Note:
 * each child process must have a unique process group ID so that our
 * background children don't receive SIGINT (SIGTSTP) from the kernel
 * when we type ctrl-c (ctrl-z) at the keyboard.
 */
void eval(char *cmdline)
{
    char *argv[MAXARGS];
    int bg;
    pid_t pid;

    bg = parseline(cmdline, argv);

    if (builtin_cmd(argv)) {
        return;
    }

    sigset_t sigchld_mask, prev;
    sigemptyset(&sigchld_mask);
    sigaddset(&sigchld_mask, SIGCHLD);
    /* Foreground job */
    if (!bg) {
        // 屏蔽SIGCHLD信号，并把之前的屏蔽集合保存到prev中
        sigprocmask(SIG_BLOCK, &sigchld_mask, &prev);
        if ((pid = Fork()) == 0) { /* child */
            // change child's gid to its pid
            // make child's gid differ from parent's gid
            setpgid(0, 0);
            if (execve(argv[0], argv, environ) < 0) {
                printf("%s: Command not found\n", argv[0]);
                fflush(stdout);
                exit(0);
            }
        }

        /* Parent */
        addjob(jobs, pid, FG, cmdline);
        waitfg(pid);
        return;
    }

    /* Background job */
    if ((pid = Fork()) == 0) {
        setpgid(0, 0);
        if (execve(argv[0], argv, environ) < 0) {
            printf("%s: Command not found\n", argv[0]);
            fflush(stdout);
            exit(0);
        }
    }

    addjob(jobs, pid, BG, cmdline);
    printf("[%d] (%d) %s", pid2jid(pid), pid, cmdline);
}

/*
 * parseline - Parse the command line and build the argv array.
 *
 * Characters enclosed in single quotes are treated as a single
 * argument.  Return true if the user has requested a BG job, false if
 * the user has requested a FG job.
 */
int parseline(const char *cmdline, char **argv)
{
    static char array[MAXLINE]; /* holds local copy of command line */
    char *buf = array;          /* ptr that traverses command line */
    char *delim;                /* points to first space delimiter */
    int argc;                   /* number of args */
    int bg;                     /* background job? */

    strcpy(buf, cmdline);
    buf[strlen(buf) - 1] = ' ';   /* replace trailing '\n' with space */
    while (*buf && (*buf == ' ')) /* ignore leading spaces */
        buf++;

    /* Build the argv list */
    argc = 0;
    if (*buf == '\'') {
        buf++;
        delim = strchr(buf, '\'');
    } else {
        delim = strchr(buf, ' ');
    }

    while (delim) {
        argv[argc++] = buf;
        *delim = '\0';
        buf = delim + 1;
        while (*buf && (*buf == ' ')) /* ignore spaces */
            buf++;

        if (*buf == '\'') {
            buf++;
            delim = strchr(buf, '\'');
        } else {
            delim = strchr(buf, ' ');
        }
    }
    argv[argc] = NULL;

    if (argc == 0) /* ignore blank line */
        return 1;

    /* should the job run in the background? */
    if ((bg = (*argv[argc - 1] == '&')) != 0) {
        argv[--argc] = NULL;
    }
    return bg;
}

/*
 * builtin_cmd - If the user has typed a built-in command then execute
 *    it immediately.
 */
int builtin_cmd(char **argv)
{
    if (argv[0] == NULL) {
        // ignore empty line
        return 1;
    }

    if (!strcmp(argv[0], "quit")) {
        // quit
        exit(0);
    }

    if (!strcmp(argv[0], "jobs")) {
        // listjobs
        listjobs(jobs);
        return 1;
    }

    if (!strcmp(argv[0], "bg") || !strcmp(argv[0], "fg")) {
        // bg & fg
        do_bgfg(argv);
        return 1;
    }

    return 0;
}

/*
 * do_bgfg - Execute the builtin bg and fg commands
 */
void do_bgfg(char **argv)
{
    int jid;
    pid_t pid;
    struct job_t *job;

    if (argv[1] == NULL) {
        printf("%s command requires PID or %%jobid argument\n", argv[0]);
        return;
    } else if (!(argv[1][0] == '%' || (argv[1][0] >= '0' && argv[1][0] <= '9'))) {
        printf("%s: argument must be a PID or %%jobid\n", argv[0]);
        return;
    }

    if (argv[1][0] == '%') {
        // jid
        jid = atoi(argv[1] + 1);
        job = getjobjid(jobs, jid);
        if (job == NULL) {
            printf("%s: No such job\n", argv[1]);
            return;
        }
    } else {
        // pid
        pid = atoi(argv[1]);
        job = getjobpid(jobs, pid);
        if (job == NULL) {
            printf("(%d): No such process\n", pid);
            return;
        }
    }

    if (!strcmp(argv[0], "bg")) {
        job->state = BG;
        kill(-job->pid, SIGCONT);
        printf("[%d] (%d) %s", jid, job->pid, job->cmdline);
    } else {
        job->state = FG;
        kill(-job->pid, SIGCONT);
        waitfg(job->pid);
    }

    return;
}

/*
 * waitfg - Block until process pid is no longer the foreground process
 */
void waitfg(pid_t pid)
{
    struct job_t *job;
    job = getjobpid(jobs, pid);

    sigset_t empty_mask;
    sigemptyset(&empty_mask);
    while (job->state == FG) {
        // 暂时将屏蔽信号集合设置为空，等待SIGCHLD进来
        sigsuspend(&empty_mask);
    }

    // 等待完成，重置屏蔽信号集合为空
    sigprocmask(SIG_SETMASK, &empty_mask, NULL);
    return;
}

/*****************
 * Signal handlers
 *****************/

/*
 * sigchld_handler - The kernel sends a SIGCHLD to the shell whenever
 *     a child job terminates (becomes a zombie), or stops because it
 *     received a SIGSTOP or SIGTSTP signal. The handler reaps all
 *     available zombie children, but doesn't wait for any other
 *     currently running children to terminate.
 */
void sigchld_handler(int sig)
{
    int status;
    int child_pid;

    // WUNTRACED | WNOHANG：立即返回。如果等待集合中的子进程都没有终止，则返回0
    //                      否则，返回停止或终止的那个子进程的pid。
    while ((child_pid = waitpid(-1, &status, WUNTRACED | WNOHANG)) > 0) {
        // 被停止了，SIGTSTP
        if (WIFSTOPPED(status)) {
            printf("Job [%d] (%d) stopped by signal %d\n", pid2jid(child_pid), child_pid, SIGTSTP);
            struct job_t *job = getjobpid(jobs, child_pid);
            job->state = ST;
        } else {
            // 被终止了，SIGINT
            if (WIFSIGNALED(status)) {
                printf("Job [%d] (%d) terminated by signal %d\n", pid2jid(child_pid), child_pid, SIGINT);
            }
            deletejob(jobs, child_pid);
        }
    }
}

/*
 * sigint_handler - The kernel sends a SIGINT to the shell whenver the
 *    user types ctrl-c at the keyboard.  Catch it and send it along
 *    to the foreground job.
 */
void sigint_handler(int sig)
{
    for (int i = 0; i < MAXJOBS; i++) {
        if (jobs[i].pid != 0 && jobs[i].state == FG) {
            // 将SIGINT转发给所有前台进程（组）
            kill(-jobs[i].pid, sig);
        }
    }
    return;
}

/*
 * sigtstp_handler - The kernel sends a SIGTSTP to the shell whenever
 *     the user types ctrl-z at the keyboard. Catch it and suspend the
 *     foreground job by sending it a SIGTSTP.
 */
void sigtstp_handler(int sig)
{
    for (int i = 0; i < MAXJOBS; i++) {
        if (jobs[i].pid != 0 && jobs[i].state == FG) {
            // 将SIGTSTP转发给所有前台进程（组）
            kill(-jobs[i].pid, sig);
        }
    }
    return;
}

/*********************
 * End signal handlers
 *********************/

/***********************************************
 * Helper routines that manipulate the job list
 **********************************************/

/* clearjob - Clear the entries in a job struct */
void clearjob(struct job_t *job)
{
    job->pid = 0;
    job->jid = 0;
    job->state = UNDEF;
    job->cmdline[0] = '\0';
}

/* initjobs - Initialize the job list */
void initjobs(struct job_t *jobs)
{
    int i;

    for (i = 0; i < MAXJOBS; i++)
        clearjob(&jobs[i]);
}

/* maxjid - Returns largest allocated job ID */
int maxjid(struct job_t *jobs)
{
    int i, max = 0;

    for (i = 0; i < MAXJOBS; i++)
        if (jobs[i].jid > max)
            max = jobs[i].jid;
    return max;
}

/* addjob - Add a job to the job list */
int addjob(struct job_t *jobs, pid_t pid, int state, char *cmdline)
{
    int i;

    if (pid < 1)
        return 0;

    for (i = 0; i < MAXJOBS; i++) {
        if (jobs[i].pid == 0) {
            jobs[i].pid = pid;
            jobs[i].state = state;
            jobs[i].jid = nextjid++;
            if (nextjid > MAXJOBS)
                nextjid = 1;
            strcpy(jobs[i].cmdline, cmdline);
            if (verbose) {
                printf("Added job [%d] %d %s\n", jobs[i].jid, jobs[i].pid, jobs[i].cmdline);
            }
            return 1;
        }
    }
    printf("Tried to create too many jobs\n");
    return 0;
}

/* deletejob - Delete a job whose PID=pid from the job list */
int deletejob(struct job_t *jobs, pid_t pid)
{
    int i;

    if (pid < 1)
        return 0;

    for (i = 0; i < MAXJOBS; i++) {
        if (jobs[i].pid == pid) {
            clearjob(&jobs[i]);
            nextjid = maxjid(jobs) + 1;
            return 1;
        }
    }
    return 0;
}

/* fgpid - Return PID of current foreground job, 0 if no such job */
pid_t fgpid(struct job_t *jobs)
{
    int i;

    for (i = 0; i < MAXJOBS; i++)
        if (jobs[i].state == FG)
            return jobs[i].pid;
    return 0;
}

/* getjobpid  - Find a job (by PID) on the job list */
struct job_t *getjobpid(struct job_t *jobs, pid_t pid)
{
    int i;

    if (pid < 1)
        return NULL;
    for (i = 0; i < MAXJOBS; i++)
        if (jobs[i].pid == pid)
            return &jobs[i];
    return NULL;
}

/* getjobjid  - Find a job (by JID) on the job list */
struct job_t *getjobjid(struct job_t *jobs, int jid)
{
    int i;

    if (jid < 1)
        return NULL;
    for (i = 0; i < MAXJOBS; i++)
        if (jobs[i].jid == jid)
            return &jobs[i];
    return NULL;
}

/* pid2jid - Map process ID to job ID */
int pid2jid(pid_t pid)
{
    int i;

    if (pid < 1)
        return 0;
    for (i = 0; i < MAXJOBS; i++)
        if (jobs[i].pid == pid) {
            return jobs[i].jid;
        }
    return 0;
}

/* listjobs - Print the job list */
void listjobs(struct job_t *jobs)
{
    int i;

    for (i = 0; i < MAXJOBS; i++) {
        if (jobs[i].pid != 0) {
            printf("[%d] (%d) ", jobs[i].jid, jobs[i].pid);
            switch (jobs[i].state) {
            case BG:
                printf("Running ");
                break;
            case FG:
                printf("Foreground ");
                break;
            case ST:
                printf("Stopped ");
                break;
            default:
                printf("listjobs: Internal error: job[%d].state=%d ", i, jobs[i].state);
            }
            printf("%s", jobs[i].cmdline);
        }
    }
}
/******************************
 * end job list helper routines
 ******************************/

/***********************
 * Other helper routines
 ***********************/

/*
 * usage - print a help message
 */
void usage(void)
{
    printf("Usage: shell [-hvp]\n");
    printf("   -h   print this message\n");
    printf("   -v   print additional diagnostic information\n");
    printf("   -p   do not emit a command prompt\n");
    exit(1);
}

/*
 * unix_error - unix-style error routine
 */
void unix_error(char *msg)
{
    fprintf(stdout, "%s: %s\n", msg, strerror(errno));
    exit(1);
}

/*
 * app_error - application-style error routine
 */
void app_error(char *msg)
{
    fprintf(stdout, "%s\n", msg);
    exit(1);
}

/*
 * Signal - wrapper for the sigaction function
 */
handler_t *Signal(int signum, handler_t *handler)
{
    struct sigaction action, old_action;

    action.sa_handler = handler;
    sigemptyset(&action.sa_mask); /* block sigs of type being handled */
    action.sa_flags = SA_RESTART; /* restart syscalls if possible */

    if (sigaction(signum, &action, &old_action) < 0)
        unix_error("Signal error");
    return (old_action.sa_handler);
}

/*
 * sigquit_handler - The driver program can gracefully terminate the
 *    child shell by sending it a SIGQUIT signal.
 */
void sigquit_handler(int sig)
{
    printf("Terminating after receipt of SIGQUIT signal\n");
    exit(1);
}

/* wrapper functions */
pid_t Fork(void)
{
    pid_t pid;

    if ((pid = fork()) < 0)
        unix_error("Fork error");
    return pid;
}
```
