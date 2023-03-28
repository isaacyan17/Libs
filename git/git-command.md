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
* `git push -f orgin master` 强制覆盖远程分支


### Branch
* `git branch`  查看本地分支
* `git branch --remote`  查看远端分支
* `git checkout -b <branch_name>`  新建并切换到新分支


### 提交
* `git diff` 查看被修改的文件
* `git add <file_path>` 把文件提交至预备提交区域
* `git add --all` 当前项目下的所有更改
* `git add .` 当前目录下的所有更改
* `git commit -m "commit"`提交文件，增加描述
* `git commit -am "commit_msg"` 自动将被修改、删除的文件（不包括未加入索引的文件）加入暂存区，并提交
* `git reset HEAD <file_path>`取消文件的暂存状态（预备提交区）

### 回退/恢复
* `git diff` 列出所有的修改
* `git diff --cached` 显示暂存区和本地仓库的差异
* `git diff <file1> <file2>` 列出某些文件的修改
* `git clean -f`  untracked状态，撤销新增的文件
* `git clean -df`  untracked状态，撤销新增的文件和文件夹
* `git reset` 暂存区的修改恢复到工作区 (已暂存，但是未提交)
* `git reset --soft`  回到未修改状态，修改的内容仍然在工作区中 (已暂存，但是未提交)
* `git reset --hard`  回到未修改状态，清空暂存区和工作区 (已暂存，但是未提交)
* `git reset --soft HEAD^` 取消上一次提交，保留提交后的修改 (已提交，未推送)
* `git reset --hard origin/master` 回退与本地远程仓库一致  (已提交，未推送)
* `git reset --hard HEAD^` 回退到本地仓库上一个版本  (已提交，未推送)



### Rebase
> 变基操作。orgin和branch1两个分支各自有多次提交，如果想把branch1上的内容提交到origin上，除了merge操作，我们还可以使用`git rebase`。它会将branch1上的commit操作暂时取消，暂存至`.git/rebase`目录下，并且将origin分支的内容更新到branch1。如果产生了冲突，则解决冲突后执行`git rebase --continue`。最后我们只需要把branch1分支上的内容，merge到origin上即可。
rebase操作可以让分支树更加线性。

* `git rebase -i  [startpoint]  [endpoint]`选出编辑区间，让操作者编辑完成合并操作   
* `git rebase <branch_name>` 将<branch_name>上的commit变基到当前分支。过程中可能会产生冲突。
* `git rebase --continue` rebase修复冲突后，继续执行变基操作。
