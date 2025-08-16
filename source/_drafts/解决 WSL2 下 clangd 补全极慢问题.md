---
abbrlink: wsl2-clangd-slow
categories: []
date: '2025-08-16T14:46:51.498816+08:00'
tags: []
title: 解决 WSL2 下 clangd 补全极慢问题
updated: '2025-08-16T14:46:53.123+08:00'
---
近期用 WSL2 下的 VSCode 写 C++ 项目时，clangd 的补全一直特别慢，到了要等待两三秒才慢悠悠地出结果的地步，Ctrl+左键跳转也一直转圈圈，写得我特别烦躁。

状态栏一直显示这个，卡好久，日志中也显示到需要五六百毫秒。

![](https://pics.r1kka.one/file/1755326924353_%E5%9B%BE%E7%89%87.png)



尝试修改了各种配置、各种重装，都无济于事。直到我今天实在受不了，尝试搜索，发现[一篇百度贴吧帖子]([https://](https://tieba.baidu.com/p/8560268468#))，完美解决了问题，现在备份在这里，便于后续查找。

---



> wsl2里默认把windows的环境变量加入了Linux环境变量里面，导致搜索都会把windows目录加进去，而这是跨文件系统的，读取会非常慢，比如shell里的智能提示补全也会一卡一卡的，可以配置选项不把windows环境变量加入。



先在sudo模式下把 `/etc/wsl.conf` 文件权限+w，然后在该文件下新建节

```ini
[interop]
enabled = false
appendWindowsPath = false
```

最后重启wsl就好了。
