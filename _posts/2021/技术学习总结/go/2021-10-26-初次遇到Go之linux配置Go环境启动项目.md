---
layout: post
title: linux配置Go环境启动项目
tags: Go
categories: Go
# 配置linux go运行环境
## 下载安装go安装包
- 下载go安装包

https://studygolang.com/dl

- 解压安装包

tar -zxvf go1.17.2.linux-amd64.tar.gz

## 配置go环境变量
```doc
//命令行仅输入“cd”到主目录下
$ cd

//ls -a可以查看主目录下的所有文件，目标文件为.bashrc
//vi .bashrc进行增添环境变量
$ vi .bashrc

//输入以下代码
export GOROOT=/data/go/env/go
export GOPATH=/data/go/project
export PATH=$PATH:/data/go/env/go/bin
export PATH=$PATH:$GOPATH/bin

//按“Esc”,接着“:wq”保存退出

//让.bashrc生效
source .bashrc
```
## bashrc与profile的区别
```doc
/etc/profile，/etc/bashrc 是系统全局环境变量设定
~/.profile，~/.bashrc用户家目录下的私有环境变量设定
当登入系统时候获得一个shell进程时，其读取环境设定档有三步
1首先读入的是全局环境变量设定档/etc/profile，然后根据其内容读取额外的设定的文档，如
/etc/profile.d和/etc/inputrc
2然后根据不同使用者帐号，去其家目录读取~/.bash_profile，如果这读取不了就读取~/.bash_login，这个也读取不了才会读取
~/.profile，这三个文档设定基本上是一样的，读取有优先关系
3然后在根据用户帐号读取~/.bashrc
至于~/.profile与~/.bashrc的不区别
都具有个性化定制功能
~/.profile可以设定本用户专有的路径，环境变量，等，它只能登入的时候执行一次
~/.bashrc也是某用户专有设定文档，可以设定路径，命令别名，每次shell script的执行都会使用它一次
```

# linux运行go应用

## 安装git拉取项目

    yum -y install git

## clone项目
    git clone https://gitee.com/simonfishing/click-max.git

## 部署项目
    go install clickMax.go
    
## 运行项目 
    nohup ./click-max &

## 遇到的问题：
### 启动项目后无法连接远程rds ，自动连接本地数据库
**原因:** 启动项目时需要加载config配置文件,会在安装包目录下寻找config文件。

**解决方案:** 需要将项目中config文件copy到安装包目录。(前端页面也需要copy到安装包目录下)

### 启动项目报错
问题描述：

      go: github.com/ansel1/merry@v1.5.1: missing go.sum entry; to add it:
	go mod download github.com/ansel1/merry
解决方案：

    通过 go mod tidy 添加需要用到但go.mod中查不到的模块,删除未使用的模块
### go mod tidy 无法拉下包
    go env -w GOPROXY=https://goproxy.cn 

