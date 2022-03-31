https://blog.csdn.net/Hello_World_QWP/article/details/80885480

## 1.从分支clone

1）`git clone -b 分支名 地址 `

2）tortoiseGit clone 指定Branch

https://blog.csdn.net/Hello_World_QWP/article/details/80885480

## 2.合并

1) 首先切换到master分支上

```shell
git  checkout master
```

2. 如果是多人开发的话 需要把远程master上的代码pull下来

```shell
git pull origin master
```

3. 然后我们把dev分支的代码合并到master上

```shell
git  merge dev
```

4. 然后查看状态及执行提交命令

```shell
git status
git push origin master    
```



## tortoiseGit

切换到master  merege 选择分支，合并



## 3.修改密码

```shell
git config user.name
git config --global user.name 
git config --global user.password "upaXM@2022""zhenghs"
第一次
后续修改密码 要该windows下 用户凭证

```

