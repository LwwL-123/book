# Git四个区域

Git本地有三个工作区域：工作目录（Working Directory）、暂存区(Stage/Index)、资源库(Repository或Git Directory)。如果在加上远程的git仓库(Remote Directory)就可以分为四个工作区域。文件在这四个区域之间的转换关系如下：

![在这里插入图片描述](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220816120626.png)

- Workspace：工作区，就是你平时存放项目代码的地方;
- Index / Stage：暂存区，用于临时存放你的改动，事实上它只是一个文件，保存即将提交到文件列表信息,一般存放在 .git 目录下的 index 文件（.git/index）中，所以我们把暂存区有时也叫作索引（index）;
- Repository：仓库区（或本地仓库），就是安全存放数据的位置，这里面有你提交到所有版本的数据。其中HEAD指向最新放入仓库的版本;
- Remote：远程仓库，托管代码的服务器，可以简单的认为是你项目组中的一台电脑用于远程数据交换;



本地的三个区域确切的说应该是git仓库中HEAD指向的版本：

![在这里插入图片描述](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220816121515.png)

Directory：使用Git管理的一个目录，也就是一个仓库，包含我们的工作空间和Git的管理空间。

WorkSpace：需要通过Git进行版本控制的目录和文件，这些目录和文件组成了工作空间。

.git：存放Git管理信息的目录，初始化仓库的时候自动创建。

Index/Stage：暂存区，或者叫待提交更新区，在提交进入repo之前，我们可以把所有的更新放在暂存区。

Local Repo：本地仓库，一个存放在本地的版本库；HEAD会只是当前的开发分支（branch）。







### 二、git reset

![在这里插入图片描述](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220831161924.png)

在复习了git提交流程与HEAD指针与commit关系之后，我们就可以先使用git reset命令来重置文件状态了。



#### 2.1、git reset --mixed

- 重置index区

--mixed是默认参数，该命令是重置HEAD和index区。例如我们修改了工作区的一个文件并通过git add命令添加到index区，但是又想要恢复刚刚的git add操作。这种情况下会重置index区的变更，保留工作区内容



- 重置本地仓库repository区

如果我们已经执行完git commit命令，但是想要进行恢复重置的话也可以使用git reset命令，但是会稍微有一些区别

git reset HEAD^ ，这个时候就会重置index区域和repository区，但是会保留工作区内容



#### 2.2、git reset --soft

该命令的主要功能是重置HEAD，保留index和工作区。

例如我们已经执行完git commit操作，这时我们发现需要commit内容存在错误，需要恢复。我们可以执行

```
git reset --soft HEAD^
```

可以看到，在执行了reset soft命令后repository区域的已经被重置，而index区域的依然被保留



#### 2.3、git reset --hard

该命令会重置掉工作区，index区和repository区，所以在使用的使用一定要小心。
