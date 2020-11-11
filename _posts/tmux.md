### 常用命令
[参考](https://www.cnblogs.com/kevingrace/p/6496899.html)  
[参考](http://mingxinglai.com/cn/2012/09/tmux/)


* 会话
```bash
tmux new -s <name>  # 新建会话
tmux ls             # 查看创建得所有会话
tmux a -t <name>    # 登录一个已知会话
ctrl+b + s          # 以菜单方式显示和选择会话
ctrl-b + d          # 退出会话，但不会关闭会话
tmux attach         # 进入tmux
ctrl + d            # 关闭会话
tmux kill-session -t <name>  # 关闭会话
tmux rename -t <name> <newname>  # 重命名
```

* 窗口
```
ctrl+b c           创建新窗口
ctrl+b n           选择下一个窗口
ctrl+b p           选择前一个窗口
ctrl+b &           确认后退出当前窗口
ctrl+b w           以菜单方式显示及选择窗口
```

* 窗格
```bash
ctrl+b  "           纵向分隔窗口
ctrl+b %            横向分隔窗口
ctrl+b 空格键       采用下一个内置布局
ctrl+b 方向键      上一个及下一个分隔窗格
ctrl+b ctrl-方向键    调整分隔窗格大小
ctrl+b x           关闭窗格
ctrl+b !            把当前窗格变为新窗口
ctrl+b [            滚屏
```

```
ctrl+b ?            显示快捷键帮助
```