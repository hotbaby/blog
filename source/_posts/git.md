---
title: Git
date: 2017-04-19 22:06:23
tags: [Git]
toc: true
---

Git is a free and open source distribute version control system designed to handle evertything from small to very large project wit speed and efficiency.

## custom domain redirect

添加DNS解析记录：

| 记录类型 | 主机记录 | 记录值            |
| -------- | -------- | ----------------- |
| CNAME    | blog     | hotbaby.github.io |

修改GitHub pages CNAME记录：

创建CNAME文件，并写入域名。 比如`echo 'blog.mengyangyang.org' >> CNAME`

hexo 更新覆盖CNAME:

在`source`目录下创建CNAME文件，并写入域名

## submodule

以引用PCI项目为例，介绍如何添加子模块

```shell
$ git clone git@github.com:hotbaby/PCI.git
$ git submodule add https://github.com/uolter/PCI.git code/origin
$ git commit -m 'Add PCI submodule.'
$ git push
```

## tag

添加tag

```shell
$ git tag tag_name
$ git push origin tag_name
$ git push origin --tags
```

某次提交打tag

```shell
$ git tag -a tag_name commit_id
$ git push origin tag_name
$ git push origin --tags # 全部tags
```

查看tag

`git tag --list`

切换到tag

`git checkout tag_name`

删除tag

```shell
$ git tag -d tag_name #删除本地tag
$ git push  origin --delete tag <tag_name> #删除服务器tag
```

## config

编辑配置文件：

`vim ~/.gitconfig`

配置编辑器：

`git config --global core.editor vim`

## reference

- [Git子模块](https://git-scm.com/book/zh/v1/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97)
- [Git子模块引用外部项目](http://wonux.tech/git-git-submodule.html)

 