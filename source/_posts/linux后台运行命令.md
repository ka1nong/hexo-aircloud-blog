---
title: linux后台运行命令&
date: 2018-07-27 10:10:33
tags:
---
<h2>程序设置后台运行</h2>

在linux下部署服务，设置后台运行是不可或缺的一个步骤，要想服务脱离终端运行，使终端关闭之后程序依旧继续跑，则需要设置后台运行。c语言代码如下：

	```c++
    void init_daemon(void)
    {
        int i,pid;
        if ((pid = fork()))
        {
            exit(0); // 是父进程，结束父进程
        }
        else if (pid < 0)
        {
            writeLog(WLOG_WARN,"[init_daemon] fork fail. %s",strerror(errno));
            exit(1);// fork失败，退出
            // 是第一子进程，后台继续执行
        }

        setsid(); // 第一子进程成为新的会话组长和进程组长
        // 并与控制终端分离
        if ((pid = fork()))
        {
            exit(0); // 是第一子进程，结束第一子进程
        }
        else if (pid< 0)
        {
            writeLog(WLOG_WARN,"[init_daemon] fork fail. %s",strerror(errno));
            exit(1); // fork失败，退出
            // 是第二子进程，继续
            // 第二子进程不再是会话组长
        }

        for (i = 0; i < 255; i++)
            close(i);
        //chdir("/tmp");//改变工作目录到/tmp
        umask(0);//重设文件创建掩模
        return;
    }
    ```
    
* 在后台运行 
为避免挂起控制终端将Daemon放入后台执行。方法是在进程中调用fork使父进程终止，让Daemon在子进程中后台执行。

	```c++
    if (pid = fork()) 
    	exit(0); // 是父进程，结束父进程，子进程继续
	```

* 脱离控制终端，登录会话和进程组 
有必要先介绍一下Linux中的进程与控制终端，登录会话和进程组之间的关系：进程属于一个进程组，进程组号（GID）就是进程组长的进程号（PID）。登录会话可以包含多个进程组。这些进程组共享一个控制终端。这个控制终端通常是创建进程的登录终端。
控制终端，登录会话和进程组通常是从父进程继承下来的。我们的目的就是要摆脱它们，使之不受它们的影响。方法是在第1点的基础上，调用setsid()使进程成为会话组长：

	```c++
    setsid(); // 说明：当进程是会话组长时setsid()调用失败。但第一点已经保证进程不是会话组长。setsid()调用成功后，进程成为新的会话组长和新的进程组长，并与原来的登录会话和进程组脱离。由于会话过程对控制终端的独占性，进程同时与控制终端脱离。
    ```

* 禁止进程重新打开控制终端 
现在，进程已经成为无终端的会话组长。但它可以重新申请打开一个控制终端。可以通过使进程不再成为会话组长来禁止进程重新打开控制终端：

	```c++
	if(pid=fork()) 
	exit(0);//结束第一子进程，第二子进程继续（第二子进程不再是会话组长） 
	```

* 关闭打开的文件描述符 
进程从创建它的父进程那里继承了打开的文件描述符。如不关闭，将会浪费系统资源，造成进程所在的文件系统无法卸下以及引起无法预料的错误。按如下方法关闭它们：
      
    ```c++
    for(i=0;i 关闭打开的文件描述符close(i);> 
    for(i=0;i< NOFILE;++i)
    ```
     
* 改变当前工作目录 
进程活动时，其工作目录所在的文件系统不能卸下。一般需要将工作目录改变到根目录。对于需要转储核心，写运行日志的进程将工作目录改变到特定目录如/tmpchdir("/")

* 重设文件创建掩模 
进程从创建它的父进程那里继承了文件创建掩模。它可能修改守护进程所创建的文件的存取位。为防止这一点，将文件创建掩模清除：umask(0); 

* 处理SIGCHLD信号 
处理SIGCHLD信号并不是必须的。但对于某些进程，特别是服务器进程往往在请求到来时生成子进程处理请求。如果父进程不等待子进程结束，子进程将成为 僵尸进程（zombie）从而占用系统资源。如果父进程等待子进程结束，将增加父进程的负担，影响服务器进程的并发性能。在Linux下可以简单地将 SIGCHLD信号的操作设为SIG_IGN。
 	```c++
	signal(SIGCHLD,SIG_IGN); // 这样，内核在子进程结束时不会产生僵尸进程。这一点与BSD4不同，BSD4下必须显式等待子进程结束才能释放僵尸进程。
    ```

