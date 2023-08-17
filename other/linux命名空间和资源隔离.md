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

## UTS（UNIX Time-sharing System）namespace

UTS namespace提供了主机名和域名的隔离，这样每个容器就可以拥有了独立的主机名和域名，在网络上可以被视作一个独立的节点而非宿主机上的一个进程。

```c
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
 
#define STACK_SIZE (1024 * 1024)
 
static char child_stack[STACK_SIZE];
char* const child_args[] = {
  "/bin/bash",
  NULL
};
 
int child_main(void* arg) {
  printf("在子进程中!\n");
  sethostname("Changed_Namespace", 12);
  execv(child_args[0], child_args);
  return 1;
}

int main() {
  printf("程序开始: \n");
  int child_pid = clone(child_main, child_stack+STACK_SIZE,
    CLONE_NEWUTS | SIGCHLD, NULL);
  printf("child_pid: %d\n", child_pid);
  waitpid(child_pid, NULL, 0);
  printf("已退出\n");
  return 0;
}
```

```bash
root@local:~#gcc test.c -o test && sudo ./test
程序开始:
child_pid: 3015059
在子进程中!
root@Changed_Name:/home/ubuntu# exit
exit
已退出
```

## IPC（Interprocess Communication）namespace

容器中进程间通信采用的方法包括常见的信号量、消息队列和共享内存。然而与虚拟机不同的是，容器内部进程间通信对宿主机来说，实际上是具有相同PID namespace中的进程间通信，因此需要一个唯一的标识符来进行区别。申请IPC资源就申请了这样一个全局唯一的32位ID，所以IPC namespace中实际上包含了系统IPC标识符以及实现POSIX消息队列的文件系统。在同一个IPC namespace下的进程彼此可见，而与其他的IPC namespace下的进程则互相不可见。

```c
//[...]
int child_pid = clone(child_main, child_stack+STACK_SIZE,
           CLONE_NEWIPC | CLONE_NEWUTS | SIGCHLD, NULL);
//[...]
```

```bash
# 先在shell中使用ipcmk -Q命令创建一个message queue
ubuntu@VM-16-4-ubuntu:~$ ipcmk -Q
Message queue id: 0
# 进入隔离环境
ubuntu@VM-16-4-ubuntu:~$ gcc test.c -o test && sudo ./test 
程序开始: 
child_pid: 3016881
在子进程中!
# 查看隔离环境中的message queue
root@Changed_Name:/home/ubuntu# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages 
```

上面的结果显示中可以发现，已经找不到原先声明的message queue，实现了IPC的隔离。

目前使用IPC namespace机制的系统不多，其中比较有名的有PostgreSQL。Docker本身通过socket或tcp进行通信。

## PID namespace

PID namespace隔离非常实用，它对进程PID重新标号，即两个不同namespace下的进程可以有同一个PID。每个PID namespace都有自己的计数程序。内核为所有的PID namespace维护了一个树状结构，最顶层的是系统初始时创建的，我们称之为root namespace。他创建的新PID namespace就称之为child namespace（树的子节点），而原先的PID namespace就是新创建的PID namespace的parent namespace（树的父节点）。通过这种方式，不同的PID namespaces会形成一个等级体系。所属的父节点可以看到子节点中的进程，并可以通过信号等方式对子节点中的进程产生影响。反过来，子节点不能看到父节点PID namespace中的任何内容。由此产生如下结论。

+ 每个PID namespace中的第一个进程“PID 1“，都会像传统Linux中的init进程一样拥有特权，起特殊作用。
+ 一个namespace中的进程，不可能通过kill或ptrace影响父节点或者兄弟节点中的进程，因为其他节点的PID在这个namespace中没有任何意义。
+ 如果你在新的PID namespace中重新挂载/proc文件系统，会发现其下只显示同属一个PID namespace中的其他进程。
+ 在root namespace中可以看到所有的进程，并且递归包含所有子节点中的进程。

```c
//[...]
int child_pid = clone(child_main, child_stack+STACK_SIZE,
           CLONE_NEWPID | CLONE_NEWIPC | CLONE_NEWUTS 
           | SIGCHLD, NULL);
//[...]
```

```bash
ubuntu@VM-16-4-ubuntu:~$ echo $$
3016529
ubuntu@VM-16-4-ubuntu:~$ gcc test.c -o test && sudo ./test 
程序开始: 
child_pid: 3018816
在子进程中!
root@Changed_Name:/home/ubuntu# echo $$
1
root@Changed_Name:/home/ubuntu# exit
exit
已退出
ubuntu@VM-16-4-ubuntu:~$ echo $$
3016529
```

在子进程的shell中执行了ps aux/top之类的命令，发现还是可以看到所有父进程的PID，那是因为我们还没有对文件系统进行隔离，ps/top之类的命令调用的是真实系统下的/proc文件内容，看到的自然是所有的进程。

