# Git Command 
常用的git命令集合

## Reference
* 常用 Git 命令清单[https://www.ruanyifeng.com/blog/2015/12/git-cheat-sheet.html](https://www.ruanyifeng.com/blog/2015/12/git-cheat-sheet.html)

## Config
### 配置
* `git config -l` 查看git配置


## Commands

### Remote
* `git log --oneline` 查看提交记录（一行显示）

### Branch
* `git branch`  查看本地分支
* `git branch --remote`  查看远端分支
* `git checkout -b <branch_name>`  新建并切换到新分支


### 提交
* `git diff` 查看被修改的文件
* `git add <file_path>` 把文件提交至预备提交区域
* `git commmit -m "commit"`提交文件，增加描述
* `git commit -am "commit_msg"` 自动将被修改、删除的文件（不包括未加入索引的文件）加入暂存区，并提交
* `git reset HEAD <file_path>`取消文件的暂存状态（预备提交区）

### 回退/恢复
* `git reset --soft HEAD^` 取消上一次提交，保留提交后的修改

### Rebase
> 变基操作。orgin和branch1两个分支各自有多次提交，如果想把branch1上的内容提交到origin上，除了merge操作，我们还可以使用`git rebase`。它会将branch1上的commit操作暂时取消，暂存至`.git/rebase`目录下，然后将origin分支的内容更新到branch1，最后把保存的commit应用到branch1上。

* `git rebase <branch_name>` 变基到<branch_name>，过程中可能会产生冲突。

