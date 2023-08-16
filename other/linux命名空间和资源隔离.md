Linux内核实现namespace的主要目的就是为了实现轻量级虚拟化（容器）服务。在同一个namespace下的进程可以感知彼此的变化，而对外界的进程一无所知。这样就可以让容器中的进程产生错觉，仿佛自己置身于一个独立的系统环境中，以此达到独立和隔离的目的。

## clone()

使用clone()来创建一个独立namespace的进程是最常见做法，它的调用方式如下。

```c
int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);
```

clone()实际上是传统UNIX系统调用fork()的一种更通用的实现方式，它可以通过flags来控制使用多少功能。一共有二十多种CLONE_*的flag（标志位）参数用来控制clone进程的方方面面（如是否与父进程共享虚拟内存等等），下面外面逐一讲解clone函数传入的参数。

+ 参数child_func传入子进程运行的程序主函数。
+ 参数child_stack传入子进程使用的栈空间
+ 参数flags表示使用哪些CLONE_*标志位
+ 参数args则可用于传入用户参数

## /proc/[pid]/ns文件

从3.8版本的内核开始，用户就可以在/proc/[pid]/ns文件下看到指向不同namespace号的文件，效果如下所示，形如[4026531839]者即为namespace号。

```shell
$ ls -l /proc/$$/ns         <<-- $$ 表示应用的PID
total 0
lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 ipc -> ipc:[4026531839]
lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 mnt -> mnt:[4026531840]
lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 net -> net:[4026531956]
lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 pid -> pid:[4026531836]
lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 user->user:[4026531837]
lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 uts -> uts:[4026531838]
```

如果两个进程指向的namespace编号相同，就说明他们在同一个namespace下，否则则在不同namespace里面。/proc/[pid]/ns的另外一个作用是，一旦文件被打开，只要打开的文件描述符（fd）存在，那么就算PID所属的所有进程都已经结束，创建的namespace就会一直存在。那如何打开文件描述符呢？把/proc/[pid]/ns目录挂载起来就可以达到这个效果，命令如下。

```shell
touch ~/uts
mount --bind /proc/27514/ns/uts ~/uts\
```

## setns()

上文刚提到，在进程都结束的情况下，也可以通过挂载的形式把namespace保留下来，保留namespace的目的自然是为以后有进程加入做准备。通过setns()系统调用，你的进程从原先的namespace加入我们准备好的新namespace，使用方法如下。

```c
int setns(int fd, int nstype);
```

参数fd表示我们要加入的namespace的文件描述符。上文已经提到，它是一个指向/proc/[pid]/ns目录的文件描述符，可以通过直接打开该目录下的链接或者打开一个挂载了该目录下链接的文件得到。

参数nstype让调用者可以去检查fd指向的namespace类型是否符合我们实际的要求。如果填0表示不检查。
为了把我们创建的namespace利用起来，我们需要引入execve()系列函数，这个函数可以执行用户命令，最常用的就是调用/bin/bash并接受参数，运行起一个shell，用法如下。

```c
fd = open(argv[1], O_RDONLY);   /* 获取namespace文件描述符 */
setns(fd, 0);                   /* 加入新的namespace */
execvp(argv[2], &argv[2]);      /* 执行程序 */
```
假设编译后的程序名称为setns。

```shell
./setns ~/uts /bin/bash   # ~/uts 是绑定的/proc/27514/ns/uts
```
至此，就可以在新的命名空间中执行shell命令了。

## unshare()

最后要提的系统调用是unshare()，它跟clone()很像，不同的是，unshare()运行在原先的进程上，不需要启动一个新进程，使用方法如下。

```c
int unshare(int flags);
```

调用unshare()的主要作用就是不启动一个新进程就可以起到隔离的效果，相当于跳出原先的namespace进行操作。这样，你就可以在原进程进行一些需要隔离的操作。Linux中自带的unshare命令，就是通过unshare()系统调用实现的。