* 按照如上设置过的程序，其父进程为1，即init进程

    ```c++
    [m_tgame@cfm-devnet-05 ~]$ ps -eFH |grep dsagent
    m_tgame   8810  8596  0  2373   728  19 10:55 pts/12   00:00:00           grep dsagent
    m_tgame  30656     1  0 48288 85468  11 Jul19 ?        00:04:18   ./dsagent --id=3.3.0.28  // 第三列即是父进程
    ```
原文链接：https://blog.csdn.net/whatday/article/details/78633702

<h2>命令设置后台运行</h2>

在终端中执行的程序，当终端退出时程序会停止运行，如下所示：

这个blog网站是基于node+hexo搭建起来的。当我需要部署时，敲入命令：
	```c++
	>sudo hexo s -p 80
	```
新开一个终端查看hexo的进程id和父进程id，敲入命令:
	```c++
	> ps -eFH |grep hexo
	```
输出如下：
	```c++
    root     11610 10463  0 15918  2124   0 10:29 pts/0    00:00:00           sudo hexo s -p 80
    root     11611 11610 22 307902 60956  0 10:29 pts/0    00:00:00             hexo                      
    ```
第二列是当前进程id，第三列是父进程的id。如上所示，10463是bash的进程id，hexo进程的父进程是11610,11610的父进程是10463.这个时候我关闭10463终端，再查看一次hexo的进程如下：
	
   	```c++
	> ps -eFH |grep hexo
	ubuntu   11897 11574  0  2617   928   0 10:34 pts/1    00:00:00           grep --color=auto hexo
	```

这个时候hexo进程退出。

&是常用的设置后台运行的命令，但是通过&设置的后台程序其父进程还是当前终端shell的进程，而一旦父进程退出，则会发送hangup信号给所有子进程，子进程收到hangup以后也会退出，如下所示：

启动brpc下面的echo_server例子
	```c++
	>  ./echo_server &
	```
查看信息如下：

	```c++
    ubuntu@VM-16-9-ubuntu:~$ ps -eFH |grep echo_server
    ubuntu   24002 22596  0 169216 22164  0 11:06 pts/1    00:00:00           ./echo_server
    ubuntu   24028 23967  0  2617   932   0 11:07 pts/3    00:00:00           grep --color=auto echo_server
    ubuntu@VM-16-9-ubuntu:~$ ps -eFH |grep bash
    ubuntu   22596 22595  0  5448  4456   0 10:35 pts/1    00:00:00         -bash
    ubuntu   23967 22595  0  5423  4140   0 11:06 pts/3    00:00:00         -bash
    ubuntu   24088 23967  0  2617   928   0 11:08 pts/3    00:00:00           grep --color=auto bas
	```
从上可以看出，echo_server的父进程是bash,这时我退出这个bash,查看信息如下：

	```c++
    ubuntu@VM-16-9-ubuntu:~$ ps -eFH |grep echo_server
    ubuntu   24150 23967  0  2617   928   0 11:09 pts/3    00:00:00           grep --color=auto echo_server
    ubuntu@VM-16-9-ubuntu:~$ ps -eFH |grep bash
    ubuntu   23967 22595  0  5423  4144   0 11:06 pts/3    00:00:00         -bash
    ubuntu   24153 23967  0  2617   924   0 11:09 pts/3    00:00:00           grep --color=auto bash
    ```
echo_server退出，如果我们要在退出shell的时候继续运行进程，则需要使用nohup忽略hangup信号，或者setsid将将父进程设为init进程(进程号为1)

    ```c++
    >nohup ./rsync.sh & 这种办法在我的机器上运行不起来
    ```

除了上面那种方法，还有一个是使用sudo命令 重新打开一个新的终端，加上后台运行命令&
	 ```c++
	> sudo hexo s -p 80 &
	```
    
查看进程
	```c++
	> ps -eFH |grep hexo
	输出：
    ubuntu   12028 11574  0  2617   928   0 10:36 pts/1    00:00:00           grep --color=auto hexo
    root     12015 11965  0 15918  2124   0 10:36 pts/0    00:00:00           sudo hexo s -p 80
    root     12016 12015 22 307924 61464  0 10:36 pts/0    00:00:01             hexo  // 父进程退依然是终端
    ```
这个时候我再关闭这个终端，再查看进程，输出如下：
	```c++
    >ps -eFH |grep hexo
    输出：
    ubuntu   12119 11574  0  2617   924   0 10:37 pts/1    00:00:00           grep --color=auto hexo
    root     12015     1  0 15918  2124   0 10:36 ?        00:00:00   sudo hexo s -p 80
    root     12016 12015  1 308436 56504  0 10:36 ?        00:00:01     hexo
    ```
这时可以看到 12015的父进程id从11965变成了1，也就是hexo进程由init进程领养了，这样终端退出之后也不会有问题。这里为什么是领养而不是退出，还不得而知
