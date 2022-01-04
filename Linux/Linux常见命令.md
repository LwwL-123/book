# Linux常见命令

## nohup命令

nohup命令用于不挂断地运行命令（关闭当前session不会中断改程序，只能通过kill等命令删除）。
使用nohup命令提交作业，如果使用nohup命令提交作业，那么在缺省情况下该作业的所有输出都被重定向到一个名为nohup.out的文件中，除非另外指定了输出文件。



1. 示例

```bash
nphup ./ttchain daemon >provider.log 2>&1 &
```



2. 2>&1

```bash
bash中：
0 代表STDIN_FILENO 标准输入（一般是键盘），
1 代表STDOUT_FILENO 标准输出（一般是显示屏，准确的说是用户终端控制台），
2 三代表STDERR_FILENO (标准错误（出错信息输出）。
```

```
> 直接把内容生成到指定文件，会覆盖原来文件中的内容[ls > test.txt],
>> 尾部追加，不会覆盖原有内容 [ls >> test.txt],
< 将指定文件的内容作为前面命令的参数[cat < text.sh]
```

2>&1就是用来将标准错误2重定向到标准输出1中的。此处1前面的&就是为了让bash将1解释成标准输出而不是文件1。至于最后一个&，则是让bash在后台执行。





## lsof

lsof的作用是列出当前系统打开文件(list open files)，不过通过`-i`参数也能查看端口的连接情况，-i后跟冒号端口可以查看指定端口信息，直接-i是系统当前所有打开的端口

`lsof -i:22 #查看22端口连接情况，默认为sshd端口` 



可以看到当前通过端口22连接到机器的一共有4个(主机名和ip已打码)，通过该命令就能清楚知道当前端口状态

 

(1) netstat -an|grep 8080
(2) lsof -i:8080

## netstat

netstat用来查看系统当前系统网络状态信息，包括端口，连接情况等，常用方式如下：

`netstat -atunlp`,各参数含义如下:

- -t : 指明显示TCP端口
- -u : 指明显示UDP端口
- -l : 仅显示监听套接字(LISTEN状态的套接字)
- -p : 显示进程标识符和程序名称，每一个套接字/端口都属于一个程序
- -n : 不进行DNS解析
- -a 显示所有连接的端口

执行后得表格一目了然，就不做截图了，当然，在众多表目中找一个特定得，肯定不那么顺手，一般该指令会遇grep配合使用，比如查找端口22,就用`netstat -tunlp | grep 22` 或者干脆`netstat -an | grep 22`就可以了，查看其它端口类似，当然也可以通过端口状态查找即`netstat -anp | grep TIME_WAIT`，即只会显示含有`TIME_WAIT`字符串的条目



区别：
**1.netstat无权限控制，lsof有权限控制，只能看到本用户**
**2.losf能看到pid和用户，可以找到哪个进程占用了这个端口**



### Mac 上无法使用netstat的-p参数

改为使用`lsof`命令，例：

```bash
lsof -i -P | grep -i "listen"
```

## SCP用法

scp命令的使用频率越来越高，大概的举例说明下这个命令

1. 获取远程服务器上的文件

```
scp -P 22 root@remoteHost:/root/test.tar.gz /home/test.tar.gz
```

端口大写P 为参数，22 表示指定连接SSH的端口，如果没有更改默认的SSH端口（即：22）可以不用添加该参数。

 root@remoteHost 表示使用root用户登录远程服务器remoteHost，

:/root/test.tar.gz 表示远程服务器上的文件，

最后面的/home/test.tar.gz表示保存到本地上的路径和文件名。

　**注意：在需要指定端口时要使用-P(大写的P)，而且要紧跟在scp之后：scp -P 12349 upload_file username@server（正确）**

2. 获取远程服务器上的目录

```
scp -r root@remoteHost:/root/testdir  /home/testdir/
```

-r 参数表示递归复制（即复制该目录下面的文件和子目录）；

/root/testdir/ 表示远程服务器上的目录，最后面的/home/testdir/表示保存在本地上的路径。



3. 将本地文件上传到服务器上

```
scp /home/upload.tar.gz root@remoteHost:/root/upload.tar.gz
```

由上例可知，scp命令大致用法为 scp [源路径] [目标路径]，
当下载文件时 源路径为服务器的路径，当上传文件时源路径为本地路径；
服务器的路径一般为 [用户名]@[主机地址/IP/域名]:[服务器上的路径]
本地路径即本地操作系统的路径，windows有win的写法，linux有linux写法，视情况而定

其余常用参数有
-4 强制使用ipv4
-6 强制使用ipv6
-v 和大多数 linux 命令中的 -v 意思一样 , 用来显示进度 . 可以用来查看连接 , 认证 , 或是配置错误 .
-C 使能压缩