此外，与其他的namespace不同的是，为了实现一个稳定安全的容器，PID namespace还需要进行一些额外的工作才能确保其中的进程运行顺利。

### PID namespace中的init进程

当我们新建一个PID namespace时，默认启动的进程PID为1。我们知道，在传统的UNIX系统中，PID为1的进程是init，地位非常特殊。他作为所有进程的父进程，维护一张进程表，不断检查进程的状态，一旦有某个子进程因为程序错误成为了“孤儿”进程，init就会负责回收资源并结束这个子进程。所以在你要实现的容器中，启动的第一个进程也需要实现类似init的功能，维护所有后续启动进程的运行状态。

PID namespace维护这样一个树状结构，非常有利于系统的资源监控与回收。Docker启动时，第一个进程也是这样，实现了进程监控和资源回收，它就是dockerinit。

### 信号与init进程

PID namespace中的init进程如此特殊，自然内核也为他赋予了特权——信号屏蔽。如果init中没有写处理某个信号的代码逻辑，那么与init在同一个PID namespace下的进程（即使有超级权限）发送给它的该信号都会被屏蔽。这个功能的主要作用是防止init进程被误杀。

那么其父节点PID namespace中的进程发送同样的信号会被忽略吗？父节点中的进程发送的信号，如果不是SIGKILL（销毁进程）或SIGSTOP（暂停进程）也会被忽略。但如果发送SIGKILL或SIGSTOP，子节点的init会强制执行（无法通过代码捕捉进行特殊处理），也就是说父节点中的进程有权终止子节点中的进程。

一旦init进程被销毁，同一PID namespace中的其他进程也会随之接收到SIGKILL信号而被销毁。理论上，该PID namespace自然也就不复存在了。但是如果/proc/[pid]/ns/pid处于被挂载或者打开状态，namespace就会被保留下来。然而，保留下来的namespace无法通过setns()或者fork()创建进程，所以实际上并没有什么作用。

我们常说，Docker一旦启动就有进程在运行，不存在不包含任何进程的Docker，也就是这个道理。

### 挂载proc文件系统

前文中已经提到，如果你在新的PID namespace中使用ps命令查看，看到的还是所有的进程，因为与PID直接相关的/proc文件系统（procfs）没有挂载到与原/proc不同的位置。所以如果你只想看到PID namespace本身应该看到的进程，需要重新挂载/proc，命令如下。

```bash
root@NewNamespace:~# mount -t proc proc /proc
root@NewNamespace:~# ps a
  PID TTY      STAT   TIME COMMAND
    1 pts/1    S      0:00 /bin/bash
   12 pts/1    R+     0:00 ps a
```

可以看到实际的PID namespace就只有两个进程在运行。

注意：因为此时我们没有进行mount namespace的隔离，所以这一步操作实际上已经影响了 root namespace的文件系统，当你退出新建的PID namespace以后再执行ps a就会发现出错，再次执行mount -t proc proc /proc可以修复错误。

### unshare()和setns()

在开篇我们就讲到了unshare()和setns()这两个API，而这两个API在PID namespace中使用时，也有一些特别之处需要注意。

unshare()允许用户在原有进程中建立namespace进行隔离。但是创建了PID namespace后，原先unshare()调用者进程并不进入新的PID namespace，接下来创建的子进程才会进入新的namespace，这个子进程也就随之成为新namespace中的init进程。

类似的，调用setns()创建新PID namespace时，调用者进程也不进入新的PID namespace，而是随后创建的子进程进入。

为什么创建其他namespace时unshare()和setns()会直接进入新的namespace而唯独PID namespace不是如此呢？因为调用getpid()函数得到的PID是根据调用者所在的PID namespace而决定返回哪个PID，进入新的PID namespace会导致PID产生变化。而对用户态的程序和库函数来说，他们都认为进程的PID是一个常量，PID的变化会引起这些进程奔溃。

换句话说，一旦程序进程创建以后，那么它的PID namespace的关系就确定下来了，进程不会变更他们对应的PID namespace。

## Mount namespaces

Mount namespace通过隔离文件系统挂载点对隔离文件系统提供支持，它是历史上第一个Linux namespace，所以它的标识位比较特殊，就是CLONE_NEWNS。隔离后，不同mount namespace中的文件结构发生变化也互不影响。你可以通过/proc/[pid]/mounts查看到所有挂载在当前namespace中的文件系统，还可以通过/proc/[pid]/mountstats看到mount namespace中文件设备的统计信息，包括挂载文件的名字、文件系统类型、挂载位置等等。

