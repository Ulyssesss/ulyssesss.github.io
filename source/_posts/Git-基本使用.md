---
title: Git 基本使用
date: 2016-10-30 14:22:14
tags:
- Git
categories:
- 技术
---
* 配置用户名和email地址  

  `$ git config --global user.name "Your Name"`  

  `$ git config --global user.email "email@example.com"`


* 初始化仓库  

   `$ git init`


<!--more-->



* 添加到暂存区，提交  

  `$ git add <filename>`  

  `$ git commit -m 'xxx, xxx, xxx'`


* 查看当前状态  

  `$ git status`


* 查看修改   

  `$ git diff <filename>`


* 查看提交日志  

  `$ git log` (--pretty=oneline 显示单行信息)


* 查看命令日志  

  `$ git reflog`


* 回退到某一版本   

  `$ git reset --hard xxx`
  `$ git reset --hard HEAD^` (上一版本)


* 丢弃工作区的修改  

  `$ git checkout -- <filename>`


* 撤销暂存区的修改   

  `$ git reset HEAD <filename>`


* 从版本库中删除   

  `$ git rm <filename>`


* 创建ssh key  

  `$ ssh-keygen -t rsa -C "youremail@example.com"`


* 关联远程仓库   

  `$ git remote add origin git@github.com:GitUsername/RepositoryName.git`


* 取消关联远程仓库   

  `$ git remote remove origin`


* 推送到远程仓库   

  `$ git push -u origin master` (-u将本地master和远程master关联)


* 克隆远程仓库  

  `$ git clone git@github.com:<GitUsername>/<RepositoryName>.git`


* 创建分支并切换  

  `$ git checkout -b <name>` (相当于 `$ git branch <name>` 和 `$ git checkout <name>` 两条指令)


* 切换到 \<name\> 分支  

  `$ git checkout <name>`


* 合并 \<name\> 分支到当前分支   

  `$ git merge <name>`

  

* 删除 \<name\> 分支   

  `$ git branch -d <name>`


* 查看分支   

  `$ git branch`


* 禁用fast forward模式合并分支   

  `$ git merge --no-ff -m 'merge with no-ff' <name>`


* 储存当前现场   

  `$ git stash`


* 查看工作现场   

  `$ git stash list`


* 恢复工作现场   

  `$ git stash apply` (不删除stash)


* 删除工作现场  

  `$ git stash drop`


* 恢复工作现场并删除   

  `$ git stash pop`


* 强制删除分支   

  `$ git branch -D <name>`


* 查看远程库信息   

  `$ git remote ` (-v更详细的信息)


* 创建标签  

  `$ git tag <name>` 


* 查看所有标签  

  `$ git tag`


* 查看标签信息   

  `$ git show <tagname>`


* 删除本地标签   

  `$ git tag -d <tagname>`


* 推送本地标签   

  `$ git push origin <tagname> `(推送全部未推送的本地标签 git push origin --tags)


* 删除一个远程标签  

  `$ git push origin :refs/tags/<tagname>`


* 强制添加忽略的文件   

  `$ git add -f <filename>`
