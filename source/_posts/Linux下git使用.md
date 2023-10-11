---
title: Linux下git使用
date: 2023-10-09 16:46:47
tags: 
     - git
     - linux
categories: 教程

---
简单介绍git的基本命令
<!-- more -->
# Linux下Git使用

### 1. git的安装

```shell
sudo apt install git
```

安装完，使用`git --version`查看git版本

### 2. 配置git

```shell
git config --global user.name "Your Name“	##配置用户 
git config --global user.email email@example.com	##配置邮箱
git config --global --list			##查看配置信息
## --global 全局配置，所有仓库生效，不加就只对当前用户有效
## --system 系统配置，对所有用户生效
```

### 3. 新建版本库

```shell
 git init
```

### 4. 工作区域与文件状态

![image-20230920204530239](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230920204530239.png)

![image-20230920204638439](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230920204638439.png)

### 5. 添加和提交文件

```shell
git init		##创建仓库
git status		##查看仓库的状态
git add			##添加到暂存库
git commit 		##提交
git rm --cached <file>...		##将文件从暂存区中去除
git log			##查看提交记录
git ls-files	##查看暂存区的文件
git commit -a -m " " #实现添加和提交两个步骤
```

### 3. 回退版本

```shell
git reset --soft
git reset --hard
git reset --mixed	
```

![image-20230921085607903](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230921085607903.png)

```shell
git reset HEAD^		##默认为mixed,回退一个版本
```

```txt
HEAD 表示当前版本
HEAD^ 上一个版本
HEAD^^ 上上一个版本
HEAD^^^ 上上上一个版本
HEAD~0 表示当前版本
HEAD~1 上一个版本
HEAD^2 上上一个版本
HEAD^3 上上上一个版本
执行 git reset HEAD 以取消之前 git add 添加，但不希望包含在下一提交快照中的缓存
```

误操作之后

```shell
git reflog		##回溯日志
git reset --hard 版本号	##回退
```

### 6. 查看差异

![image-20230921091532989](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230921091532989.png)

```shell
git diff				#默认比较工作区和暂存区之间的差异
git diff HEAD			#比较工作区和版本库之间的差异
git diff --cached		#比较暂存区和版本库之间的区别
git diff 版本号	版本号	#比较两个版本之间的差异
git diff HEAD~ HEAD		#如回退版本
```

![image-20230921092511736](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230921092511736.png)

### 7. 删除文件

方法1：先删除本地文件，再提交

```shell
rm -rf 3.txt	##删除本地中的文件
git add .		##删除暂存区中的文件
git commit -m 'deleted 3.txt'	##删除工作区文件
```

方法2

```shell
git rm 2.txt	##删除本地和暂存区中文件
git commit -m 'deleted 2.txt'	##删除工作区文件
```

![image-20230921093523261](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230921093523261.png)

### 8. 忽略文件

![image-20230921093644360](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230921093644360.png)

```shell
 echo "*.log" > .gitignore		##表示忽略所有日志文件
```

![image-20230921094538072](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230921094538072.png)

### 9. 远程仓库github

注册github账号，创建仓库

### 10. ssh配置和克隆仓库

创建ssh密钥

```shell
cd ~
cd .ssh		#如果显示文件不存在，就之间执行以下命令
ssh-keygen -t rsa -b 4096 -C "xxx@email.com"	#直接enter,如果是第二次执行，记得更改文件名，不然会覆盖之前的id_rsa文件，且不可逆
```

![image-20230921101312117](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230921101312117.png)

执行之前的命令会生成以下两个文件，有.pub的是公钥文件，没有的是私钥文件，复制公钥文件到github的Settings里的ssh配置

如果是第一次配置就配置完了，如果是第二次，更改了文件名的，就需要新建一个config文件，内容为

![image-20230921101639858](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230921101639858.png)

意思是：当我们在访问github.com这个网站的时候。使用的是test这个文件里的密钥

```shell
 git clone git@github.com:xxx.git		##克隆新建的远程仓库
```

![image-20230921102254407](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230921102254407.png)

```shell
git push	##将本地文件推送到远程仓库
```

### 11. 关联本地仓库和远程仓库

1. 本地无仓库

```shell
echo "# fist-repo" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:xxx.git
git push -u origin main
```

2. 本地已经有仓库

```shell
git remote add origin git@github.com:xxx.git		##添加一个远程仓库
git branch -M main									##指定分支的名称为main
git push -u origin main								##把本地的main分支与远程的orgin main分支关联
```



```shell
git remote -v		##查看本地仓库对应的远程仓库别名
```

![image-20230921103436235](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230921103436235.png)

![image-20230921103522078](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230921103522078.png)

### 12. 分支

```shell
git branch			#查看分支
git branch	dev		#新建分支dev
git checkout dev	#切换到分支dev(有风险，有时会用来恢复文件)
git switch main		#切换到分支main
git merge dev		#将要被合并的分支(dev)合并到当前分支(main)
git log --graph --oneline --decorate --all	##查看分支情况
git branch -d dev	#删除分支(已经合并)
git branch -D dev	#删除分支(未合并，强行删除)
```

![image-20230921140159925](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230921140159925.png)

### 13. 回退和rebase

![image-20230921140826350](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230921140826350.png)

![image-20230921141221770](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230921141221770.png)

![image-20230921141308973](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230921141308973.png)

### 14. 分支管理和工作流模型

1. **git flow模型**

![img](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/v2-04b62035a49e04e997d3cb05e22970a9_r.jpg)

![img](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/v2-f20d5d1eab3704639fe64531776b4bac_720w.webp)

![img](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/v2-c7b37d13cfb22d77c840bac3cceee6a7_r.jpg)

2. **github flow模型**

![image-20230921141728150](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230921141728150.png)

![image-20230921141750346](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230921141750346.png)

