[Writeup](http://csapp.cs.cmu.edu/3e/shlab.pdf)
# 目的

熟悉进程控制和信号机制


# 实现部分

初始工程仅仅实现了shell的架子，有一些重要的方法没有实现:

* eval 解析和解释命令行的主要方法 70line
* builtin_cmd 识别和解释命令行,quit,fg,bg,jobs 25line
* do_bgfg 实现bg和fg  50line
* waitfg 等待前台job完成 20line
* sigchld_handler 捕获SIGCHILD信号 80line
* sigint_handler 捕获SIGINT(ctrl-c)信号  15line
* sigstp_handler 捕获SIGTSTP(ctrl-z)信号 15line

每次修改tsh.c文件后，通过`make`命令重新编译，然后执行`make test0X`

# Hints
* 阅读第八章
* 先从trace01.txt开始，和trshref.out输出一样，再移动到trace02.txt
* waitpid, kill , fork, execve,setpgid , sigprocmask是非常有用的,waitpid里面的WUNTRACED和WNOHANG参数页很有用.
* 在实现signal handler方法时，使用-pid而不是pid参数，确保SIGINT和SIGTSTP信号是发送给前台进程组
* 分配的棘手部分之一是确定waitfg和sigchld处理函数之间的工作分配。
	* 在waitfg方法里，使用sleep频繁睡眠
	* 在sigchld_handler中，使用waitpid.
* 在eval里面，为了避免竞争，父进程需要在addjob之前阻塞SIGCHLD，之后再unblock, 子进程由于继承了blocked向量，所以执行新进程之前也要unblock.
* 当您从标准Unix shell运行shell时，shell在前台进程组中运行。
如果您的shell随后创建了一个子进程，则默认情况下，该子进程也将成为前台进程组的成员。
由于输入ctrl-c会将SIGINT发送到前台组中的每个进程，因此输入ctrl-c会将SIGINT发送到您的shell以及您的shell创建的每个进程，这显然是不正确的
	* 解决方法是：在fork之后，但在execve之前，子进程应调用setpgid（0，0），这会将子进程放入一个新的进程组中，该组的组ID与该子进程的PID相同。
这样可以确保在前台进程组中只有一个进程，即Shell。
键入ctrl-c时，shell应捕获所得的SIGINT，然后将其转发到适当的前台作业（或更准确地说，是包含前台作业的进程组）。


<font color=red>hint第五点提示非常重要，最开始做任务的时候，在waitfg和sigchld_handler都通过waitpid来回收进程，结果各种bug.

SIGINT和SIGTSTP这两个信号的行为要么是终止进程，要么是暂停进程, 它们最终都会被sigchld_handler捕获.所以sigint_handler和sigtstp_handler里面都只是简单的发送信号, 具体的逻辑放在sigchld_handler里面，通过WIFEXITED等宏可以判断出当前进程的退出状态，然后执行相应的逻辑. 至于waitfg里面，我们简单的通过while sleep状态的方法就可以实现.
</font>