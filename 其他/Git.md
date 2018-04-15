
## log 相关 :
```
git  log -n 5   

git log --author=""

git log --grep="54733094l"

git log --oneline 

git log --stat 
显示改了哪些文件
git log -p
显示改的那些代码
```

## checkout:
```
git checkout -- a.py 缓存区 -> 工作区 

git checkout a1e8fb5 
    会切换到指定版本，如果不是Head，会自动新拉一个分支
    同时会丢失缓存区

git checkout a1e8fb5 a.py
    不会切分支
    缓存区和工作区没commit的都丢失
    指定文件切到 之前分支，并add到缓存区

git checkout head a.py
    丢弃没commit的所有内容
```

## revert
```
git revert <commit>
这个可以回退这个版本，如果这个文件之后又被改过，就会和改动进行merge
```


## reset
```
git reset <commit>
不会改变工作目录，缓存区和项目历史还原到 commit

git reset --hard <commit>
重设工作目录，缓存区，和项目历史

git reset --soft <commit>
不会改工作目录，把历史还原，缓存自动提交撤销的保存

重点是，确保你只对本地的修改使用git reset——而不是公共更改。如果你需要修复一个公共提交，git revert命令正是被设计来做这个的。
```


## clean
```
新的文件
未被跟踪的文件，reset --hard的时候不会生效。这个两个命令相结合，你就可以将工作目录回到之前特定提交时的状态。

git clean -f <path>
某个目录删除文件

git clean -n 
git告诉你哪些会删除，而不是真的删除

git clean -df
某个目录删除文件 和文件夹

git clean -xf
移除当前目录下未跟踪的文件，以及Git一般忽略的文件。
```

## commit amend
```
git commit --amend --no-edit
把上次的和缓存区里的一起提交,不改提交信息
```

## rebase
```
merge master之前

先git rebase -i <base>

冲突的时候 git add -u，就不会提交其他的文件
```

## pull & fetch
``` 
fetch 是不会自动merge，而新生成一个分支

pull 自动merge

git pull --rebase <remote> 
这是使用git rebase合并远程分支和本地分支，而不是使用git merge。

git config --global branch.autosetuprebase always
所有的pull都是rebase了
```
