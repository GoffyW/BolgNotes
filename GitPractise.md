# GitPractise

### 基础操作

##### 0、本地新建文件夹，创建Git工作空间

##### 1、在此文件夹右键Git Bash Here，输入命令

```
git init
```

此步操作为初始化git工作区间

##### 2、将需要上传的项目复制到工作区间

##### 3、输入命令git status 查看git工作区间的文件变化

```
git status
```

##### 4、输入命令git add . 将文件加入到暂存区

```
git add .
```

##### 5、将文件提交到本地仓库

```
git commit -m "the first commit"
```

##### 6、在Github种新建仓库（省略）

##### 7、复制仓库链接（HTTPS链接或SSH）

##### 8、将本地仓库关联到Github仓库

```
git remote add origin https://...
```

##### 9、将本地仓库同步到GitHub仓库

```
git push origin master
```

##### 10、git拉取远程分支与本地代码合并：

```
git pull origin master	#不推荐，因为这样直接合并，无法提前处理冲突
```

 推荐下面这种方式合并远程仓库代码到本地

```
git fetch origin master #获取远端的origin/master分支
git log -p master..origin/master #查看本地master与远端origin/master的版本差异
git merge origin/master #合并远端分支origin/master到当前分支
```

### Git分支操作

##### git创建分支

```
git branch branch1 #这样创建完分支之后并不会自动切换到该分支
git checkout -b branch2 #这样创建分支之后会自动切换到该分支
```

**git查看本地分支**

```
git branch
```

##### git查看远程分支

```
git branch -r
```

##### git查看所属分支

```
git branch -a
```

##### git切换分支

```
git checkout branch1
```

##### 测试直接Push则会遇到错误

```
$ git push
fatal: The current branch branch1 has no upstream branch.
To push the current branch and set the remote as upstream, use

    git push --set-upstream origin branch1
```

##### 则需先将远端的分支拉下来

```
git push --set-upstream origin branch1
```

##### 然后切换到主分支上

```
git checkout master
```

##### 然后合并分支冲突

如果想删除Github上的某个文件夹，使用下面命令

```
git rm -r --cached out  #--cached不会把本地的out删除
git commit -m 'delete out dir'
git push -u origin master
```

##### git删除远程仓库文件夹(useless为文件夹名字)：

```
git rm --cached -r useless
git commit -m "remove directory from remote repository"
git push
```

##### 合并分支代码

```
git  checkout master # 切换到需要合并的分支上面（切换到Maste）
git pull origin master #多人协作下最好进行Pull
git  merge dev #dev的代码合并到当前分支
git status
git push origin master
```