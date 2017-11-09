### 1. 创建一个版本库
```
$ mkdir git-repo
$ cd git-repo
$ git-init-db
```
tip:先创建一个文件夹，进入文件夹，然后`git-init-db`初始化；此时就会在该文件夹下创建一个`.git`文件夹。
用`ls -a`查看： `cat.git/HEAD`查看HEAD中的内容。HEAD是一个索引，总是指向你当前编辑的分支。ref保存指向对象的索引。

### 2. 将文件加入到版本库索引中git-add * 等价于git-update-index --add *
- 新建hello和example文件，并向里面添加相应内容。
```
$ echo "Hello world" > hello
$ echo "Silly example" > example
```
- 用git-add 将这两个文件加入到版本文件索引中。
`git-add hello example`
等价：`git-update-index --add hello example`

tip：`git-add`可以将一个目录下所有的文件都加入到版本控制中。
tip：`git-add`命令只是刷新git跟踪信息，并没有把内容提交到git跟踪内容之内。还需要`git-commit`来真是提交。
移除版本文件管理：
`git-update-index --force-remove foo.java`

添加所有改动到暂存区：
`git add -A`   添加所有内容
`git add .`  表示添加新文件和编辑过的文件，但是不包括删除了的文件
`git add -u`  添加编辑或删除的文件，不包括新建的文件也就是:不会处理untracted的文件
`git add -i .` 查看当前目录所有修改的文件。

### 3. 提交内容到版本库：git-commit
`git-status` 查看版本库状态；
提交到版本库：
```
git-commit -m "commit info"
git commit -am ""
git commit --amend
```

### 4. 查看当前工作目录与版本库中的不同git-diff
`git diff`   比较工作去和暂存区的差异
`git diff --cached`  比较暂存区和仓库的差异 `--stged` 相同
`git diff HEAD`  比较工作区和仓库的差异
`git diff daf8f  fdwer9  >> diff.txt`  比较两个commit之间的区别,输出到`diff.txt` 文件中

对于修改，我们可以通过`git-update-index`和`git-commit`提交；也可以通过
`git-commit -a -m "commit info"`

### 5. 分支相关操作：git branch，git checkout
```
git branch branchname   创建到分支，但是不会切换到分支
git branch -d branchname 删除分支
git branch -v 查看分支信息
```
```
git checkout -b branchname2   创建并切换到分支2
git checkout master 切换到master分支上
git checkout [版本号,通过log查看得到] [文件名name1] 将文件name1切换到指定版本号的时候
git checkout HEAD a.java 切换最近一次提交的修改上
git checkout HEAD~ a.java 切换到上上一次提交的修改，如果要切换到前面第三次的修改状态则：HEAD~3
```

### 6. 重置HEAD到指定的状态：reset
```
git reset HEAD 将暂存区的变更撤销到工作区
git mixed HEAD~ 将上一次commit的内容拷贝到暂存区
git hard HEAD~ 将上一次commit的内容拷贝的暂存区和工作区
git soft HEAD~ 只将HEAD的指针指到上一次commit的状态
```

### 7.撤销已经修改的内容：revert
```
git revert [版本号] 撤销相应版本的修改，且这次撤销作为一次修改，被记录下来
git reset --hard [版本号] 同上，但是，这次撤销不会被记录，且该版本之上的所有修改将被删除，这是一个不可逆的修改
撤销修改：
git checkout -- file //将工作区的修改丢弃
git reset HEAD 将暂存区的内容丢弃，后面可以加目录或文件名
git checkout .    丢弃所有修改
git reset --hard  hash  撤销到某个commit节点
git revert HEAD  撤销到前一个commit点
git checkout *.java  撤销所有java文件的修改
git reset --soft HASH   返回到某个commit节点。保留修改
```

### 8.将某一个修改合并到某个分支：cherry-pick
先进入到master分支
`git cherry-pick [对应修改的版本号]` 将某一个修改合并到master分支上，其他的修改不合并

