---
title: Centos 6.5 安装 tmux 最新版
date: 2018-06-06 20:32:15
tags: linux
---

## Contex
- linux Centos6.5,64位
- tmux 2.7

<!--more-->

## 流程

### 下载安装libevent

先安装libevent，在后面编译好tmux后会链接到这里来
```
wget https://github.com/downloads/libevent/libevent/libevent-2.0.21-stable.tar.gz

tar zxvf libevent-2.0.21-stable.tar.gz

cd libevent-2.0.21-stable

./configure

make && make install
```

### 安装tmux

```
wget https://github.com/tmux/tmux/releases/download/2.7/tmux-2.7.tar.gz

tar zxvf tmux-2.7.tar.gz

cd tmux-2.7

./configure

make && make install
```

在执`./configure`的时候可能遇到的问题
- configure: error: no acceptable C compiler found in $PATH
解决的办法是:
```
 yum install gcc
```

- configure: error: "curses not found"
这个问题解决办法是：
```
yum install ncurses-devel
```


然后`make && make install` 完成之后输入`tmux`检验是否安装成功，这时候可能会出现异常
```
tmux: error while loading shared libraries: libevent-2.0.so.5: cannot open shared object file: No such file or directory
```

解决办法是：
```
ln -s /usr/local/lib/libevent-2.0.so.5 /usr/lib64/libevent-2.0.so.5`
```


在输入`tmux`应该就成功了。







