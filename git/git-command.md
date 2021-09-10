# Git Command 
常用的git命令集合

## Reference
* 常用 Git 命令清单[https://www.ruanyifeng.com/blog/2015/12/git-cheat-sheet.html](https://www.ruanyifeng.com/blog/2015/12/git-cheat-sheet.html)

## Config
### 配置
* `git config -l` 查看git配置

## Commands
### 提交
* `git diff` 查看被修改的文件
* `git add <file_path>` 把文件提交至预备提交区域
* `git commmit -m "commit"`提交文件，增加描述
* `git commit -am "commit_msg"` 自动将被修改、删除的文件（不包括未加入索引的文件）加入暂存区，并提交
* `git reset HEAD <file_path>`取消文件的暂存状态（预备提交区）

### 回退/恢复
* `git reset --soft HEAD^` 取消上一次提交，保留提交后的修改


