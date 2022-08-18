---
title: 在Github上备份Hexo博客
tags: hexo
categories: 博客
abbrlink: 5d759798
date: 2021-07-27 23:13:25
---

# 原理
我们的博客是托管到 GitHub 上的. 而我们每次上传（hexo d）的是网页文件, 不是我们
的文章, 所以我们如果想上传文章, 但同时不会干扰到网页部署, 就在 GitHub 的博客仓
库上建立一个分支 hexo, 这个 hexo 分支的作用就是用来保存我的 MarkDown 文章, 站点
配置和一些其他文件.

这样hexo d推送的是 master 分支, 而git push推送的是 hexo 分支, 互不干扰.

# 建立一个中转站
建立一个文件夹, 名字随便, 我这里叫 hexo
```(C++)
git clone https://github.com/yang-shuohao/yang-shuohao.github.io.git
```
其实这里仅仅只是为了获得版本管理的.git隐藏文件夹.

# 建立分支
建立一个分支, 我这里分支名为 hexo
```(C++)
git checkout -b hexo
```

# 清空 hexo 分支
克隆下来的都是一些编译后的静态网页, 我们不需要. 删除除了.git文件夹的所有文件.
```(C++)
git add --all
git commit -m  "清空hexo分支仓库"
git push --set-upstream origin hexo
```
这里同时设置了以后默认为hexo分支, 回到博客的根目录下就能看到.

我们的博客的站点配置文件_config.yml的默认提交分支为 master.
```(C++)
deploy:
  type: git
  repo: https://github.com/yang-shuohao/yang-shuohao.github.io.git
  branch: master #提交的默认分支
```
# 移动文件
把.git文件夹移动到博客的根目录下.

# 提交源文件
到了这一步有个注意点, 如果你的主题文件是克隆 Github 下来的, 那么会带有该主题的
版本管理文件, 也就是.git文件夹. 所以主题下面的要删除.git文件夹和
.gitignore文件, 否则会忽略这个 butterfly 主题的上传.

.deploy_git是部署静态文章用的, 需要保留.

之后每次更新博客之后, push 源文件到hexo分支即可备份

# 个人备份习惯
个人而言习惯于先备份文件再生成博客。即先执行git add .，git commit -m "Backup"，git push origin hexo将博客备份完成，然后执行hexo g -d发布博客。

# 恢复博客
目前假设本地Hexo博客基础环境已经搭好（其实很简单，只需要安装git和Nodejs（我下载的版本为LTS node-v14.17.3-x64.msi)，然后通过`npm install -g hexo-cli`安装hexo就可以了）。

## 克隆项目到本地
输入下列命令克隆博客必须文件(hexo分支)：
```
git clone https://github.com/yourgithubname/yourgithubname.github.io
```
## 恢复博客
在克隆的那个文件夹下输入如下命令恢复博客：
```
npm install hexo-cli
npm install
npm install hexo-deployer-git
```
在此不需要执行hexo init这条指令，因为不是从零搭建起新博客。

# 迁移后可能遇到的问题
## node版本过高
重新安装低版本 node

## .deploy_git异常
```
rm -rf .deploy_git
hexo g
hexo d
```