进程在创建mount namespace时，会把当前的文件结构复制给新的namespace。新namespace中的所有mount操作都只影响自身的文件系统，而对外界不会产生任何影响。这样做非常严格地实现了隔离，但是某些情况可能并不适用。比如父节点namespace中的进程挂载了一张CD-ROM，这时子节点namespace拷贝的目录结构就无法自动挂载上这张CD-ROM，因为这种操作会影响到父节点的文件系统。

2006 年引入的挂载传播（mount propagation）解决了这个问题，挂载传播定义了挂载对象（mount object）之间的关系，系统用这些关系决定任何挂载对象中的挂载事件如何传播到其他挂载对象。所谓传播事件，是指由一个挂载对象的状态变化导致的其它挂载对象的挂载与解除挂载动作的事件。

- 共享关系（share relationship）。如果两个挂载对象具有共享关系，那么一个挂载对象中的挂载事件会传播到另一个挂载对象，反之亦然。
- 从属关系（slave relationship）。如果两个挂载对象形成从属关系，那么一个挂载对象中的挂载事件会传播到另一个挂载对象，但是反过来不行；在这种关系中，从属对象是事件的接收者。

一个挂载状态可能为如下的其中一种：

- 共享挂载（shared）
- 从属挂载（slave）
- 共享/从属挂载（shared and slave）
- 私有挂载（private）
- 不可绑定挂载（unbindable）

传播事件的挂载对象称为共享挂载（shared mount）；接收传播事件的挂载对象称为从属挂载（slave mount）。既不传播也不接收传播事件的挂载对象称为私有挂载（private mount）。另一种特殊的挂载对象称为不可绑定的挂载（unbindable mount），它们与私有挂载相似，但是不允许执行绑定挂载，即创建mount namespace时这块文件对象不可被复制。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308172027102.png)

共享挂载的应用场景非常明显，就是为了文件数据的共享所必须存在的一种挂载方式；从属挂载更大的意义在于某些“只读”场景；私有挂载其实就是纯粹的隔离，作为一个独立的个体而存在；不可绑定挂载则有助于防止没有必要的文件拷贝，如某个用户数据目录，当根目录被递归式的复制时，用户目录无论从隐私还是实际用途考虑都需要有一个不可被复制的选项。

默认情况下，所有挂载都是私有的。设置为共享挂载的命令如下。

```bash
mount --make-shared <mount-object>
```

从共享挂载克隆的挂载对象也是共享的挂载；它们相互传播挂载事件。

设置为从属挂载的命令如下。

```bash
mount --make-slave <shared-mount-object>
```

从从属挂载克隆的挂载对象也是从属的挂载，它也从属于原来的从属挂载的主挂载对象。

将一个从属挂载对象设置为共享/从属挂载，可以执行如下命令或者将其移动到一个共享挂载对象下。

```bash
mount --make-shared <slave-mount-object>
```

如果你想把修改过的挂载对象重新标记为私有的，可以执行如下命令。

```bash
mount --make-private <mount-object>
```

通过执行以下命令，可以将挂载对象标记为不可绑定的。

```bash
mount --make-unbindable <mount-object>
```

这些设置都可以递归式地应用到所有子目录中。

在代码中实现mount namespace隔离与其他namespace类似，加上CLONE_NEWNS标识位即可。

```c
//[...]
int child_pid = clone(child_main, child_stack+STACK_SIZE,
           CLONE_NEWNS | CLONE_NEWPID | CLONE_NEWIPC 
           | CLONE_NEWUTS | SIGCHLD, NULL);
//[...]
```

执行的效果就如同PID namespace一节中“挂载proc文件系统”的执行结果，区别就是退出mount namespace以后，root namespace的文件系统不会被破坏。

##  Network namespace

通过上节，我们了解了PID namespace，当我们兴致勃勃地在新建的namespace中启动一个“Apache”进程时，却出现了“80端口已被占用”的错误，原来主机上已经运行了一个“Apache”进程。怎么办？这就需要用到network namespace技术进行网络隔离啦。

Network namespace主要提供了关于网络资源的隔离，包括网络设备、IPv4和IPv6协议栈、IP路由表、防火墙、/proc/net目录、/sys/class/net目录、端口（socket）等等。一个物理的网络设备最多存在在一个network namespace中，你可以通过创建veth pair（虚拟网络设备对：有两端，类似管道，如果数据从一端传入另一端也能接收到，反之亦然）在不同的network namespace间创建通道，以此达到通信的目的。

一般情况下，物理网络设备都分配在最初的root namespace（表示系统默认的namespace，在PID namespace中已经提及）中。但是如果你有多块物理网卡，也可以把其中一块或多块分配给新创建的network namespace。需要注意的是，当新创建的network namespace被释放时（所有内部的进程都终止并且namespace文件没有被挂载或打开），在这个namespace中的物理网卡会返回到root namespace而非创建该进程的父进程所在的network namespace。

