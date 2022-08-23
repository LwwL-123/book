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





相关命令

1. 修改本地已被跟踪文件，文件进入未暂存区域。

**2.** 未暂存区域转到暂存区域

- git add files

3. 暂存区提交到本地仓库

- git commit -m

4. 直接从未暂存区提交到本地仓库

- git commit -am
- 经测试，对已跟踪的文件可以正确执行，而对于未跟踪文件（即新增文件）则会出错

5. 本地库回退到暂存区，可以修改后再次commit提交

- git reset --soft hash值
- git reset --soft origin/master
- 一般回退到暂存区的文件作排查用，不要直接修改，不然会同时出现在暂存区和未暂存区（其实即使修改了也木有太大关系）

**6.** **本地库回退到未暂存区**

- **git reset –mixed** *hash**值*
- **git reset –mixed** *origin/master*
- 一般回退到未暂存状态就是为了进一步的修改

**7.** **本地库回退到文件初始状态（即此版本的）**

- **git reset –hard** *hash**值*

- 注意这里，通常先执行一次fetch，保证本地版本是origin的最新版本，然后再回退。（最厉害的是，这么操作不会有冲突，直接让文件变成和origin保持一致）

  - **git fetch origin**
  - **git reset –hard** *origin/master*
  - 特别注意：这么操作会使你对文件的修改全部消失，还原成最初状态。

- (针对上一条情况衍生讲解)通常在推送到origin时，先要pull，然后再推送，一般是修改提交了的文件和

  pull下来的同一个文件产生冲突（所以建议修改代码前，一定先要pull）

  - **git pull**
  - **git push** *origin master*

**8.** **暂存区回退到未暂存区**

- **git reset –** *files*
- git rest
  - 撤销所有暂存区的文件

**9.** **未暂存区回退到文件初始状态**

- **git checkout –** *files*

**10.** **暂存区回退到文件初始状态**

- **git checkout head –** *files*



```go
package main

import (
	"fmt"
)

func main() {
	// 用一个map保存新冠的名字
	covid := map[byte]bool{}
	covid['L'] = true
	m, n := 6, 5
	names := [][]byte{{'A', 'B', 'C', 'D', 'E'}, {'F', 'G', 'H', 'I', 'J'}, {'K', 'L', 'M', 'N', 'O'}, {'P', 'Q', 'R', 'S', 'T'}, {'U', 'V', 'W', 'X', 'Y'}, {'Z', '0', '1', '2', '3'}}
	//定义结点移动的四个方向，分别是上下左右
	dirs := [][]int{{-1, -1}, {-1, 0}, {-1, 1}, {0, -1}, {0, 0}, {0, 1}, {1, -1}, {1, 0}, {1, 1}}
	// 队列，用于存储结点的坐标
	queue := []now{}
	// 保存结果
	res := []byte{}
	for i := 0; i < m; i++ {
		for j := 0; j < n; j++ {
			if covid[names[i][j]] {
				level := 0
				queue = append(queue, now{i, j})
				// 当前节点已经访问
				names[i][j] = '.'

				for len(queue) != 0 {
					l := len(queue)
					level++
					for i := 0; i < l; i++ {
						if level <= 2 {
							// 取出队头
							x, y := queue[i].i, queue[i].j
							// 遍历九个方向
							for k := 0; k < 9; k++ {
								mx := x + dirs[k][0]
								my := y + dirs[k][1]
								if mx >= 0 && mx < m && my >= 0 && my < n && names[mx][my] != '.' {
									queue = append(queue, now{mx, my})
									res = append(res, names[mx][my])
									names[mx][my] = '.'
								}
							}
						}
					}
					queue = queue[l:]
				}
			}
		}
	}

	for i := 0; i < m; i++ {
		for j := 0; j < n; j++ {
			fmt.Printf("%c ", names[i][j])
		}
	}

	for _, v := range res {
		fmt.Printf("%c", v)
	}
}

type now struct {
	i, j int
}

```