### 9. git merge 和git rebase区别
注意：merge会把两个分支合成一个分支，分支开发我更喜欢：rebase！
`git merge`：将两个分支合并成一个新的分支
`git rebase` ：将a分支的修改，添加到b分支上
`git rebase origin`  把origin分支的修改添加到当前分支
如果两个分支有冲突，则修改冲突，然后：
`git rebase --continue` 继续上次的rebase
`git pull --rebase` 把你本地的每个与远程的提交不同取消掉，然后保存这些本地不同提交，然后合并远程提交，在把刚刚保存的提交应用到当前分支上。
c5,c6,就是本地提交
，生产了新的c7提交。

```
git merge branchA branchB  将branchB合并到branchA
git merge branchC  将当前分支合并到branchC 
git merge origin/master zongzhe.hu 将本地远程clone下的master分支合并到本地zongzhehu分支上
git log --graph  查看分支合并图
```

### 10、分支合并：merge、pull、fetch
`fetch` 从远程仓库拉取到本地仓库
`merge` 将本地仓库的内容合并到某个分支 `git merge --abort` 取消合并
`pull 等价于 fetch+merge`,拉取合并

拉去远程分支，到本地操作。
```
$ git fetch origin zongzhe.hu 取回origin主机的zongzhe.hu分支
$ git branch -r 查看所有远程分支
$ git checkout -b newlocalbranch origin/zongzhe.hu  在拉去下来的远程分支下新建一个分支
$ git merge origin/zongzhe.hu 再将远程分支合并到本地的当前分支（newlocalbranch）
```
这样就能在本地对远程的一个分支进行查看，操作了。
也可以直接用pull
```
$ git pull <远程主机名> <远程分支名>:<本地分支名>
$ git pull <remote> <branch>将远程分支内容合并到本地分支(本地分支没有关联到远程分支)
```
`git pull` 命令的作用是，取回远程主机某个分支的更新，再与本地的指定分支合并
Tip：如果远程分支与当前分支合并，则本地分支名可以省略。

### 11. 查看日志：log, status, reflog
`status` 查看工作去状态
`log` 查看提交日志
`reflog` 本地所有操作，不在远程仓库

### 12.删除不相关的文件 git clean：删除unversioned，untrackted文件，checkout 别人分支删除不属于他分支的内容
```
# 删除 untracked files
git clean -f

# 连 untracked 的目录也一起删掉
git clean -fd
 
# 连 gitignore 的untrack 文件/目录也一起删掉 （慎用，一般这个是用来删掉编译出来的 .o之类的文件用的）
git clean -xfd
 
# 在用上述 git clean 前，墙裂建议加上 -n 参数来先看看会删掉哪些文件，防止重要文件被误删
git clean -nxfd
git clean -nf
git clean -nfd
```

### 13. 本地推送到远程：push
`git push <远程主机名> <本地分支名>:<远程分支名>`
`git push origin dev`  将当前dev分支推送到远程dev分支
`git push origin :zongzhe`   删除远程分支origin后面有个空格
等价于：
`git push origin --delete zongzhe`
`git push -f origin dev` 将与本地对应的dev强制push到远程
注意：origin 后面有一个空格。

### 14.与远程分支建立追踪关系
`git branch --set-upstream zongzhe.hu origin/zongzhe.hu`
将本地`zongzhe.hu`分支追踪远程`zongzhe.hu`分支，`origin/zongzhe.hu`表示远程`zongzhe.hu`分支

### 15.比较本地与远程分支的差异
1. 在本地仓库位置，取回远程分支的内容，fetch不会修改本地分支的内容
`git fetch origin zongzhe.hu`
2. 比较本地分支和远程分支之间的差异
`git diff zongzhe.hu origin/zongzhe.hu`
3. 远程分支已经修改，本地未同步的变更
`$ git diff HEAD...origin/zongzhe.hu`
4.本地分支已经修改，远程未同步的变更；
`git diff origin/zongzhe.hu...HEAD`
`git merge origin/zongzhe.hu` 再将远程分支合并到本地的当前分支（newlocalbranch）
来自：`https://github.com/KylinGu/Test/blob/master/GitLearningExp.md`

### 16. 本地分支与远程分支关联
`git branch --set-upstream-to=origin/<branch> dev`
eg:`git branch --set-upstream-to=origin/dev dev`