当我们说到network namespace时，其实我们指的未必是真正的网络隔离，而是把网络独立出来，给外部用户一种透明的感觉，仿佛跟另外一个网络实体在进行通信。为了达到这个目的，容器的经典做法就是创建一个veth pair，一端放置在新的namespace中，通常命名为eth0，一端放在原先的namespace中连接物理网络设备，再通过网桥把别的设备连接进来或者进行路由转发，以此网络实现通信的目的。

在建立起veth pair之前，新旧namespace该如何通信呢？答案是pipe（管道）。我们以Docker Daemon在启动容器dockerinit的过程为例。Docker Daemon在宿主机上负责创建这个veth pair，通过netlink调用，把一端绑定到docker0网桥上，一端连进新建的network namespace进程中。建立的过程中，Docker Daemon和dockerinit就通过pipe进行通信，当Docker Daemon完成veth-pair的创建之前，dockerinit在管道的另一端循环等待，直到管道另一端传来Docker Daemon关于veth设备的信息，并关闭管道。dockerinit才结束等待的过程，并把它的“eth0”启动起来。整个效果类似下图所示。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308172039559.png)

跟其他namespace类似，对network namespace的使用其实就是在创建的时候添加CLONE_NEWNET标识位。也可以通过命令行工具ip创建network namespace。在代码中建立和测试network namespace较为复杂，所以下文主要通过ip命令直观的感受整个network namespace网络建立和配置的过程。

首先我们可以创建一个命名为test_ns的network namespace。

```bash
# ip netns add test_ns
```

当ip命令工具创建一个network namespace时，会默认创建一个回环设备（loopback interface：lo），并在/var/run/netns目录下绑定一个挂载点，这就保证了就算network namespace中没有进程在运行也不会被释放，也给系统管理员对新创建的network namespace进行配置提供了充足的时间。

通过ip netns exec命令可以在新创建的network namespace下运行网络管理命令。

```bash
# ip netns exec test_ns ip link list
3: lo: <LOOPBACK> mtu 16436 qdisc noop state DOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

上面的命令为我们展示了新建的namespace下可见的网络链接，可以看到状态是DOWN,需要再通过命令去启动。可以看到，此时执行ping命令是无效的。

```bash
# ip netns exec test_ns ping 127.0.0.1
connect: Network is unreachable
```

启动命令如下，可以看到启动后再测试就可以ping通。

```bash
# ip netns exec test_ns ip link set dev lo up
# ip netns exec test_ns ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_req=1 ttl=64 time=0.050 ms
...
```

这样只是启动了本地的回环，要实现与外部namespace进行通信还需要再建一个网络设备对，命令如下。

```bash
# ip link add veth0 type veth peer name veth1
# ip link set veth1 netns test_ns
# ip netns exec test_ns ifconfig veth1 10.1.1.1/24 up
# ifconfig veth0 10.1.1.2/24 up
```

第一条命令创建了一个网络设备对，所有发送到veth0的包veth1也能接收到，反之亦然。
第二条命令则是把veth1这一端分配到test_ns这个network namespace。
第三、第四条命令分别给test_ns内部和外部的网络设备配置IP，veth1的IP为10.1.1.1，veth0的IP为10.1.1.2。
此时两边就可以互相连通了，效果如下。

```bash
# ping 10.1.1.1
PING 10.1.1.1 (10.1.1.1) 56(84) bytes of data.
64 bytes from 10.1.1.1: icmp_req=1 ttl=64 time=0.095 ms
...
# ip netns exec test_ns ping 10.1.1.2
PING 10.1.1.2 (10.1.1.2) 56(84) bytes of data.
64 bytes from 10.1.1.2: icmp_req=1 ttl=64 time=0.049 ms
...
```

可以通过下面的命令查看，新的test_ns有着自己独立的路由和iptables。

```bash
ip netns exec test_ns route
ip netns exec test_ns iptables -L
```

路由表中只有一条通向10.1.1.2的规则，此时如果要连接外网肯定是不可能的，你可以通过建立网桥或者NAT映射来决定这个问题。如果对此非常感兴趣，可以阅读Docker网络相关文章进行更深入的讲解。

做完这些实验，你还可以通过下面的命令删除这个network namespace。

```bash
# ip netns delete netns1
```

这条命令会移除之前的挂载，但是如果namespace本身还有进程运行，namespace还会存在下去，直到进程运行结束。

通过network namespace我们可以了解到，实际上内核创建了network namespace以后，真的是得到了一个被隔离的网络。但是我们实际上需要的不是这种完全的隔离，而是一个对用户来说透明独立的网络实体，我们需要与这个实体通信。所以Docker的网络在起步阶段给人一种非常难用的感觉，因为一切都要自己去实现、去配置。你需要一个网桥或者NAT连接广域网，你需要配置路由规则与宿主机中其他容器进行必要的隔离，你甚至还需要配置防火墙以保证安全等等。