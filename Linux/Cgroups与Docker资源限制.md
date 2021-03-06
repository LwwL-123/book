# Cgroups限制和Namespace隔离

## 1. 容器和虚拟机的区别

在操作系统中，一个进程的虚拟内存被分为内核空间（kernel space）和用户空间（user Space）

Kernel space 可以执行任意命令，调用系统的一切资源；User space 只能执行简单的运算，不能直接调用系统资源，必须通过系统接口（又称 system call），才能向内核发出指令。

而容器事实上就是一个用户程序，它同样通过调用系统接口实现功能。对于 docker 而言，如果宿主机是 linux，它就直接使用宿主机内核，如果宿主机是 macos 或者 windows，docker服务会提供一个公用的 linux内核供各容器调用。



![image-20211118141922226](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325171143.png)





那么，容器和虚拟机的区别也就明显了，它们虚拟化的层次并不一样。

<img src="https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325171147.png" alt="image-20211118142312761" style="zoom:150%;" />

虚拟机完整虚拟了内核和用户空间，而 docker 仅仅虚拟了用户空间，那么 docker 必然更轻量、更快。

再回头看第一张图，也就清晰了。虚拟机建立在虚拟硬件层之上，每个虚拟机都有独立的内核和用户程序以及依赖库；而 docker 容器建立在宿主机内核和 docker 服务之上，使用共同的内核，每个容器仅仅是用户程序、依赖库不同；再更进一步，普通的程序都使用共同的依赖库，只有程序文件不同。





## 2. Namespace做隔离

Linux 内核从版本 2.4.19 开始陆续引入了 namespace 的概念。其目的是将某个特定的全局系统资源通过抽象方法使得namespace 中的进程看起来拥有它们自己的隔离的全局系统资源实例。Linux 内核中实现了六种 namespace，按照引入的先后顺序，列表如下：

纵观全局，从普通的应用程序，到docker 容器，再到虚拟机技术，无非是对磁盘空间、系统资源和隔离性、安全性的权衡取舍而已。至于 docker 的自动化，对比虚拟机似乎并不公平，于 Vagrant 这样的工具相比倒是更加合适。

![img](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325171151.jpg)

例如，容器进程启动时，只要启用了Mount Namespace，并将自己打包的文件系统挂载好，就可以实现每个容器仅看到自己的文件，实现文件资源的隔离。总之，Docker 守护进程创建容器实例时都启用了相应的namespace，使得容器中的进程都处于一种隔离的运行环境之中。

### **那么如何启用相应Namespace呢？**

通过系统调用clone()来创建一个具有独立Namespace的进程是最常见的做法，它可以通过flags参数传入相应标志位来控制进程的各种状态，如以下示意代码：

```
pid = clone(fun,stack,flags,clone_arg);
(flags:CLONE_NEWPID  | CLONE_NEWNS |
    CLONE_NEWUSER | CLONE_NEWNUT |
    CLONE_NEWIPC  | CLONE_NEWUTS |
    ...)
```

**docker run中namespace相关参数**

- --ipc string IPC namespace to use
- --pid string PID namespace to use
- --userns string User namespace to use
- --uts string UTS namespace to use

你可以在容器启动的时候，指定这些参数，从而强制容器运行在特定namespace之中。例如，你可以指定 --pid host，从而让容器进程使用宿主机进程空间，此时容器可以看到host上所有的进程（想象这样一个场景，你把常用的性能诊断工具都打包到一个镜像中，然后必要的时候在服务器上使用此镜像进行问题分析，此时加上该参数会很方便）。







## 3. Cgroups (Linux control groups)

通过Namespace，容器实现了资源的隔离，从而每个容器看起来都像是拥有自己独立的运行环境。注意，只是看起来。因为容器使用cpu、内存等并不受限制，假如某个容器占用这些资源过高，就可能会造成其它容器运行迟缓甚至异常，这就需要Cgroups了。



Linux CGroup全称Linux Control Group， 是Linux内核的一个功能，用来限制，控制与分离一个进程组群的资源（如CPU、内存、磁盘输入输出等）。

- 其典型的子系统如下：
  - cpu 子系统，主要限制进程的 cpu 使用率。
  - cpuacct 子系统，可以统计 cgroups 中的进程的 cpu 使用报告。
  - cpuset 子系统，可以为 cgroups 中的进程分配单独的 cpu 节点或者内存节点。
  - memory 子系统，可以限制进程的 memory 使用量。
  - blkio 子系统，可以限制进程的块设备 io。
  - devices 子系统，可以控制进程能够访问某些设备。
  - net_cls 子系统，可以标记 cgroups 中进程的网络数据包，然后可以使用 tc 模块（traffic control）对数据包进行控制。
  - freezer 子系统，可以挂起或者恢复 cgroups 中的进程。
  - ns 子系统，可以使不同 cgroups 下面的进程使用不同的 namespace。



而Cgroups的实现也很有意思，它并不是一组系统调用，linux将其实现为了文件系统，这很符合Unix一切皆文件的哲学，因此我们可以直接查看。

例如，我在ubuntu18.04系统中，直接执行mount -t cgroup即可看到，系统已经自动在sys/fs/cgroup目录下挂载好了相应文件，每个文夹件代表了上面所讲的某种资源类型。



**如何使用Cgroups呢**

很简单，我们可以直接在相应资源控制组目录下创建文件夹，系统会自动创建需要的文件，例如，在上述cpu目录下创建hello目录，然后看到相应文件已自动创建。



**docker对Cgroups的使用**

默认情况下，docker 启动一个容器后，就会在 /sys/fs/cgroup 目录下的各个资源目录下生成以容器 ID 为名字的目录，在容器被 stopped 后，该目录被删除。那么，对容器资源进行控制的方式，就同上边的例子一样，显而易见了



docker提供了–cpu-period、–cpu-quota两个参数控制容器可以分配到的CPU时钟周期。–cpu-period是用来指定容器对CPU的使用要在多长时间内做一次重新分配，而–cpu-quota是用来指定在这个周期内，最多可以有多少时间用来跑这个容器。跟–cpu-shares不同的是这种配置是指定一个绝对值，而且没有弹性在里面，容器对CPU资源的使用绝对不会超过配置的值。

cpu-period和cpu-quota的单位为微秒（μs）。cpu-period的最小值为1000微秒，最大值为1秒（10^6 μs），默认值为0.1秒（100000 μs）。cpu-quota的值默认为-1，表示不做控制。

举个例子，如果容器进程需要每1秒使用单个CPU的0.2秒时间，可以将cpu-period设置为1000000（即1秒），cpu-quota设置为200000（0.2秒）。当然，在多核情况下，如果允许容器进程需要完全占用两个CPU，则可以将cpu-period设置为100000（即0.1秒），cpu-quota设置为200000（0.2秒）。
