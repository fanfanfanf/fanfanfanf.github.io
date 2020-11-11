
```bash
git init  # 创建仓库
git add readme.txt  # 添加文件  ||  更新文件
git commit -m "wrote a readme file"   # 提交更新
git status    # 查看状态，是否有修改
git diff readme.txt    # show diff

git add . # 他会监控工作区的状态树，使用它会把工作时的所有变化提交到暂存区，包括文件内容修改(modified)以及新文件(new)，但不包括被删除的文件。
git add -u # 他仅监控已经被add的文件（即tracked file），他会将被修改的文件提交到暂存区。add -u 不会提交新文件（untracked file）。（git add --update的缩写）
git add -A # 是上面两个功能的合集（git add --all的缩写）

git log
git log --pretty=oneline   # log
git log --author=bob
git log --graph --oneline --decorate --all  # 展示所有的分支

git reset --hard HEAD^     # 版本回退，用HEAD表示当前版本，上一个版本就是HEAD^，上上一个版本就是HEAD^^，当然往上100个版本写成HEAD~100。
git reset --hard 3628164   # 按版本号回退
git reflog                 # 历史纪录
git checkout -- readme.txt # 撤销未提交的修改
git reset HEAD readme.txt  # 同上

git push origin master     # （commit后）将某个分支提交到远端仓库
git pull                   # 更新本地仓库至最新改动
git pull <远程主机名> <远程分支名>:<本地分支名>
git merge <branch>         # 合并其他分支到当前分支

git fetch origin master    # 取回origin主机的master分支
git branch -r              # 查看本地保存的远程分支
git branch -a              # 查看本地保存的所有分支
>> origin/master           # 所取回的更新，在本地主机上要用”远程主机名/分支名”的形式读取
git checkout -b newBrach origin/master  # 在远程分支的基础上创建一个新的分支

git checkout -b feature_x  # 创建一个叫做“feature_x”的分支，并切换过去
git checkout master        # 切换回主分支
git branch -d feature_x    # 把新建的分支删掉

git tag 1.0.0 1b2e1d63ff   # 创建一个叫做 1.0.0 的标签

git clone 下载源码
git tag　列出所有版本号
git checkout　+某版本号　

# 可以认为git pull是git fetch和git merge两个步骤的结合
git fetch origin master:tmp  # 在本地新建一个temp分支，并将远程origin仓库的master分支代码下载到本地temp分支
git diff tmp               # 来比较本地代码与刚刚从远程下载下来的代码的区别
git merge tmp              # 合并temp分支到本地的master分支
git branch -d temp         # 如果不想保留temp分支 可以用这步删除

git clone <repository> --recursive     # 递归的方式克隆整个项目
git submodule add <repository> <path>  # 添加子模块
git submodule init                     # 初始化子模块
git submodule update                   # 更新子模块
git submodule foreach git pull         #　拉取所有子模块
 
git submodule update --init --recursive　　# 下载更新子模块


git stash           # 保留未commit的更改
git stash pop       # 恢复stash中的内容
git stash list      # 查看保存的stash内容
git stash apply --index [stashid] #恢复但不删除
git stash drop [stashid] #删除

git cherry-pick commitid    # 想某分支的某个commit复制到当前分支
git cherry-pick commitid1..commitid100  #复制多个
```
[其他](http://rogerdudler.github.io/git-guide/index.zh.html)