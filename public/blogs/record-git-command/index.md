### 本地同步远程分支代码
```
git pull
git pull --rebase #有冲突时使用
```

### 丢弃本地变更，强制同步远程
```
git reset --hard  origin/分支名
```

### 查看本地变更的文件列表
```
git status
```

### git本地回滚到指定的commit
```
git log --oneline
git reset --hard commit-id(要回滚的commit)
```