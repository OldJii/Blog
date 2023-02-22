---
title: "Mac多个Git账户共存"
date: 2023-02-14T10:46:28+08:00
draft: false
tags: ["Git"]
categories: ["工作"]
---

## 清除配置
查看 Git 本地配置

```shell
git config --list
```

清除用户名和邮箱

```shell
git config --global --unset user.name
git config --global --unset user.email
```

<!--more-->

## 生成 ssh-key
使用 `ssh-keygen` 命令生成 ssh-key，并手动指定 id
```shell
ssh-keygen -t rsa -f ~/.ssh/id_rsa_xx1@gmail.com -C "xx1@gmail.com"
```
生成成功会输出以下内容
```shell
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/james/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /Users/james/.ssh/id_rsa.
Your public key has been saved in /Users/james/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:rwtxjGTJPoV9Mg8lFSf8D4X6jFexWVXKOMRaVyo+RO8 xx1@gmail.com
The key's randomart image is:
+---[RSA 3072]----+
|        .o=o+. .*|
|     . + o.=+++o.|
|      * * .==o== |
|     + + *ooo++  |
|      = S .+o+E  |
|       + .. +..  |
|      .   ..     |
|       . .       |
|        o.       |
+----[SHA256]-----+
```
然后继续生成你的其他 Git 账号对应的 ssh-key
```shell
ssh-keygen -t rsa -f ~/.ssh/id_rsa_xx2@gmail.com -C "xx2@gmail.com"
```

## 信任 ssh-key
使用 `ssh-add` 命令将生成的 ssh-key 添加到 ssh-agent 信任列表
```shell
ssh-add ~/.ssh/id_rsa_xx1@gmail.com
ssh-add ~/.ssh/id_rsa_xx2@gmail.com
```
如果遇到 `Could not open a connection to your authentication agent.` ，输入
```shell
ssh-agent bash
```
然后重复执行 `ssh-add` 即可解决

## 配置公钥
复制对应公钥，配置到对应 Git 网站中（GitHub / GitLab）
```shell
pbcopy < ~/.ssh/id_rsa_xx1@gmail.com.pub
pbcopy < ~/.ssh/id_rsa_xx2@gmail.com.pub
```
这步具体的操作如果有不清楚的可以参考 [Adding a new SSH key to your GitHub account](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)

## 配置 config
进入 ssh 目录
```shell
open ~/.ssh/
```
按照以下规则编辑 config 文件，没有则创建

|键|值|说明|
|-|-|-|
|Host|主机|自己起|
|Hostname|主机名|Git 公有地址，比如 gitub.com / gittee.com|
|IdentityFile|身份文件|rsa 文件路径|
|User|用户|自己起，一般邮箱就好|

编辑完 config 内容如下
```shell
Host github1.com
Hostname github.com
IdentityFile ~/.ssh/id_rsa_xx1@gmail.com
User xx1@gmail.com

Host github2.com
Hostname github.com
IdentityFile ~/.ssh/id_rsa_xx2@gmail.com
User xx2@gmail.com
```

## 测试链接
测试 Git 账号是否连接成功，`git@` 之后是 config 文件中配置的 Host
```shell
ssh -T git@github1.com
ssh -T git@github2.com
```
连接成功会有以下输出
```shell
Hi xxx! You've successfully authenticated, but GitHub does not provide shell access.
```