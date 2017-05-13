# 基本使用
## 版本库创建和基本使用
- 初始化git仓库: `git init`
- 添加文件到版本控制:  
`git add readme.txt`  
`git add file1.txt file2.txt`
- 提交: `git commit -m "这是一次提交"`
- 查看状态: `git status`
- 查看diff: `git diff readme.txt`
- 查看log:  
`git log`  
`git log --pretty=oneline`  
`git log --graph`: 可以看到分支图

## 版本控制
- 版本回退:  
`git reset --hard HEAD^` 回退到上个版本，同HEAD~1  
`git reset --hard HEAD~1`  
`git reset --hard ea34578`：直接写commitId也ok
`git reflog`: 查看命令历史
- 工作区和暂存区：
    - 工作区：可以理解为当前仓库中已经修改，但是没有add到暂存区的文件
    - 暂存区：需要一次性提交的文件修改（关键词：一次性提交；修改）
> 注：git追踪的是修改，所以git commit提交的是add之前的修改，add之后的修改不会提交。 
- 撤销工作区中的文件修改：  
`git chekout -- readme.txt` 
这个命令会把readme.txt在工作区中的修改撤销，即：恢复到上一次的add，或者没有add则恢复到commit，或者没有commit则恢复到和版本库一样的状态。
- 从版本库删除：`git rm test.txt`

## 远程仓库
- 为远程仓库关联一个名称（一般叫做origin，用于后续的pull/push操作)：  
`git remote add origin <url>`
- 将本地仓库推送到远程：  
`git push -u origin master`: 首次推送时，-u（set-upstream)参数构建本地和远程分支的关联  
`git push origin master`: 本地分支推送到远程
- 从远程克隆分支：`git clone`

## 分支管理
- 查看分支：`git branch`  
- 创建分支：`git branch <name>`  
- 切换分支：`git checkout <name>`  
- 创建+切换分支：`git checkout -b <name>`  
- 合并某分支到当前分支：`git merge <name>`  
- 删除分支：`git branch -d <name>`
- 强行删除未合并到主分支的分支：`git branch -d <name>`
- 禁用FastForward的merge:  
`git merge --no-ff -m "merge without fast forward" dev`: 如果可能的话，git会使用FastForward来合并分支，即：只通过改变指针指向，来实现合并。但是我们可以禁用，使用--no-ff参数，这样每次合并一定会产生一个commit，所以我们可通过-m来添加一个commit的消息。
- git stash保存现场：  
`git stash`: 将当前的暂存区的内容保存下来  
`git stash list`：查看当前stash过的内容（可以stash多次）  
`git stash apply`：将stash过的内容恢复  
`git stash drop`：删掉上一次的stash记录  
`git stash pop`：恢复并删除，等同于git stash apply+pop
`git stash stash@{0}`: 恢复到某一个stash状态  

## 标签管理
- 查看tag: `git tag`  
- 在当前分支的最新commit上打一个标签： `git tag <tagname>`
- 在当前分支的指定commitId上打一个标签： `git tag <tagname> <commit id>`
- 显示某个tag详细信息： `git show <tagname>`  
- 创建一个带有说明文字的标签： `git tag -a v0.1 -m "I add a tag named v0.1"`
- 本地删除一个标签：`git tag -d <tagname>`
- 本地标签push到远程： `git push origin <tagname>`
- 本地所有标签push到远程： `git push origin --tags`
- 删除远程标签： 先本地删除，再使用`git push origin :refs/tags/v0.9`

## 其他
### Git的安装
Linux下安装：
`sudo apt-get install git-core`
Mac安装：
需要安装Homebrew
Windows安装：
Cygwin或者是直接下载exe

### git的配置
- 配置全局的用户名和密码：  
```
$ git config --global user.name "Your Name"  
$ git config --global user.email "email@example.com"
```
注：其中的--global表示所有的git仓库都使用这个配置，也可以对某个仓库指定不同的用户名和Email地址
- 配置别名：  
`git config --global alias.last 'log -1'`  
配置的信息都在`.git/config`文件中